---
title: Discord Game SDK for UE
description: Unreal Engine support for Discord Rich Presence, Game Invites and Overlay.
publishDate: 2025-03-09 00:00:00
url: https://github.com/imjuniper/UnrealDiscordGameSDK
img: ./discord-game-sdk-for-ue.png
filters:
  - Unreal Engine 5
  - C++
tags:
  - Plugin
  - Unreal Engine 5
  - C++
---

## Technologies used

- Unreal Engine 5.4-5.5
- [Discord Game SDK](https://discord.com/developers/docs/developer-tools/game-sdk) (their legacy SDK)

## Features

- Automatically download the Discord Game SDK binaries (they cannot be redistributed)
- Rich presence updates
- Game and guild invites
- Overlay toggling

## Overview

I've been wanting to mess with Discord's Rich Presence for a while now, but there was no Unreal Engine integration available, and third-party ones either weren't up to date or didn't structure things in a way that was optimal to me. So, obviously I decided to make yet another Discord integration plugin for Unreal. I will not upload it to Fab, because Discord now has its own plugin and I don't want the maintenance and support to be a burden.

Despite Discord having released a new plugin with an official Unreal Engine integration, I think my plugin is a bit better in some aspects. For starters, my plugin uses latent actions with different output execution pins for functions requiring callbacks, whereas the official plugin requires the developer to manually subscribe to events on their `DiscordLocalPlayerSubsystem`. Their new plugin also currently requires the user of the game/app to manually authenticate with Discord using OAuth, which makes sense when using features such as lobbies and friend lists, or when on a platform other than desktop. However, if you only want to use it for rich presence on desktop platform, this login step adds friction. The old Game SDK communicates directly with the Discord app via RPC, skipping the login step.

### Automatic download of the SDK binaries

After reading online about the SDK, I found out that Discord doesn't allow redistribution of their SDK, so I had to find a way to download the binaries straight from Discord in a user-friendly way. Inspired by the [Plugin Downloader plugin](https://github.com/Phyronnaz/PluginDownloader) downloads and extracts files, as well as the way the [Voxel Plugin](https://voxelplugin.com) creates a popup when starting the engine to register shader hooks, my plugin notifies the user if the DLL for its platform is missing, prompting them to automatically download it. Once the download is done, the user is prompted to restart the engine.

If the user doesn't download the SDK or if the DLLs aren't available for any other reason, the subsystem aborts its initialization and disable its tick. Its subobjects will then automatically return errors when calling their functions, instead of trying to call methods on invalid pointers and crashing.

### Creating designer-friendly BP functions

The Discord Game SDK uses a *lot* of async functions. In fact, the vast majority of calls require a callback function and don't return anything. The functions that don't need a callback will usually return a status code and require a pointer as a parameter to get the *actual* data you want, making it rather annoying to deal with. The worst part is that some callbacks are **never called**, despite the function obviously working. Finally, some of the functions are still in the SDK, but have been deprecated and no longer work at all. They are no longer documented on the website, but will come up in autocomplete results.

So with that in mind, I needed to wrap all the useful functions in a way that was easy to use for designers and programmers using only Blueprints. I first started by creating wrappers that were meant for C++ use, but using mostly Unreal types, for example:

```cpp
void UDiscordUserManager::GetUser(const int64 UserID, TFunction<void(discord::Result, discord::User const&)> Callback) const
{
	if (!DiscordSubsystem->IsActive())
	{
		Callback(discord::Result::InternalError, discord::User{});
		return;
	}

	Internal_UserManager->GetUser(UserID, Callback);
}
```

Using this function, I then created a `UFUNCTION(BlueprintCallable)` overload, that calls a generic subclass of `FPendingLatentAction`:

```cpp
void UDiscordUserManager::GetUser(const UObject* WorldContext, const FLatentActionInfo LatentInfo, const int64 UserID, FDiscordUser& User, EDiscordOutputPins& OutputPins) const
{
	FDiscordLatentAction::CreateAndAdd(WorldContext, LatentInfo, OutputPins, [this, UserID, &User](auto* Action) mutable
	{
		GetUser(UserID, [&User, Action](discord::Result Result, discord::User const& ResultUser) mutable
		{
			User = FDiscordUser(ResultUser);
			Action->FinishOperation(Result == discord::Result::Ok);
		});
	});
}
```

The `FDiscordLatentAction` first triggers the "regular" unnamed execution pin, so that the Blueprint function can continue and then calls the lambda function that was passed in. It will then wait until `FinishOperation(...)` is called or until a default 5 second timeout. Once that is done, it will call `Response.FinishAndTriggerIf(...)` and the corresponding execution pin will be called, with the output parameters having been set in the lambda function before calling `FinishOperation(...)`.

Using this, I was able to create functions that are intuitive and simple to call in Blueprint. No need to bind manually bind events, the result is available on the same node. The return code is not exposed to Blueprints, because I found that most of the time it doesn't provide any information that could be easily fixed at runtime. It is printed to the log as an error, for the developer to get more information. When using the C++ versions of the functions however, the callback will have it as a parameter.

### Supporting multiple Unreal Engine versions

I starting developing it on Unreal 5.5, but when I gave it to a friend using 5.4, they obviously encountered issues because the `SetTickableTickType()` function did not exist yet. Once again inspired by the Voxel Plugin, I [recreated their `UE_VERSION` macro](https://github.com/imjuniper/UnrealDiscordGameSDK/blob/4765fb8433f04b2bf6a5427848c33fe1943a65bd/Source/DiscordRuntime/Public/DiscordSubsystem.h#L14), giving me an easy way to support multiple engine versions at once. Using the macro, I disabled the calls to `SetTickableTickType()` and added a definition for `IsAllowedToTick()` when using versions older than 5.5. I haven't tested it on 5.3 and lower, but I'm fairly confident it won't have issues. I might go back and check support for them eventually, but it's not a priority currently.
