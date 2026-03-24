# MATLAB R2025b + Unreal Engine 5.3 Co-Simulation Setup
**Author:** Veera Tanish Reddy Doorsala — CAV Team

This guide walks through setting up the MATLAB/Unreal co-simulation with live LiDAR. Each step has the normal instructions followed by any problems I ran into so you know what to expect.

---

## What You Need

- **MATLAB R2025b** — Automated Driving Toolbox, Vehicle Dynamics Blockset, Lidar Toolbox, Simulink 3D Animation
- **Unreal Engine 5.3** — via Epic Games Launcher
- **Visual Studio 2022**

---

## Step 1 — Install Visual Studio 2022

When installing, make sure these workloads are checked:
- Desktop development with C++
- Game development with C++
- Windows 10/11 SDK
- MSVC v143 build tools

If VS is already installed without these, open the **Visual Studio Installer** → **Modify** → add them.

> **Problem:** Midway through the project build, VS said it needed more tools. It was the "Game development with C++" workload missing. Just add everything listed above upfront.

---

## Step 2 — Install the MATLAB Support Package

1. MATLAB → **Home → Add-Ons → Get Add-Ons**
2. Search and install: **Automated Driving Toolbox Interface for Unreal Engine Projects**
3. After installing, run this to find where MATLAB stored the files — you'll need this path later:

```matlab
matlabshared.supportpkg.getSupportPackageRoot
```

> **Problem:** Navigating to `C:\ProgramData` in File Explorer said "can't find path." That folder is hidden by default.
>
> Fix — paste this into the File Explorer address bar:
> ```
> %ProgramData%\MATLAB\SupportPackages\R2025b
> ```
> Or let MATLAB open it directly:
> ```matlab
> winopen(fullfile(matlabshared.supportpkg.getSupportPackageRoot, 'toolbox', 'shared', 'sim3dprojects', 'spkg'))
> ```

---

## Step 3 — Set MATLABROOT Environment Variable

The Unreal plugin needs to know where MATLAB is installed or the build will fail.

1. Windows Key → search "Environment Variables" → **Edit the system environment variables**
2. Click **Environment Variables** → under **System Variables** → **New**
   - Name: `MATLABROOT`
   - Value: `C:\Program Files\MATLAB\R2025b`
3. Click OK → **restart your computer** (required)

> **Problem:** Build failed with:
> ```
> MATLABROOT environment variable appears unset
> (did you open the project from within MATLAB?)
> ```
> I had set the variable but hadn't restarted yet. The restart is what makes VS see it.
>
> You can also run this in MATLAB before building as a backup:
> ```matlab
> setenv('MATLABROOT', matlabroot)
> ```

---

## Step 4 — Extract the AutoVrtlEnv Project

The parking lot scene is bundled inside MATLAB. Copy it to a local folder:

```matlab
sim3d.utils.copyExampleSim3dProject('C:\My_Lidar_Project')
```

You should now have `C:\My_Lidar_Project\AutoVrtlEnv\AutoVrtlEnv.uproject`.

> ⚠️ No spaces in the path. `C:\My Lidar Project\` will cause issues later.

> **Problem:** Tried using a hardcoded `v7.1` version path and got `Error using winopen: file not found`. That subfolder name changed in R2025b.
>
> Fix — use the broader path and browse manually:
> ```matlab
> winopen(fullfile(matlabshared.supportpkg.getSupportPackageRoot, 'toolbox', 'shared', 'sim3dprojects', 'spkg'))
> ```

---

## Step 5 — Copy the MathWorks Plugins to Unreal

Create this folder if it doesn't exist:
```
D:\UE_5.3\Engine\Plugins\Marketplace\MathWorks\
```

Open the plugins source folder:
```matlab
winopen(fullfile(matlabshared.supportpkg.getSupportPackageRoot, 'toolbox', 'shared', 'sim3dprojects', 'spkg', 'plugins'))
```

Go inside each container folder and copy the inner plugin folder directly into `MathWorks\`:

| Open this | Copy this inner folder | Paste into MathWorks\ |
|---|---|---|
| `mw_simulation\` | `MathWorksSimulation\` | `MathWorks\MathWorksSimulation\` |
| `mw_automotive\` | `MathWorksAutomotiveContent\` | `MathWorks\MathWorksAutomotiveContent\` |
| `rr_materials\` | `RoadRunnerMaterials\` | `MathWorks\RoadRunnerMaterials\` |

Final structure should look like:
```
MathWorks\
├── MathWorksSimulation\
├── MathWorksAutomotiveContent\
└── RoadRunnerMaterials\
```

> **Problem:** Copied the folders including the `mw_simulation\` wrapper which gave the wrong nested structure:
> ```
> MathWorks\mw_simulation\MathWorksSimulation\   ← wrong
> MathWorks\MathWorksSimulation\                 ← correct
> ```
> Unreal kept throwing "module could not be loaded" until fixing the structure.

> **Problem:** Folder name capitalization matters. `mathworkssimulation` won't work — has to be `MathWorksSimulation` with capital M, W, S.

---

## Step 6 — Verify the Binaries Folder

Check that `MathWorksSimulation` has a `Binaries` subfolder:
```
MathWorksSimulation\
├── Binaries\
│   └── Win64\
│       └── UnrealEditor-MathWorksSimulation.dll
├── Content\
├── Source\
└── MathWorksSimulation.uplugin
```

> **Problem:** Even after copying the plugins correctly, still got "module could not be loaded." The `Binaries` folder was completely missing.
>
> The `mw_simulation` container only has source files. The compiled `.dll` files are in a different location:
> ```matlab
> winopen(fullfile(matlabshared.supportpkg.getSupportPackageRoot, 'toolbox', 'shared', 'sim3d', 'plugins', 'UE5.3', 'MathWorksSimulation'))
> ```
> Copy this entire folder (it has `Binaries\` inside) and overwrite the one in `MathWorks\`.

---

## Step 7 — Associate .uproject Files with Unreal

If `AutoVrtlEnv.uproject` has a blank white icon, Windows doesn't know what to open it with.

1. Right-click `AutoVrtlEnv.uproject` → **Open with → Choose another app**
2. Navigate to `D:\UE_5.3\Engine\Binaries\Win64\UnrealEditor.exe`
3. Check **"Always use this app"** → OK

---

## Step 8 — Generate Visual Studio Project Files

Run in Command Prompt:
```cmd
"D:\UE_5.3\Engine\Binaries\DotNET\UnrealBuildTool\UnrealBuildTool.exe" -projectfiles -project="C:\My_Lidar_Project\AutoVrtlEnv\AutoVrtlEnv.uproject" -game -rocket -progress
```

This creates `AutoVrtlEnv.sln` in your project folder.

> **Problem:** Right-clicking the `.uproject` file had no "Generate Visual Studio project files" option. The Unreal shell extension wasn't registered because Unreal was installed on `D:\` instead of `C:\`.
>
> Fix — run the CMD command above directly. It does the same thing.

---

## Step 9 — Build the C++ Binaries

```cmd
devenv "C:\My_Lidar_Project\AutoVrtlEnv\AutoVrtlEnv.sln" /Build "Development Editor|Win64"
```

Wait for `Build: 1 succeeded`.

> **Problem:** "AutoVrtlEnv could not be compiled. Try rebuilding from source manually." The root cause was `MATLABROOT` not being set (Step 3). After restarting with the variable set and rerunning the command, it compiled fine.

> **Problem:** Inside Visual Studio, "Development Editor" in the dropdown either didn't appear or couldn't be selected.
>
> Fix — close VS, delete `AutoVrtlEnv.sln`, the `.vs\` folder, and `Intermediate\` from the project folder, rerun Step 8, open the fresh `.sln`.

> **Problem:** Couldn't delete the `Saved` folder — "folder in use" even after closing Unreal.
>
> Fix — open Resource Monitor (`Windows + R` → `resmon`) → CPU tab → Associated Handles → search for the folder name → right-click the process → End Process.
>
> Or force it:
> ```cmd
> rmdir /s /q "C:\My_Lidar_Project\AutoVrtlEnv\Saved"
> ```

---

## Step 10 — Open the Project in Unreal

Double-click `AutoVrtlEnv.uproject` or click **Open Unreal Editor** from inside the Simulink Scene Configuration block.

If it says "Modules are missing — rebuild?" → click **Yes**.

> **Problem:** "Unable to open this project, as it was made with a newer version of Unreal Engine" when opening in UE 4.27. R2025b only works with UE 5.3.

> **Problem:** The "Open Unreal Editor" button in Simulink was greyed out or did nothing.
>
> Fix — kill any background Unreal processes in Task Manager first. Make sure the `Project` field points to your `.uproject`. Run:
> ```matlab
> setpref("Simulation3D", "PluginVersion", "25.1.0")
> ```
> If still nothing, bypass it:
> ```matlab
> editor = sim3d.Editor('C:\My_Lidar_Project\AutoVrtlEnv\AutoVrtlEnv.uproject');
> open(editor);
> ```

---

## Step 11 — Load the Parking Lot Map

Unreal opens to an empty scene by default. Load the actual map:

1. Press `Ctrl+Space` to open the Content Drawer
2. Search `LargeParkingLot`
3. Double-click the icon with the **orange bar** at the bottom
4. Click **Don't Save** if it asks about the current scene

To avoid doing this every time: `Edit → Project Settings → Maps & Modes` → set both startup maps to `LargeParkingLot`

> **Tip:** If the checkerboard texture issue is bothering you, try the **Highway** map instead — it loaded with full textures immediately without any shader compilation wait.

> **Problem:** Search returned nothing. Fix — in the Content Drawer click the **gear icon** → check **Show Plugin Content** → search again.

> **Problem:** Simulation ran but was in a void — no ground, no parking lot. The `EmptyScene` was still loaded. Stop the simulation, load `LargeParkingLot`, restart.

---

## Step 12 — Configure the Simulink Model

This is what the final model looks like:

<img width="1551" height="582" alt="Screenshot 2026-03-10 181551" src="https://github.com/user-attachments/assets/4ce404f3-35d8-47cf-b4c6-b67e291458c4" />


**Simulation 3D Scene Configuration**
- Scene source: `Unreal Editor`
- Project: `C:\My_Lidar_Project\AutoVrtlEnv\AutoVrtlEnv.uproject`
- Sample time: `1/60`

**Simulation 3D Vehicle with Ground Following**
- Actor name: `SimulinkVehicle1`
- Connect Constant blocks to X, Y, Yaw: X=`20`, Y=`0`, Yaw=`0`

**Simulation 3D LiDAR**
- Parent name: `SimulinkVehicle1`
- Connect `Point Cloud` output → `Point Cloud` input on the Point Cloud Viewer block
- Connect `Distance` output → `Intensity` input on the Point Cloud Viewer block

> **Problem:** Simulation opened in the built-in Simulation 3D Viewer instead of Unreal. Fix — change `Scene source` to `Unreal Editor` and set the project path.

> **Problem:** Simulation stuck on "Loading" forever. The X, Y, Yaw ports had nothing connected. Connect those Constant blocks.

> **Problem:** "Check the position of vehicle — large variation in terrain." Either EmptyScene was loaded or the car spawned at (0,0,0). Load the parking lot map and set X=20.

---

## Step 13 — Set Stop Time to inf

In the Simulink toolbar change Stop Time from `10.0` to `inf`.

> **Problem:** Simulation kept stopping after 10 seconds. That's just the default stop time.

---

## Step 14 — Run the Simulation

Order matters:

1. Open Unreal — confirm the parking lot is loaded
2. Set Stop Time to `inf`
3. Click **Run** in Simulink — wait for status bar to say `Initializing`
4. Click the green **Play ▶** in Unreal

The car appears in Unreal and the Point Cloud Viewer in Simulink shows live LiDAR dots.

> **Problem:** Unreal froze completely after clicking Play. Couldn't click anything.
>
> Why — Play was clicked before Run in Simulink. Unreal waited for a signal from MATLAB that never came.
>
> Fix — `Alt+Tab` to Simulink → click Stop. If that doesn't work, Task Manager → `UnrealEditor.exe` → End Task. Always Run Simulink first, then Play Unreal.

> **Problem:** "Incompatible version of 3D Simulation engine: 25.2.0. Allowed versions are 25.1.0."
>
> Fix:
> ```matlab
> setpref("Simulation3D", "PluginVersion", "25.1.0")
> ```
> Or open `MathWorksSimulation.uplugin` in Notepad and change `"VersionName": "25.2.0"` to `"VersionName": "25.1.0"`.

> **Problem:** `sim3d.utils.checkEnvironment` returned "Unable to resolve the name." That function doesn't exist in this version. Use the `setpref` command above instead.

---

## Step 15 — Make the Car Move (Optional)

Replace the Constant block on X with a **Ramp** block, Slope = `5`. Keep Y and Yaw as Constants at `0`. The car drives forward and the LiDAR scan updates in real time.

> **Problem:** Car wasn't moving even with the simulation running. A Constant holds the car at a fixed position. Swap it for a Ramp.

---

## Step 16 — Known Visual Issue

The parking lot might look like a gray checkerboard on first load. Unreal is compiling shaders. The LiDAR data is fully accurate during this — it's purely a visual thing.

Wait for "Compiling Shaders (###)" in the bottom-right of Unreal to hit 0. Or `Build → Build Reflection Captures` while stopped.

If the checkerboard is annoying just use the **Highway** map — it loaded with full textures right away.

FPS spikes when switching windows — Unreal throttles itself when not focused. Fix: `Edit → Editor Preferences` → search "Background" → uncheck **Use Less CPU when in Background**.

---

## All Commands

```matlab
% Find support package root
matlabshared.supportpkg.getSupportPackageRoot

% Open spkg folder
winopen(fullfile(matlabshared.supportpkg.getSupportPackageRoot, 'toolbox', 'shared', 'sim3dprojects', 'spkg'))

% Open plugin Binaries source
winopen(fullfile(matlabshared.supportpkg.getSupportPackageRoot, 'toolbox', 'shared', 'sim3d', 'plugins', 'UE5.3', 'MathWorksSimulation'))

% Extract project
sim3d.utils.copyExampleSim3dProject('C:\My_Lidar_Project')

% Fix version mismatch
setpref("Simulation3D", "PluginVersion", "25.1.0")

% Set MATLABROOT for current session
setenv('MATLABROOT', matlabroot)

% Manually open Unreal (bypasses broken button)
editor = sim3d.Editor('C:\My_Lidar_Project\AutoVrtlEnv\AutoVrtlEnv.uproject');
open(editor);
```

```cmd
:: Generate VS project files
"D:\UE_5.3\Engine\Binaries\DotNET\UnrealBuildTool\UnrealBuildTool.exe" -projectfiles -project="C:\My_Lidar_Project\AutoVrtlEnv\AutoVrtlEnv.uproject" -game -rocket -progress

:: Build binaries
devenv "C:\My_Lidar_Project\AutoVrtlEnv\AutoVrtlEnv.sln" /Build "Development Editor|Win64"

:: Force delete locked folder
rmdir /s /q "C:\My_Lidar_Project\AutoVrtlEnv\Saved"

:: Verify MATLABROOT is set
echo %MATLABROOT%
```

---

*Veera Tanish Reddy Doorsala — CAV Team*
