# Setup unreal engine source build

# Installation

1. Clone the unreal engine source repo https://github.com/EpicGames/UnrealEngine
2. Move to the cloned location and run
```sh
Setup.sh
GenerateProjectFiles.sh
```
3. Build the shader compiler
The guide states that you need xcode for this but since I don't like xcode (especially because it hogs 70GB of my available 32GB ram o.O) we use commandline instead
```sh
xcodebuild -scheme ShaderCompileWorker build
```
4. Build the engine
```sh
xcodebuild -scheme UE5 build #takes a while
```

5. Place projet inside the root of the `./Engine` folder. And you are good to go.

# Import project

There are two ways to get a project to compile with the new engine.

## Open through editor
You can convert previous projects through `Engine/Binaries/Mac/unrealEditor`.
1. Open `Engine/Binaries/Mac/unrealEditor`
2. Click the project you want to open in the new engine
3. Ignore all the errors when you try to open the project.
4. Now you can
   1. Open with xcode and build
   2. Generate project files and build with `xcodebuild`

## Copy project to engine source root
If you copy your project to the same directory as `Engine` e.g. the root of the source directory.

1. `cd` into the newly created project directory
2. Generate project files including option (maybe with `--Engine` option)
3. Now you can
   1. Open with xcode and build
   2. Generate project files and build with `xcodebuild`
   3. Generate project files and build make

# Quixel not working with source build on macOS

A commit results in a segfault and should be removed
```sh
git show 28baa676467d1c86c03e657f043262904358e815
```

Building the source engine skips building a backend daemon for the bridge. So the following needs to be copied from a real engine release.
```sh
cp -r /Users/Shared/Epic Games/UE_5.2/Engine/Plugins/Bridge/ThirdParty/Mac/node-bifrost.app\
      ~/UnrealEngine/Engine/Plugins/Bridge/ThirdParty/Mac/node-bifrost.app
```
