# SDL3 - part I

SDL3 is not a .EXE. it's a collection of scripts and DLL files that can be called on to access basic features like creating windows or accept input from the keyboard.

> [!NOTE]
> This course teaches game programming just about as "from scratch" as is educationally viable. Once we know how these systems operate and move through the semester we will off-load a bunch of this heavy lifting onto game engines and well groomed applications with a lot of large and shiny helpful buttons. This course aims to empower you by teaching you
>
> 1) how things work "behind the scenes"
> 2) how to not become dependant on a ready-made game engine
> 3) if you can do this, then nothing a LIA will throw at you will feel difficult in comparison
> 4) how to survive in the uncomfortable role as someone who needs to learn new things

## Project structure

To start working with a new SDL3 project we need to store it in a folder. This folder will now be called our "root". In the root folder we will need the following folders:

```
root/
├── build/
├── include/
├── lib/
├── assets/
└── src/
```

- **build/** — holds the compiled version of our game
- **include/** — holds header files (.h) that tell us what functions we can call from the .DLL files
- **lib/** — holds all our .DLL files
- **assets/** — sound effects, PNGs, etc
- **src/** — all script files (.cpp, .h) that we create ourselves

## Setting up SDL3

We need to download SDL3 from GitHub: https://github.com/libsdl-org/SDL/releases

Download `SDL3-devel-3.4.2-VC.zip`. Extract it with 7-Zip.

Back in our project root folder, inside the **lib** folder we copy:
- `SDL3.dll` and `SDL3.lib` from `lib/x64/`

In our **include** folder we add all the `.h` files from the devel's `include/` folder. Copy the entire `SDL3/` subdirectory.

Our root folder should now look like this:

```
root/
├── build/
├── include/
│   └── SDL3/
│       └── (all the .h files)
├── lib/
│   ├── SDL3.dll
│   └── SDL3.lib
├── assets/
└── src/
```

## Helix Editor

For this course, we will be writing our code in the Helix Editor: https://helix-editor.com/

Download the pre-built binary `helix-25.07.1-x86_64-windows.zip` and extract it to a folder on your computer.

> [!NOTE]
> 
> > We need to remember this location as we will be referencing this specific address on our computer soon.

Running `hx.exe` will bring up the Helix editor. To quit the editor (don't panic) type `:` to bring up the command line, type a single `q` and press enter.

Additionally, for simpler text editing and pseudo-code examples we will be working with Sublime Text: https://www.sublimetext.com/

## PowerShell and environment variables

We will be starting the Helix editor from the command line using Windows PowerShell. If your computer doesn't already have PowerShell installed, download it from Microsoft.

To give PowerShell access to our Helix editor, we need to create our first User Environment Variable. From the Control Panel on Windows, find System Properties → Environment Variables. Add a new entry to the `Path` list pointing to the Helix folder:

```
D:\helix-25.01.1-x86_64-windows
```

> [!NOTE]
> The `hx.exe` is not part of the Path — the *folder containing it* is.

Once set up, type `hx` inside PowerShell to bring up the Helix Editor.

## Creating PowerShell functions

We can create small functions that PowerShell can call on. A function is a collection of commands that run in sequence. This is done by writing those functions to the **profile** being used by PowerShell. Type `$profile` in PowerShell to find its path.

Right now it's empty. Open it with:

```powershell
hx $profile
```

Or if you prefer Sublime Text:

```powershell
subl $profile
```

### Our first function

```powershell
function hello {
}
```

This is an empty function named `hello`. Any code between the curly braces `{ }` will be executed when the function runs.

Add some content:

```powershell
function hello {
    'hello, sailor!'
}
```

Reload the profile:

```powershell
. $profile
```

Now try typing `hello` in the terminal.

### The `dev` function

The first useful function we'll make is a `dev` function. Its job is to start our text editor and open our game project's main script file, `main.cpp`.

First, create `main.cpp` in the `src/` folder:

```powershell
New-Item main.cpp
```

Now add the `dev` function to `$profile`:

```powershell
function dev {
    param([Parameter(Mandatory=$true)][string]$project)
    $config = GetConfig $project
    if ($null -eq $config) { return }

    Set-Location -Path "$($config.path)\src"
    $env:GAMEPROJECT = $project
    hx $config.main_file
}
```

This function:
1. Requires a project name
2. Gets a JSON config based on the project name
3. If the project doesn't exist in JSON, return (do nothing)
4. Changes directory to the project's `src/` folder
5. Saves the project name to an environment variable
6. Opens the main file in Helix

## JSON configuration

We need a JSON file to store project data. Create `projects.json` in your PROJECTS folder:

```json
{
    "radio": {
        "path": "D:\\SDL_RADIO_GAME",
        "main_file": "main.cpp",
        "build_system": "cmake"
    },
    "pilot": {
        "path": "D:\\PROJECTS\\HEARTBURNER",
        "main_file": "main.cpp",
        "build_system": "cmake"
    }
}
```

Store the path to this JSON file in `$profile`:

```powershell
$ConfigPath = "D:\PROJECTS\projects.json"
```

### The `GetConfig` function

```powershell
function GetConfig {
    param([string]$projectName)
    $config = Get-Content $ConfigPath | ConvertFrom-Json
    return $config.$projectName
}
```

This function:
1. Takes a project name as parameter
2. Reads the JSON file and converts it from JSON
3. Returns the data block for that project

## Installing build tools

We need a few more things before we can build:

1. **CMake** — Build System Generator: https://cmake.org/download/
2. **Ninja** — Build system: https://github.com/ninja-build/ninja/releases
3. **LLVM/Clang** — Compiler: https://github.com/llvm/llvm-project/releases
4. **Visual Studio Build Tools** with "Desktop Development with C++"

> [!NOTE]
> LLVM contains a compiler called `clang`. We will be using `clang++` in PowerShell.

### Check installation

```powershell
hx --version
subl --version
cmake --version
clang --version
```

Verify the target is `x86_64-pc-windows-msvc` (not MinGW).

### The build pipeline

```
cmake → ninja → clang++
```

- **CMake** uses `CMakeLists.txt` to learn about the project
- **Ninja** takes cmake's instructions and schedules compilation
- **Clang++** compiles the C++ code into machine code

## CMakePresets.json

Create `CMakePresets.json` in the project root:

```json
{
    "version": 3,
    "configurePresets": [
        {
            "name": "default",
            "generator": "Ninja",
            "binaryDir": "${sourceDir}/build",
            "cacheVariables": {
                "CMAKE_BUILD_TYPE": "Debug",
                "CMAKE_CXX_COMPILER": "clang++"
            }
        }
    ],
    "buildPresets": [
        {
            "name": "default",
            "configurePreset": "default"
        }
    ]
}
```

### The `goto` function

```powershell
function goto {
    param([string]$project)
    $config = GetConfig $project
    Set-Location -Path $config.path
}
```

### The `build` function

```powershell
function build {
    param(
        [Parameter(Mandatory=$true)][string]$project,
        [string]$preset_name = "default"
    )
    goto $project
    cmake --preset $preset_name
    cmake --build --preset $preset_name
}
```

## CMakeLists.txt

Create `CMakeLists.txt` in the project root:

```cmake
cmake_minimum_required(VERSION 3.25)
project(Heartburner LANGUAGES CXX)
set(CMAKE_CXX_STANDARD 20)

# Automatically find all .cpp files in src/
file(GLOB_RECURSE SOURCES "src/*.cpp")

# Create executable
add_executable(${PROJECT_NAME} ${SOURCES})

# Add include directory
target_include_directories(${PROJECT_NAME} PRIVATE include)

# Automatically find and link all .lib files in lib/
file(GLOB_RECURSE LIB_FILES "${CMAKE_SOURCE_DIR}/lib/*.lib")
target_link_libraries(${PROJECT_NAME} PRIVATE ${LIB_FILES})

# Automatically copy all .dll files to executable directory
file(GLOB_RECURSE DLL_FILES "${CMAKE_SOURCE_DIR}/lib/*.dll")
add_custom_command(TARGET ${PROJECT_NAME} POST_BUILD
    COMMAND ${CMAKE_COMMAND} -E copy_if_different
    ${DLL_FILES}
    "$<TARGET_FILE_DIR:${PROJECT_NAME}>"
)
```

## First program

Create `src/main.cpp`:

```cpp
#include <windows.h>

int main() {
    Sleep(2000);
    return 0;
}
```

Build and run:

```powershell
build pilot
```

Double-click the executable in the `build/` folder. The program does nothing for 2 seconds, then closes.

## Summary

You have now learned:
- JSON syntax
- PowerShell scripting syntax
- CMake syntax
- How to use the terminal
- How to compile a game
- How to set up a development environment
- Environment variables (User vs System)
- File and folder structure for game projects
- How to work with text editors (Helix, Sublime Text)
- The relationship between build systems (CMake, Ninja) and compilers (Clang++)
- How to download and integrate external libraries (SDL3)
- How to create and use functions with parameters
- The concept of entry points in programming (main.cpp)
- How different file types work together (.dll, .h, .cpp, .json, .txt)
- Basic project organization and directory management
- The importance of case-sensitivity in programming
- How to navigate and manipulate file paths in Windows