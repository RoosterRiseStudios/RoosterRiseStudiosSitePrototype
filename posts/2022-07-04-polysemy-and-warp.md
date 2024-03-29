---
title: Warp as an effect in Polysemy
author: Thomas Mahler
tags: clean-architecture, haskell, polysemy, algebraic-effects, domain-driven-design, ports-and-adapters, hexagonal-architecture, onion-architecture, servant, warp, io-monad, testability, architecture, algebraic, polysemy-library, polysemy-effects
---


[![Actions Status](https://github.com/thma/PolysemyCleanArchitecture/workflows/Haskell%20CI/badge.svg)](https://github.com/thma/PolysemyCleanArchitecture/actions)
<a href="https://github.com/thma/PolysemyCleanArchitecture"><img src="https://thma.github.io/img/forkme.png" height="20" ></a>

## Introduction

Two years ago I wrote [Implementing Clean Architecture with Haskell and Polysemy](https://thma.github.io/posts/2020-05-29-polysemy-clean-architecture.html) where I demonstrated how core elements of Clean Architecture can be implemented with Polysemy in quite an elegant way.

A few months ago an interesting question was raised in the [issues section of the respective GitHub project](https://github.com/thma/PolysemyCleanArchitecture/issues/2):

> I'm wondering, if we want to change the server to serverless, is it easy to do that? 
> If we treat the server as an effect, can we just change the [polysemy effect] interpreter?

This question points to one of the grey spots of my original design: The whole polysemy effect stack is executed as part of the `createApp` function which provides a `Network.Wai.Application` which is then executed by `Warp.run`.

Wouldn't it be much more elegant to also define the server execution as an effect? This would make the overall design more homogenous by giving the final control to the Polysemy effect interpreter. 

In this post I'm going to present a design that provides a solution for this.

## The 'old' way

In order to better understand the issue lets have a brieve look at the original design.

```haskell
main :: IO ()
main = do
  config <- loadConfig           -- load configuration (e.g. from file)
  let p = port config            -- obtain http port from config
  Warp.run p (createApp config)  -- create Application based on config and run it
```

As we can see from the `main` function our whole app is just an ordinary `Network.Wai.Application` which is executed by `Warp.run`.

So the final control of this application is managed by warp. We don't see anything of Polysemy here. Only if we dig deeper we'll find that the Polysemy effect stack is interpreted under the hood of the `Application`:

```haskell
createApp :: Config -> Application
createApp config = serve reservationAPI (liftServer config)


liftServer :: Config -> ServerT ReservationAPI Handler
liftServer config = hoistServer reservationAPI (interpretServer config) reservationServer
  where
    interpretServer :: (Show k, Read k, ToJSON v, FromJSON v)
                    => Config -> Sem '[KVS k v, Input Config, Trace, Error ReservationError, Embed IO] a -> Handler a
    interpretServer conf sem  =  sem
          & runSelectedKvsBackend conf
          & runInputConst conf
          & runSelectedTrace conf
          & runError @ReservationError
          & runM
          & liftToHandler

    liftToHandler :: IO (Either ReservationError a) -> Handler a
    liftToHandler = Handler . ExceptT . fmap handleErrors

    handleErrors :: Either ReservationError b -> Either ServerError b
    handleErrors (Left (ReservationNotPossible msg)) = Left err412 { errBody = pack msg}
    handleErrors (Right value) = Right value
```

This design works perfectly fine and also follows the common practise to exploit the Servant [`hoistServer`](https://hackage.haskell.org/package/servant-server-0.19.1/docs/Servant-Server.html#v:hoistServer) mechanism.

The only complaint is that the Polysemy effect interpreter is not executed as the outermost piece of code. Accordingly we have to *deploy our application into warp* rather than having Polysemy in control and *executing warp as an effect*.


## The new solution

Ok let's take the challenge and try to fix this 'bug'!

My overall approach goes like this:

1. Define a new `ExternalInterfaces.AppServer` effect. This will allow to abstractly define
   usage of an application server as a Polysemy effect.

2. Provide a Warp based implementation `ExternalInterfaces.WarpAppServer`.
   This will interprete the `AppServer` effect by running [Warp](http://www.aosabook.org/en/posa/warp.html).

3. write a main function that drives the whole Polysemy effect stack, including the Warp server. 
   
### The AppServer effect

First we define an abstract `AppServer` effect which captures the notion of executing a web application inside an http based server.
We'll support two constructors `ServeApp` which is more generic as it is modelled after the function signature of `Warp.run`.
The second constructor `ServeAppFromConfig` is matching the design of my original polysemy app where different elements of the application 
can be configured by a central `Config`:

```haskell
{-# LANGUAGE TemplateHaskell #-}

module ExternalInterfaces.AppServer where

import           InterfaceAdapters.Config
import           Network.Wai              (Application)
import           Polysemy                 (makeSem)

data AppServer m a where
  ServeApp           :: Application -> AppServer m () -- ^ serve a given application on a port
  ServeAppFromConfig :: Config -> AppServer m ()      -- ^ construct an application from a configuration and serve it

-- using TemplateHaskell to generate serveApp and serveAppFromConfig effect functions
makeSem ''AppServer
```

### The Warp based implementation of the effect

Implementing the effect is straightforward. `ServeApp` is directly implemented by `Warp.run`.
`ServeAppFromConfig` creates an `Application` instance based on the `Config` and then executes it with `Warp.run`:

```haskell
module ExternalInterfaces.WarpAppServer where

import           ExternalInterfaces.AppServer
import           ExternalInterfaces.ApplicationAssembly (createApp, loadConfig)
import           InterfaceAdapters.Config               (Config (..))
import qualified Network.Wai.Handler.Warp               as Warp (run)
import           Polysemy                               (Embed, Member, Sem,
                                                         embed, interpret, runM)

-- | Warp Based implementation of AppServer
runWarpAppServer :: (Member (Embed IO) r) => Int -> Sem (AppServer : r) a -> Sem r a
runWarpAppServer port = interpret $ \case
  -- this is the more generic version which maps directly to Warp.run
  ServeApp app -> embed $ Warp.run port app

  -- serving an application by constructing it from a config
  ServeAppFromConfig config ->
    embed $
      let app = createApp config
       in Warp.run port app
```

### Putting everything together in a new main function

The `main` function now contains the outer Polysemy effect interpreter:

```haskell
main :: IO ()
main = do
  config <- loadConfig      -- loading the config as before
  let p = port config       -- obtain http port from config
  serveAppFromConfig config -- declaring to run config based app on an AppServer
    & runWarpAppServer p    -- use Warp to interprete this effect
    & runM                  -- finally lower the `Sem` stack into `IO ()`
```

Interpretation of the *inner* effect stack is still handled by `createApp` but the actual control of the complete program now lies within the control of the outer effect interpreter in the `main` function. The `inner` effect stack is executed as part of the `runWarpAppServer` Polysemy interpreter function.

### Bonus: An AWS Lambda HAL based implementation of the AppServer effect

One of the many benefits of Polysemy is that it allows to provide multiple implementations of a given effect declaration.
I'll demonstrate this by presenting an implementation of the `AppServer` effect that allows to host our application on AWS Lambda (using [wai-handler-hal](https://github.com/bellroy/wai-handler-hal)).

```haskell
module ExternalInterfaces.HalAppServer where

import           ExternalInterfaces.AppServer
import           ExternalInterfaces.ApplicationAssembly (createApp)
import           InterfaceAdapters.Config               (Config)
import           AWS.Lambda.Runtime                     (mRuntime)
import qualified Network.Wai.Handler.Hal as Hal         (run)
import           Polysemy                               (Embed, Member, Sem,
                                                         embed, interpret, runM)

-- | wai-handler-hal Based implementation of AppServer
runHalAppServer :: (Member (Embed IO) r) => Sem (AppServer : r) a -> Sem r a
runHalAppServer = interpret $ \case
  ServeApp app -> embed $ mRuntime $ Hal.run app
  ServeAppFromConfig config ->
    embed $
      let app = createApp config
       in mRuntime $ Hal.run app
```

This looks very similar to the Warp implementation. The main difference is that we don't have to deal with an http port number as this will be managed by AWS Lambda.

Here is an alternative main function that shows how to interprete our application with `runHalAppServer`:

```haskell
main :: IO ()
main = do
  config <- loadConfig
  serveAppFromConfig config
    & runHalAppServer -- use HAL to run rest application
    & runM    
```


### Conclusion

I really like how this new approach cleans up the overall control flow and puts everything under control of the polysemy effect interpreter.
Following this way helps to make the separation between domain and use case logic from external interfaces and outward infrastructure even more distinct.

As an additional bonus we can now transparently execute our application on different hosting environments just by switching effect interpreter functions.
