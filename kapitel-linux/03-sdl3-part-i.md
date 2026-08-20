# SDL3 - part I (Linux)

> **Linux:** This chapter is adapted for Linux.

SDL3 is not a .EXE. it's a collection of shared objects (.so) and header files that can be called on to access basic features like creating windows or accept input from the keyboard.

> [!NOTE]
> This course teaches game programming just about as "from scratch" as is educationally viable. Once we know how these systems operate and move through the semester we will off-load a bunch of this heavy lifting onto game engines and well groomed applications with a lot of large and shiny helpful buttons. This course aims to empower you by teaching you
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
- **include/** — holds header files (.h)
- **lib/** — holds any static libraries we might need
- **assets/** — sound effects, PNGs, etc
- **src/** — all script files (.cpp, .h) that we create ourselves

## Setting up SDL3

On Linux we install SDL3 and build tools directly from the package manager:

```bash
# Debian/Ubuntu
sudo apt install libsdl3-dev cmake ninja-build clang

# Fedora
sudo dnf install SDL3-devel cmake ninja-build clang

# Arch Linux
sudo pacman -S sdl3 cmake ninja clang
```

This installs SDL3 system-wide:
- Headers go to `/usr/include/SDL3/` or similar
- Libraries go to `/usr/lib/` or similar
- CMake's `find_package` can locate them automatically

No manual downloading or copying of DLLs needed.

## Editor

For this course, we will be writing our code in **your editor (e.g. nvim)** (Neovim). You can also use VS Code or any editor you prefer.

> [!NOTE]
> your editor (e.g. nvim) is a terminal-based editor. To quit: press Escape to ensure you're in Normal Mode, then type `:q` and press Enter.

## bash and environment variables

We will be using bash (the default shell on most Linux distributions). Our editor is already available via `$PATH` since it was installed through the package manager.

## Creating bash functions

We can create small functions that bash can call on. A function is a collection of commands that run in sequence. This is done by adding those functions to `~/.bashrc`.

Open it with:

```bash
your-editor ~/.bashrc
```

Add our functions at the bottom of the file.

### Our first function

```bash
function hello {
}
```

This is an empty function named `hello`. Any code between the curly braces `{ }` will be executed when the function runs.

Add some content:

```bash
function hello {
    echo "hello, sailor!"
}
```

Reload the profile:

```bash
source ~/.bashrc
```

Now try typing `hello` in the terminal.

### The `dev` function

The first useful function we'll make is a `dev` function. Its job is to start our text editor and open our game project's main script file, `main.cpp`.

First, create `main.cpp` in the `src/` folder:

```bash
touch src/main.cpp
```

Now add the `dev` function to `~/.bashrc`:

```bash
dev() {
    local project="$1"
    local config
    config=$(GetConfig "$project")

    if [[ -z "$config" ]]; then
        echo "Project '$project' not found."
        return 1
    fi

    local path
    path=$(echo "$config" | jq -r '.path')

    cd "$path/src" || return 1

    export GAMEPROJECT="$project"
    your-editor main.cpp
}
```

This function:
1. Gets a JSON config based on the project name
2. If the project doesn't exist in JSON, return with an error
3. Changes directory to the project's `src/` folder
4. Saves the project name to an environment variable
5. Opens the main file in your editor

## JSON configuration

We need a JSON file to store project data. Create `projects.json` in your PROJECTS folder:

```bash
touch ~/Projects/projects.json
```

Open it and add:

```json
{
    "radio": {
        "path": "/home/YOUR_USER/Projects/SDL_RADIO_GAME",
        "main_file": "main.cpp",
        "build_system": "cmake"
    },
    "pilot": {
        "path": "/home/YOUR_USER/Projects/HEARTBURNER",
        "main_file": "main.cpp",
        "build_system": "cmake"
    }
}
```

Replace `YOUR_USER` with your actual username.

Store the path to this JSON file in `~/.bashrc`:

```bash
export projectConfig="$HOME/Projects/projects.json"
```

### The `GetConfig` function

```bash
GetConfig() {
    local projectName="$1"
    jq -r ".\"$projectName\" // empty" "$projectConfig"
}
```

This function:
1. Takes a project name as parameter
2. Reads the JSON file using `jq` (a JSON parser for the command line)
3. Returns the data block for that project

## Installing build tools

If you haven't already, install the build tools:

```bash
# Debian/Ubuntu
sudo apt install cmake ninja-build clang

# Fedora
sudo dnf install cmake ninja-build clang

# Arch Linux
sudo pacman -S cmake ninja clang
```

> [!NOTE]
> LLVM contains a compiler called `clang++`. We will be using it from bash.

### Check installation

```bash
cmake --version
ninja --version
clang++ --version
```

### The build pipeline

```
cmake -> ninja -> clang++
```

- **CMake** uses `CMakeLists.txt` to learn about the project
- **Ninja** takes cmake's instructions and schedules compilation
- **Clang++** compiles the C++ code into machine code

## CMakePresets.json

Create `CMakePresets.json` in the project root:

```bash
touch CMakePresets.json
```

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

```bash
goto() {
    local project="$1"
    local config
    config=$(GetConfig "$project")
    local path
    path=$(echo "$config" | jq -r '.path')
    cd "$path" || return 1
}
```

### The `build` function

```bash
build() {
    local project="$1"
    local preset="${2:-default}"
    goto "$project" || return 1
    cmake --preset "$preset"
    cmake --build --preset "$preset"
}
```

## CMakeLists.txt

Create `CMakeLists.txt` in the project root:

```cmake
cmake_minimum_required(VERSION 3.25)
project(Heartburner LANGUAGES CXX)
set(CMAKE_CXX_STANDARD 20)

# Find SDL3 from system
find_package(SDL3 REQUIRED)

# Automatically find all .cpp files in src/
file(GLOB_RECURSE SOURCES "src/*.cpp")

# Create executable
add_executable(${PROJECT_NAME} ${SOURCES})

# Include directory
target_include_directories(${PROJECT_NAME} PRIVATE include)

# Link SDL3
target_link_libraries(${PROJECT_NAME} PRIVATE SDL3::SDL3)
```

> **Linux:** Instead of manually listing `.lib` files and copying `.dll`s, we use `find_package(SDL3 REQUIRED)` which locates the system-installed SDL3 and links the correct shared library automatically.

## First program

Create `src/main.cpp`:

```cpp
#include <SDL3/SDL.h>

int main() {
    SDL_Delay(2000);
    return 0;
}
```

> **Linux:** The book uses `#include <windows.h>` and `Sleep(2000)` — windows.h doesn't exist on Linux. SDL3 has `SDL_Delay()` which does the same thing.

Build and run:

```bash
build pilot
./build/Heartburner
```

The program does nothing for 2 seconds, then closes.

## Summary

You have now learned:
- JSON syntax
- bash scripting syntax
- CMake syntax
- How to use the terminal
- How to compile a game
- How to set up a development environment
- Environment variables
- File and folder structure for game projects
- How to work with text editors (your editor (e.g. nvim))
- The relationship between build systems (CMake, Ninja) and compilers (Clang++)
- How to install and integrate external libraries (SDL3 via package manager + find_package)
- How to create and use functions with parameters
- The concept of entry points in programming (main.cpp)
- How different file types work together (.so, .h, .cpp, .json, .txt)
- Basic project organization and directory management
- The importance of case-sensitivity in programming
- How to navigate and manipulate file paths on Linux