# DLLs Memory and Hot Reloading - Part I

Ok! So we've managed to get things to interact, move and get rendered in SDL3. That is fantastic. It was a long journey to get here. During this and in upcoming lectures we will focus on 3 things:

1. Learning how to write gameplay code
2. Learning more about performance
3. Learning more about C++

This lecture will teach us how to expand our Cmakelists.txt to generate not only our .exe but also a .dll that will be responsible for holding most of our game, making our .exe just a very small entry point.

Why do we want to do this? Because we want to enable something called hot-reloading. (https://zylinski.se/posts/hot-reload-gameplay-code/) "Hot reloading gameplay code means that you swap out the code that controls the behavior of your game while the game is running. Why? To improve and tweak your gameplay code without having to restart the game."

Without this set up we have to stop running our .exe to make changes to the game, then recompile the game and run it again, getting back to the gamestate we're looking for. This becomes so useful when we want to make adjustments to parts of the game that happens a bit into our game, or requires a lot of tweaking to get right.

Here's the breakdown of how we will achieve this:

1. Change our cmakelists.txt to compile our project differently, creating the .exe and our new .dll
2. Set up what is called a memory arena to hold all the memory we are allowing our game to use
3. Break that block of memory into pieces we can use
4. Call into our .dll from our .exe from our main() function and pass along our memory arena
5. Write a reload function for PowerShell

Doing all these steps will break our program for a while as these changes are part of a bigger sweeping change. So for a while nothing will successfully compile.

Lets begin by looking at a new version of our cmakelists.txt it has the following changes:

1. It has been cleaned up and things are sorted in more manageable blocks
2. We create both a .EXE and a .DLL
3. We flag some scripts as being for the .EXE and the rest as belonging to the .DLL
4. We use comments to help distinguish the different parts of our cmakelists.txt

Here it is in full:

```cmake
cmake_minimum_required(VERSION 3.25)
project(Heartburner LANGUAGES CXX)
set(CMAKE_CXX_STANDARD 20)
set(CMAKE_EXPORT_COMPILE_COMMANDS ON)
set(DLL_NAME ${PROJECT_NAME}_game)
# Flag .EXE specific files
set(EXE_EXCLUSIVE
  ${CMAKE_SOURCE_DIR}/src/main.cpp
  ${CMAKE_SOURCE_DIR}/src/arena.cpp
)
# Create the file references
file(GLOB_RECURSE LIB_FILES "${CMAKE_SOURCE_DIR}/lib/*.lib")
file(GLOB_RECURSE DLL_FILES "${CMAKE_SOURCE_DIR}/lib/*.dll")
file(GLOB_RECURSE DLL_EXCLUSIVE "src/*.cpp")
list(REMOVE_ITEM DLL_EXCLUSIVE ${EXE_EXCLUSIVE})
# EXE
add_executable(${PROJECT_NAME} ${EXE_EXCLUSIVE})
target_include_directories(${PROJECT_NAME} PRIVATE include)
target_link_libraries(${PROJECT_NAME} PRIVATE ${LIB_FILES})
# GAME DLL
add_library(${DLL_NAME} SHARED ${DLL_EXCLUSIVE})
target_include_directories(${DLL_NAME} PRIVATE include)
target_link_libraries(${DLL_NAME} PRIVATE ${LIB_FILES})
# Copy DLLs
add_custom_command(TARGET ${PROJECT_NAME} POST_BUILD
  COMMAND ${CMAKE_COMMAND} -E copy_if_different
    ${DLL_FILES}
    "$<TARGET_FILE_DIR:${PROJECT_NAME}>"
)
```

Lets look at the different parts of the cmake file:

```cmake
set(DLL_NAME ${PROJECT_NAME}_game)
```

This lets us define a new variable called `DLL_NAME` and make it the same as `PROJECT_NAME` but with `_game` appended to the end of it. Now we can use this name in all places where we want to specify that we're talking about the .DLL and not the .EXE, without having to manually type out the name each time.

```cmake
# Flag .EXE specific files
set(EXE_EXCLUSIVE
  ${CMAKE_SOURCE_DIR}/src/main.cpp
  ${CMAKE_SOURCE_DIR}/src/arena.cpp
)
```

Here we store all .cpp files we want to have included with our .EXE in a single array that we've named `EXE_EXCLUSIVE` — should we need more files to be compiled into our .EXE we will have to manually modify this list.

> [!NOTE]
> Another method is to take all the .cpp files we want to have and store them in a separate subdirectory then point our functions towards that folder. But this time we'll do the manual work.

```cmake
file(GLOB_RECURSE LIB_FILES "${CMAKE_SOURCE_DIR}/lib/*.lib")
file(GLOB_RECURSE DLL_FILES "${CMAKE_SOURCE_DIR}/lib/*.dll")
file(GLOB_RECURSE DLL_EXCLUSIVE "src/*.cpp")
list(REMOVE_ITEM DLL_EXCLUSIVE ${EXE_EXCLUSIVE})
```

These are all the collections of files we will need. Using the `list()` function we can make changes to a list, in this case we use the method `REMOVE_ITEM` to strip the `EXE_EXCLUSIVE` files from the `DLL_EXCLUSIVE` files, so they have no overlap between files.

```cmake
# EXE
add_executable(${PROJECT_NAME} ${EXE_EXCLUSIVE})
target_include_directories(${PROJECT_NAME} PRIVATE include)
target_link_libraries(${PROJECT_NAME} PRIVATE ${LIB_FILES})
# GAME DLL
add_library(${DLL_NAME} SHARED ${DLL_EXCLUSIVE})
target_include_directories(${DLL_NAME} PRIVATE include)
target_link_libraries(${DLL_NAME} PRIVATE ${LIB_FILES})
```

The `add_executable` and `add_library` functions are the only part that is different between these. They specify if we should compile the following files into a .exe or a .dll. We specify the name of the file using our handy variables `PROJECT_NAME` and `DLL_NAME` then the `EXE_EXCLUSIVE` and `DLL_EXCLUSIVE` file lists are added respectively.

> [!NOTE]
> dll stands for dynamically linked library
> [!NOTE]
> Our `add_library` creates both a .dll and a .lib file as both are needed for us to work with our .dll

```cmake
# Copy DLLs
add_custom_command(TARGET ${PROJECT_NAME} POST_BUILD
  COMMAND ${CMAKE_COMMAND} -E copy_if_different
    ${DLL_FILES}
    "$<TARGET_FILE_DIR:${PROJECT_NAME}>"
)
```

Unchanged from before, this function takes all the .dll files we located in our lib folder and copies them over to our build folder.

Inside our PowerShell $profile we've added a new function:

```powershell
// $profile
function reload {
  param([Parameter(Mandatory=$true)][string]$project)
  $config = GetConfig $project
  $SourceDir = $config.path
  $BuildDir = "$SourceDir/build"
  $cachePath = "$BuildDir/CMakeCache.txt"
  $projectLine = Get-Content $cachePath | Select-String "CMAKE_PROJECT_NAME:STATIC"
  $projectName = $projectLine.Line.Split("=")[1]
  cmake --build $BuildDir --target "${projectName}_game"
}
```

1. It uses our `GetConfig` function to fetch the path to our build folder
2. Inside the build folder, it looks for a file called `CMakeCache.txt` — this is automatically added by cmake when it is being configured inside our build function `cmake --preset $preset_name`
   > [!NOTE]
   > this reload function relies on us having built our project beforehand.
3. We read the contents of our Cache looking for the line of text that includes the `CMAKE_PROJECT_NAME:STATIC` text. Because on that line, just afterwards is a `=` followed by the name of our project set by our cmakelists.txt. We then store the name of our game in the `$projectName` variable by splitting the line we found at the position of the first (and only) `=` then takes the second text.
   > [!NOTE]
   > arrays start at 0, so index 1 is actually the second entry
4. We then tell cmake to build the project again, but only the target named `NameOfOurProject_game`. So if our game is called "Asteroids" in our cmakelists.txt then `$projectName` will be "Asteroids_game" — this is the name of our .dll, ensuring that that is the only thing being compiled.

So if we follow the syntax of having `_game` as a suffix for our created .dll for all projects then this will work just fine. If we ever need more flexibility with our naming scheme we can store info like this name in for example our `projects.json`.

Now we've configured our CmakeLists.txt and added a powershell function to help us later. The next step is to begin learning about memory management and how to set up a memory arena as well as learning more about pointers. We'll be returning to our practice example.cpp that we added a long time ago to try small projects. This is a single file script that shows how we can work with what is called a memory arena and we'll look at how we can think about memory and pointers pointing to memory.

```cpp
// practice project
// part 1
#include <iostream>
#include <windows.h>
using namespace std;
struct MemoryArena {
  unsigned char* base;
  size_t size;
  size_t used;
};
void* Arena_Add(MemoryArena* arena, size_t size){
  if(arena->used + size > arena->size){
    return nullptr; // Safety so we can't go beyond our arena size
  }
  void* latest_point = arena->base + arena->used;
  arena->used += size;
  return latest_point;
}
void Arena_Initialize(MemoryArena* arena, void* start, size_t size){
  arena->base = (unsigned char*)start;
  arena->size = size;
  arena->used = 0;
}
void Arena_Reset(MemoryArena* arena){
  arena->used = 0;
}
struct Character {
  enum CHARACTER_TYPE {HERO, ENEMY};
  CHARACTER_TYPE my_character_type;
  int health;
  int damage;
  bool is_alive;
  char* name;
};
```

```cpp
// practice project
// part 2
struct Game_Data {
  Character* characters;
  int character_count;
  int score;
};
Game_Data* gameData;
MemoryArena arena;
int main(){
  size_t memory_size = (1024 * 1024 * 4); // 4mb
  void* blob_of_memory = malloc(memory_size);
  if(blob_of_memory == nullptr){
    return 1;
  }
  Arena_Initialize(&arena, blob_of_memory, memory_size);
  gameData = (Game_Data*)Arena_Add(&arena, sizeof(Game_Data));
  gameData->character_count = 10;
  gameData->score = 0;
  void* characters_start_point = Arena_Add(&arena, sizeof(Character) * gameData->character_count);
  gameData->characters = (Character*)characters_start_point;
  gameData->characters[3].health = 32;
  cout << gameData->characters[3].health << endl;
  Sleep(2000);
  return 0;
}
```

In this program we:

1. Create a few structs: `MemoryArena`, `GameData`, `Character`
   Then we use these structs to hold variables, and our `GameData` struct holds `Character` structs itself. But notice how the variable name is `characters` in plural, but we only store a single pointer — this should mean that we are only storing a single character. We are actually storing a collection of characters inside memory by pointing to the first character only. We'll get back to how that is set up once we have a better understanding of the program in its entirety.

2. We create our `MemoryArena` struct
   > [!NOTE]
   > lets conceptualize a memory arena as a continuous block of memory, each thing laid out next to the previous.

This struct holds very little in terms of stuff, but is very powerful. Our three variables are:

- `unsigned char* base;` — This a pointer to the first address in memory of our arena. We need to use an `unsigned char*` instead of a `void*` as this allows us to add a number to our base to move further down our block of memory. If we had our pointer as a `void*` it would need to be cast all the time before we attempt to do arithmetic (+, -, x, etc).
  > [!NOTE]
  > we will learn about casting a bit later in this lecture
- `size_t size;` — `size_t` is a type of variable, like an int, that holds a whole number, but `size_t` is larger than an `int` and especially made to help us store how big something is. `size_t` is also unsigned, meaning that compared to a int it can't store a negative number. This variable is meant to tell us how large our memory block is, whilst the `unsigned char*` pointer above just tells us where it starts.
- `size_t used;` — We update this variable each time we specify what the next piece of memory is used for. So we know we aren't overwriting other parts of our memory when adding new things to it. Also, by resetting this to 0 we actually delete all memory in the arena all at once. We do this in the `Arena_Reset()` function

3. We've created three functions:
   - `Arena_Add()` — This function tells our memory arena to tag a portion of memory as used.
   - `Arena_Initialize()` — This function sets up the memory arena by setting up the values of our struct.
   - `Arena_Reset()` — This function sets the size of our `size_t used` variable to 0, meaning that to the memory arena no part of memory is tagged and should something new be added into memory then it will write it at the start of the memory block.

But before we can use this memory arena we need to find a place in memory where we can store our continuous chunk. In other applications where we create and free memory willy-nilly our memory lives all over our heap — in this program we will store all our memory in one location and once we're done with it we will free it all at once. The upside to this is:

1. We know that our program will not crash due to insufficient memory — if it starts up then we know we managed to allocate enough memory.
2. Our memory lives in tidy blocks on our heap, making accessing them faster as the CPU doesn't have to go back to RAM as often to fetch a chunk of memory. This is perhaps the biggest performance win between our data-oriented approach and the object-oriented approach found in game engines like Unity.

```cpp
size_t memory_size = (1024 * 1024 * 4); // 4mb
void* blob_of_memory = malloc(memory_size);
```

This code, that runs as the very first thing we do in our programs entry point, finds a chunk of unused data that is 4mb large and sets it aside for us. It's just filled with junk data right now — but it is ours and no other program or background operation will use it.

Later, this blob of memory will be owned by our .EXE and passed to our .DLL each tick, allowing our game to work with the memory. But because the memory is owned by our .EXE, if we ever recompile the .DLL nothing will happen to our data, allowing our changed functions to immediately start using the same memory without us needing to restart our game.

The `void*` variable is a pointer to the first bit in memory at the start of our memory chunk. Used alongside our `memory_size` we know the start of our memory chunk and how large it is.

The `malloc` function sets aside a specified size in memory then returns a pointer to the first bit. 1kb (kilobyte) is 1024 bytes, a mb is 1024 of those. Then multiplying that by 4 we get our total of 4mb. A small footprint for our program, but in this example we're using almost none of it at all.

```cpp
if(blob_of_memory == nullptr){
  cout << "failed to allocate memory" << endl;
  return 1;
}
```

This if-statement checks if our pointer returned from `malloc` actually successfully managed to find a place in memory to point to. If not we enter the scope of the if-statement and our `main()` returns with an error code of 1 after printing an error. From the Microsoft C++ documentation: "malloc returns a void pointer to the allocated space, or NULL if there's insufficient memory available". NULL is from C, a language that C++ builds off of. The modern C++ style wants us to use `nullptr` instead. It is safer as NULL is just another way of saying 0, whilst `nullptr` is not a number but the result of a non-viable operation. Meaning that if we accidentally had a NULL in our code and we tried to use it in arithmetic we would be allowed to. But a `nullptr` will throw an error, which is better, because we want to stop such undefined behaviour.

```cpp
Arena_Initialize(&arena, blob_of_memory, memory_size);
```

Here we take our `blob_of_memory`, AKA the pointer to the first byte in the memory chunk, along with the size of the memory chunk and the arena and pass all of these to our `Arena_Initialize()` function. It is then responsible for setting up the `MemoryArena` by assigning these values to the relevant variables (`base` and `size`) as well as setting `used` to 0, meaning that nothing has been declared inside this chunk.

> [!NOTE]
> this has just set some initial state for our arena, it is yet to have anything actually useful inside of it

```cpp
gameData = (Game_Data*)Add_To_Arena(&arena, sizeof(Game_Data));
gameData->character_count = 10;
gameData->score = 0;
```

Here we must take a moment to learn about **casting** — casting is the process of telling our compiler that we want to take data that is one type, and treat it as if it were another type. Not all types can be cast to each other, but the `void*` returned from our `Add_To_Arena` function can be cast into any other pointer. Just as for example a `float` value can be cast into an `int`, but of course in the process it gets stripped of its decimal value:

```cpp
float decimalValue = 2.5;
int cast_to_int = (int)decimalValue;
printf(decimalValue) // this prints 2.5
printf(cast_to_int) // this prints '2'
```

So what we're doing is allocating inside the arena enough memory to store the `Game_Data` struct, meaning that our memory holds one pointer to a character, and two ints. We store this space in memory using our `gameData` pointer. Now part of our memory arena has been allocated, this increases our `used` variable by that amount. So any new data allocated to the arena will start from this adjusted position (`base + used`) instead of at just `base`.

Now we need to allocate some memory for our collection of characters:

```cpp
size_t character_memory_footprint = sizeof(Character) * gameData->character_count;
void* characters_start_point = Add_To_Arena(&arena, character_memory_footprint);
gameData->characters = (Character*)characters_start_point;
```

A single instance of our `Character` struct takes up a certain amount of memory. The `sizeof()` function calculates this for us. We have decided to have 10 characters allocated into memory. This is stored in our `character_count` variable. We can therefore multiply the size of one `Character` by that amount to get the total amount of memory that 10 Characters will need. We then use our `Add_To_Arena` function to tag that amount of memory as being used to store our 10 Characters. The function returns a `void*` pointing to the first byte of our character memory chunk. Lastly we cast this `void*` into a `Character*` so that the compiler knows what type of memory we have stored.

Now inside our memory arena we have laid out the memory for 10 characters sequentially. And because we know the size of a `Character` and we know they are packed next to each other in memory we can use the array `[]` symbols to fetch one of the ten characters by specifying its position in the memory 0-9.

> [!NOTE]
> remember that arrays start at 0 instead of at 1. This means that element 9 is the 10th and last element.

```cpp
gameData->characters[3].health = 32;
cout << gameData->characters[3].health << endl;
```

Here we find the memory address of the 4th character and set the value of the health variable stored at this point in memory to 32. Then just to make sure we've successfully set everything up we print the value stored at that point to our console using `cout << value << endl;`. So after we've assigned our `gameData` to our memory arena we can stop thinking about the arena and just work with the pointer as normal.

Now that we know more about how a memory arena, casting and memory layouts work we are ready to bring this into our SDL3 project to add new files and set up necessary boilerplate to allow us to work with our .EXE and .DLL solution.