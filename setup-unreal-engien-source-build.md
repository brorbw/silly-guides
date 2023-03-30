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
