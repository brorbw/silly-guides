# Emacs and Unreal Engine
Disclamer#1: Im a snowflake and this is for the snowflakes.

Quick preamble. I have been using my hackintosh for playing around with Unreal for a while. I like macOS and I like gamedevelopment.
A week ago I started to get into the c++ documentation and was surprised that you as a developer are forced to use `Xcode`, `CLion` or `VSCode`. I personally find the `Xcode` ecosystem very limited in terms of customization. Having used `evil` since always (vim keybindings for emacs) `Xcode` makes it hard to navigate and I tend to avoid non-modal editing. `CLion` is a good project in my opinion but again a bit limiting. `VSCode` have some of the expandability and customization I'm looking for in an editor. Mostly avoiding the mouse as much as possible. I used `VSCode` for about a week and it would sometimes just puke errors even tough everything would build fine. Before I started using `VSCode` I tried my luck with the two guides on how to integrate Emacs in an UE development workflow. No luck.

There are two main issues I faced. One was editing preferably with intellisense the other was building. I don't mind not having building integrated in my editor. I can just use `XCode` for building targets. Editing without intellisense however is a thing of the past IMO.

## Code completion

For emacs there are a _few_ ways to get code-completion thus I won't bore you with the details. I considered a few ways to get code-completion. Using `rtags`, `ccls` or `clangd`.

In order to use either of them I would need to build an index for the project. To do that I would need either a `Makefile` or `CMakeLists.txt`. 

To generate those files we have to use the `GenerateProjectFiles.sh` from `/Users/Shared/Epic Games/UE_5.0EA/Engine/Build/BatchFiles/Mac`. The generator has options for both `-Makefile` and `-CMakefile` and both will work. Actually there are a few more options than available through the editor and they can be found [here](https://github.com/EpicGames/UnrealEngine/blob/99b6e203a15d04fc7bbbf554c421a985c1ccb8f1/Engine/Source/Programs/UnrealBuildTool/UnrealBuildTool.cs#L400-L413)

The command will look something like for CMake

    /Users/Shared/Epic\ Games/UE_5.1/Engine/Build/BatchFiles/Mac/GenerateProjectFiles.sh \
	-project="/path/to/unreal/project/name.uproject" -game -CMakefile # This should be absolute path


Now we have a CMake file or a Makefile. While there is no issue when using either the `-CMakefile` or even `-CLion` option to generate the `CMakelists.txt` the `-Makefile` flag has some issues because silly Unreal thinks we are using Linux. In order to fix this we have to edit the `Makefile`. 

    sed -i -E "s/^PROJECT.*//g" Makefile
    sed -i -E "s/Linux/Mac/g" Makefile
    sed -i -E "s/PROJECTBUILD/BUILD/g" Makefile

In order for any of the proposed ways for code-completion to work we must have a `compile_commands.json` in our project directory. To generate those:

For CMake

    cmake -DCMAKE_EXPORT_COMPILE_COMMANDS=ON .

For Make we need one more step. We need `bear`.

    make clean
    bear -- make
	
Then we have the `compile_commands.json` and are ready to use them with our preferred way for code-completion. I would refrain from using `ccls` since it will just crash Emacs. I have not tried `rtags`.

Using this setup I get code-completion way faster and better than any of the _supported_ tools and I can jump to Unreal headerfiles with easy. Thus developing for Unreal went from a PITA to a real joy. 

# Building
Since we now have exported a `Makefile` compiling is as easy as running `make <ProjectName>`.

Hope this helps.


## Update
After having used this for a couple of days I have some findings. First and formost. Creating the `CMake` project makes is very easy for both `ccls` and `clangd` to find everying. However the makefile that `CMake` generate is big, very big. Thus I would reccomend using both methods. Use `CMake` to generate `comple_commands.json` then delete the `Makefile` and generate a new one with the build tool. Replace the strings with sed as shown above. This will improve your building times. Otherwise building will be painfully slow. If you want Unreal to hotload you should use the target named `<projectname>Editor`. 

    ninja DEMOEditor  14.21s user 6.07s system 93% cpu 21.752 total
    make HELEditor  7.81s user 2.77s system 129% cpu 8.150 total

______________

# UPDATE

I have created a small script that should bootstrap a UE project to use with `make` and `compile_commands.json` for editors like `VIM` and `Emacs`

```sh
#!/usr/bin/zsh

# First we create the "good" compile_commands.json
/Users/Shared/Epic\ Games/UE_5.1/Engine/Build/BatchFiles/Mac/GenerateProjectFiles.sh \
	-project="$PWD/$(basename $PWD).uproject" -game -CMakefile

cmake -DCMAKE_EXPORT_COMPILE_COMMANDS=ON .

# Remove the bloated Makefile
rm Makefile

# Create a simple makefile
/Users/Shared/Epic\ Games/UE_5.1/Engine/Build/BatchFiles/Mac/GenerateProjectFiles.sh \
	-project="$PWD/$(basename $PWD).uproject" -game -Makefile

# Correct the the makefile becase we are _not_ on Linux

sed -i -E "s/^PROJECT.*//g" Makefile
sed -i -E "s/Linux/Mac/g" Makefile
sed -i -E "s/PROJECTBUILD/BUILD/g" Makefile

# We are all set now

echo "Use the command:"
echo "make $(basename $PWD)Editor"
echo "to get hotloading"
```
