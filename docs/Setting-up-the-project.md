---
weight: -2
---

# Setting up the Virtual Anatomy repository

To begin development of this application, you need to retrieve Virtual Anatomy project from the [GitHub repository](https://github.com/jrcz-data-science-lab/Virtual-Anatomy-UE) and download [Unreal Engine](https://www.unrealengine.com/en-US) (at least version 5.5), which can be found in the [Epic Games Launcher](https://store.epicgames.com/en-US/download).

!!! Note 

    If you use 'git clone' in order to retrieve this project from GitHub, we advise you to increase git buffer size beforehand so that the clone process is succesful:

    git config --global http.postBuffer 524288000

    (In this case it is 500MB buffer; adjust as needed, but this value should work in this case)

## Required Unreal modules & plugins

The game module (`Source/VirtualAnatomy/VirtualAnatomy.Build.cs`) depends on several engine modules and plugins:

- **Public dependency modules** (used at runtime and in editor):
  - `Core`, `CoreUObject`, `Engine`
  - `InputCore`
  - `HTTP`, `Json`, `JsonUtilities` (quiz API and JSON payloads)
  - `EnhancedInput` (modern input mapping for camera/user controls)
  - `Niagara` (blood and diagnosis visual FX)

- **Private dependency modules**:
  - `Slate`, `SlateCore` (UI framework)
  - `UMG` (UI widgets: menus, sidebars, tree view, quiz UI, etc.)

- **Editor-only private modules** (only for the Editor target):
  - `UMGEditor`
  - `ToolWidgets`

On a fresh Unreal 5.5 install these modules are available by default, but you should ensure the following plugins are **enabled** in your project:

- **Niagara**
- **Enhanced Input**

If you see build errors complaining about missing modules (e.g. `Niagara` or `EnhancedInput`), double-check that these plugins are enabled under **Edit → Plugins** and that you are using a compatible UE 5.5 installation.

> Renderer note: the choice of renderer (e.g. Vulkan vs DirectX 12) is **not** configured in `VirtualAnatomy.Build.cs`. It is controlled via project config, typically in `Config/DefaultEngine.ini` under the Windows target settings (for example by setting `DefaultGraphicsRHI=DefaultGraphicsRHI_Vulkan`).

## Windows

For Windows, you will also need [Visual Studio](https://visualstudio.microsoft.com) with the following workload items added:

- Game Development with C++
- .NET SDK (can be installed by enabling .NET desktop development)

Once you have everything set up, you can launch the project by opening the `VirtualAnatomy.uproject` file with Unreal Engine.

To add the necessary files to open the project with Visual Studio, run `Tools > Generate Visual Studio Project`. Then, open the project in Visual Studio by opening the generated `VirtualAnatomy.sln` file.

## MacOS

For macOS install [Rider](https://www.jetbrains.com/rider/), and launch the project by opening the `VirtualAnatomy.uproject` with Rider, and... that's it?

## Linux

### Getting unreal engine
We have to compile unreal engine from source-code in order to use necessary plugins 

1. Visit unreal engine site and request their source code from [this](https://www.unrealengine.com/en-US/ue-on-github) link they have a guide there how to do that
2. Once you have the source code you can fork the repo and clone it directly to the desired directory on your compute, i suggest that you do it under `~/Software/`

```bash
git clone <repo url>.git
```

3.  When this is done go to the directory of the cloned game engine

```bash
cd ~/Software/UnrealEngine
```

4. Execute the following command to setup unreal engine for compilation

```shell
./Setup.sh
```
This should take couple of minutes

5. Now execute another bash script to generate the project files 

```shell
./GenerateProjectFiles.sh
```

6. Now run the make file to compile the engine 
```shell
make
```
This takes around 2 hours so, you only have to wait

7. After this is done you can execute Unreal Editor and select any template to see if everything works

```shell
 cd ~/Software/UnrealEngine/Engine/Binaries/Linux
 ./UnrealEditor
```

### Getting Jet Brains Rider

1. Download Jet Brains rider from [this link](https://www.jetbrains.com/rider/download/#section=linux)  now go the the directory where `.tar.gz` file downloaded and execute following command 

```bash
sudo tar -xzf Rider.tar.gz -C /opt
```

>NOTE: the `Rider.tar.gz` can have various letters and strings after each, just change the file name to match the one you downloaded

2. Navigate to `opt` directory and execute  the following commands
```shell
cd ./opt/JetBrains Rider-2024.2.3 #can be different version

./Rider.sh
```

This will open your The jet brains rider, feel free to create desktop shortcut if you want from **Tools -> Create Desktop Entry**

### Compiling plugins 

1. Download the epic assets manager from the Flat hub

```shell
flatpak run io.github.achetagemes.epic_asset_manager
```

2. Open the epic assets manager and enter your epic login info.
3. Open the market place in bottom left corner of the application 
4. Search for the `VaRest` plugin and Download it, it should be included in `~/Document/Epic Vault/VaRestPlugin_5.5`
5. Go to the `VaRestPlugin_5.5/data/Engine/Plugins/Marketplace/VaRestPlugin`
6. Open file `VaRest.uplugin` in any text editor and add `"Linux"` to the `PlatformAllowList`, if in the future plugins there is no modules add it there, this one should have it though
```txt
"Modules": [
		{
			"Name": "VaRest",
			"Type": "Runtime",
			"LoadingPhase": "PreDefault",
			"PlatformAllowList":[
				"Linux"
			]
		},
		{
			"Name": "VaRestEditor",
			"Type": "UncookedOnly",
			"LoadingPhase": "Default",
			"PlatformAllowList":[
				"Linux"
			]
		}
```

7. Make a copy of your Unreal Engine folder as it can broke to the point you would have to rebuild entire engine if we use it for the compilation of the plugin.

```shell
cd ~/Software
cp UnrealEngin ./UnrealEngine-cpy -rrm 
```

8.  Navigate to the batch files of the copied engine and compile the plugin
```shell
cd ~/Software/UnrealEngine-cpy/Engine/Build/BatchFiles

./RunUAT.sh BuildPlugin -Plugin="/home/name/Documents/EpicVault/VaRestPlugin_5.5/data/Engine/Plugins/Marke
tplace/VaRestPlugin/VaRest.uplugin" -Package="/home/name/Documents/EpicVault/VaRestPlugin_5-build" -TargetPlatforms=Linux

# package is the directory where you want the compiled plugin to be in
```

This will again take around 1 hour to compile 

Once it is done move the plugin from its compiled directory to the `~/Software/UnrealEngine/Engine/Plugins/Markteplace` if there is no market place directory create it 

In case of any issues of compiling the plugin refer to [this](https://youtu.be/G8T8T9qlEy8?t=186) video. 

If you can not start your unreal engine after you have done this or getting error that the some library is missing, you unfortunately must remove the engine and recompile it from scratch.

### Running the project 

Now we can finally get to the project we are supposed to be running. Clone the project from the organisation's  repository.

Now open the rider use it to open the file called  `ThreeDAnatomy.uproject` if you get errors that there is no engine associated with this project or that the project could not be loaded navigate to the  `~/.config/EpicGames/UnrealEngine` and use open `Install.ini`, there you should see something like this 

```txt
[Installations]
{61290131-F9C9-4A40-9CB3-202F4B612D47}=/home/name/Software/UnrealEngine
```

Add another line with following content

```txt
UE_5.5=/home/wpsimon09/Software/UnrealEngine
```

Now open the `ThreeDAnatomy.uproject` and change the `"EngineAssociation"` key to have the value of `5.4` 

```text
{
        "FileVersion": 3,
        "EngineAssociation": "5.5",
        "Category": "",
        "Description": "",
//other stuff
}
```

Now close the rider and reopen the `ThreeDAnatomy.uproject` with it, it might still display that it was not loaded but eventually after quite some time it should load it, the loading is signalised by the progress bars on the bottom left of Rider interface  

Note that it loaded both the project and engine.


![[Rider-interface-loaded-project-and-engine.png]]

Now you have to build the project by right clicking on the `ThreeDAnatomy` and selecting `BuildSelectedProjects` 

>IMPORTANT: Never select the option to build the solution as it will rebuild entire Unreal Engine together with it

Once everything is build you can press play in top right corner (or debug) it will open the unreal editor where you can start working
