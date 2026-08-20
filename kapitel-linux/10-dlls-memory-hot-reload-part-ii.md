> **Linux:** This chapter is adapted for Linux.

# DLLs Memory and Hot Reloading - Part II

It's time to head back to our SDL3 project to set up the boilerplate necessary to use our executable + shared library (.so) system.

In our previous example we had everything in one placeholder practice example.cpp. Now we will start breaking things into separate .cpp files along with corresponding .h files.

At the end of this lecture we will have the following files in our src folder:

- `arena.cpp` — holds the implementation of functions from arena.h
- `arena.h` — holds the declaration of functions for our memory arena as well as the arena struct
- `common.h` — A helper .h file containing some helpful macros to figure out memory sizes in kb, mb and gb
- `game.cpp` — Acts as the "entry point" for the shared library and performs our Input, Update, Draw routines
- `game.h` — Holds the definitions for the functions used in game.cpp and has them tagged in such a way that we can find them from our main executable
- `gameState.h` — a .h file containing the struct with all variables used inside the game
- `main.cpp` — our executable entry point, initializes everything, sets up memory and the game loop. Calls into our shared library through functions found in game.h

The process of breaking out parts of code into its own files is industry standard, as it allows clearer boundaries between files and makes reasoning about them simpler.

As our executable has no access to game.cpp directly we need to specify where it can find each of its relevant functions:

- `Initialize`
- `HandleEvents`
- `Update`
- `Draw`

This must be done in a few steps:

1. Flag the functions inside game.h in such a way as to be usable in this way
2. For each function, create a 'function pointer'
3. Create a struct to hold these 'function pointers' in one location
4. Connect each 'function pointer' to the right function in the shared library
5. Using the struct, call each function where appropriate

Once we have all the necessary boilerplate set up, we can actually ignore most of it, working instead as we normally would. So it's a bit of upfront costs for a lot of benefit later on.

The `arena.h` and `arena.cpp` files are very similar to our practice example, but lifted into their own files:

```cpp
// arena.h
#pragma once
namespace Memory {
  struct Arena {
    unsigned char* base;
    size_t size;
    size_t used;
  };
  void Initialize(Arena* arena, void* memory, size_t size);
  void* Allocate(Arena* arena, size_t size);
  void Reset(Arena* arena);
}
```

At the top of this .h file we write `#pragma once` — we will be doing this for ALL .h files we ever write. This is a not-so-nice feature of C++ where without it our .h file will be copied above all files that implement it, meaning that our executable or shared library bloats unnecessarily. By adding `#pragma once` our compiler knows to only add these once, which is enough.

We have encapsulated our struct and function declarations in a namespace we've named `Memory`. This means that when we `#include "arena.h"` we can only access the struct and functions by first specifying the namespace — for example: `Memory::Initialize()`

The struct `Arena` holds the same three variables as our practice example and the three functions are the same as well. We first call `Initialize` to make sure our `size` variable is set, our `used` is zeroed and our `base` pointer points at the first byte in memory.

```cpp
// arena.cpp
#include "arena.h"
#include <cstring>
void Memory::Initialize(Arena* arena, void* mem_start, size_t size) {
  arena->base = (unsigned char*)mem_start;
  arena->size = size;
  arena->used = 0;
}
void* Memory::Allocate(Arena *arena, size_t size) {
  void* front = arena->base + arena->used;
  arena->used += size;
  memset(front, 0, size);
  return front;
}
void Memory::Reset(Arena *arena){
  arena->used = 0;
}
```

We include `arena.h` at the top so we can create the functions declared in the .h file. But to work with those functions we need to remember to specify their namespace, otherwise we aren't connecting our functions to those inside the .h file, but instead creating new functions with the same names.

Like in our practice example we are using the same functions, but have opted for the more appropriately named `Allocate` rather than `Add_To_Arena`. We've also added a new line of code to our `Allocate`:

```cpp
memset(front, 0, size);
```

This code makes sure that the block of memory that we've allocated here is free from garbage data by putting the value 0 across the board. This defensive pattern is called **zero-allocation** — it makes all numbers 0 and all pointers become `nullptr`.

We add `#include <cstring>` as that is the header that holds `memset()`.

That's it for our memory arena. Mostly it's all the same stuff, just in its own two files and with our defensive pattern added.

Lets look at our `common.h`:

```cpp
// common.h
#pragma once
#define KILOBYTES(n) ((size_t)n * 1024)
#define MEGABYTES(n) (KILOBYTES(n) * 1024)
#define GIGABYTES(n) (MEGABYTES(n) * 1024)
constexpr size_t GAME_MEMORY_ALLOWANCE = MEGABYTES(10);
```

Once again we have our `#pragma once` at the top. What follows are three macros that simplify getting the correct `size_t` for different sizes of memory.

We use this header file in our `main.cpp` to simplify our `malloc` (memory allocation) and our minimal `gameState.h`:

```cpp
// gamestate.h
#pragma once
#include <SDL3/SDL.h>
struct GameData {
  SDL_FRect rect;
  float move_speed;
};
```

> **Linux:** On Linux, include `<SDL3/SDL.h>` instead of individual SDL headers.

Inside our `GameData` struct we currently just specify two variables, our rectangle and how fast we are going to want it to move.

Ok, we've looked at four out of seven files, but those were the short and simple ones. `game.cpp` and `game.h` have some new logic but 90% of our boilerplate code lives inside our `main.cpp`. Lets tackle `game.h` and `game.cpp` next:

```cpp
// game.h
#pragma once
#include <SDL3/SDL.h>
#include "gameState.h"
extern "C" {
  void Initialize(GameData* data);
  bool HandleEvents(GameData* data, SDL_Event event);
  void Draw(GameData* data, SDL_Renderer* renderer);
  void Update(GameData* data, float dt);
  void OnQuit(SDL_Renderer* renderer);
}
```

> **Linux:** On Linux, we use `__attribute__((visibility("default")))` instead of `__declspec(dllexport)`. For simplicity here, we just mark them `extern "C"` and use compiler flags to control visibility. The `__declspec` is Windows-specific and not needed on Linux.

`extern "C"` ensures that during compilation, our functions won't have their names modified in any way. This is essential when we want to access them later by referencing their names exactly, otherwise the name will be changed in a linking process called **name-mangling**. (https://en.wikipedia.org/wiki/Name_mangling)

```cpp
// game.cpp
#include "game.h"
extern "C" {
  void Initialize(GameData* data){
    data->rect.x = 100;
    data->rect.y = 100;
    data->rect.h = 50;
    data->rect.w = 50;
    data->move_speed = 100;
  }
  bool HandleEvents(GameData *data, SDL_Event event){
    if(event.type != SDL_EVENT_KEY_DOWN){
      return true;
    }
    if(event.key.key == SDLK_ESCAPE){
      return false;
    }
    return true;
  }
  void Update(GameData* data,float dt){
    const bool* keys = SDL_GetKeyboardState(NULL);
    if(keys[SDL_SCANCODE_RIGHT]){
      data->rect.x += data->move_speed * dt;
    }
    if(keys[SDL_SCANCODE_LEFT]){
      data->rect.x -= data->move_speed * dt;
    }
    if(keys[SDL_SCANCODE_UP]){
      data->rect.y -= data->move_speed * dt;
    }
    if(keys[SDL_SCANCODE_DOWN]){
      data->rect.y += data->move_speed * dt;
    }
  }
  void Draw(GameData* data, SDL_Renderer* renderer){
    SDL_SetRenderDrawColor(renderer, 250, 70, 8, 255);
    SDL_RenderClear(renderer);
    SDL_SetRenderDrawColor(renderer, 150, 0, 100, 255);
    SDL_RenderFillRect(renderer,&data->rect);
    SDL_RenderPresent(renderer);
  }
  void OnQuit(SDL_Renderer* renderer){
    SDL_DestroyRenderer(renderer);
  }
}
```

Our `game.cpp` is similar to last time. We wrap each function inside the `extern "C"` scope to mirror the change in the .h file, which is necessary to ensure that both declaration and implementation avoid having the function names be name-mangled.

Lets look at the `Update()` function again:

```cpp
// game.cpp
void Update(GameData* data,float dt){
  const bool* keys = SDL_GetKeyboardState(NULL);
  if(keys[SDL_SCANCODE_RIGHT]){
    data->rect.x += data->move_speed * dt;
  }
  if(keys[SDL_SCANCODE_LEFT]){
    data->rect.x -= data->move_speed * dt;
  }
  if(keys[SDL_SCANCODE_UP]){
    data->rect.y -= data->move_speed * dt;
  }
  if(keys[SDL_SCANCODE_DOWN]){
    data->rect.y += data->move_speed * dt;
  }
```

Now that we know more about pointers we can see that our `keys` variable holds a pointer to the first byte of the place in memory where all keys (bool's) are laid out sequentially. The `const` keyword means that the variables we find at the memory being pointed to can not be changed by us accidentally in code.

> **NOTE:** we would need to store the result of the keys from the previous tick if we want to know if a key has been released or just pressed down this tick. But that is for a later lecture

Enough stalling, lets start digging into our new `main.cpp`:

```cpp
// main.cpp
#include <dlfcn.h>
#include <cstdio>
#include <sys/stat.h>
#include <SDL3/SDL.h>
#include "common.h"
#include "arena.h"
#include "gameState.h"
```

> **Linux:** On Linux we use `<dlfcn.h>` instead of `<windows.h>`. This gives us `dlopen`, `dlsym`, `dlclose` instead of `LoadLibrary`, `GetProcAddress`, `FreeLibrary`. We also use `<sys/stat.h>` for file timestamps.

We're starting to include quite a lot of headers. We'll stick to best practices when it comes to ordering our .h files:

1. Start with the standard library headers (that are passed by using `<>`)
2. After those we include all headers we have not written ourselves. In this case the SDL3 headers.
3. After that we add our custom made headers

```cpp
// main.cpp
SDL_Window* window;
SDL_Renderer* renderer;
```

We will be passing the pointer to the renderer to our shared library.

```cpp
// main.cpp
Uint64 NOW = 0;
Uint64 PREV = 0;
```

Unsigned integers to store the time between ticks so we can calculate our delta time.

```cpp
// main.cpp
constexpr const char* NAME_OF_LIB = "libHeartburner_game.so";
constexpr const char* NAME_OF_TEMP_LIB = "libHeartburner_temp.so";
```

> **Linux:** Shared libraries on Linux use the `.so` extension and typically follow the `lib` prefix convention.

A `char*` is a pointer that points to the first byte of a series of individual characters. The `constexpr` keyword makes the variable a **compile-time constant**.

```cpp
// main.cpp
typedef void (*Function_Initialize) (GameData* data);
typedef bool (*Function_HandleEvents) (GameData* data, SDL_Event event);
typedef void (*Function_Update) (GameData* data, float dt);
typedef void (*Function_Draw) (GameData* data, SDL_Renderer* renderer);
typedef void (*Function_OnQuit) (SDL_Renderer* renderer);
```

We need a way of accessing the functions we've set as exported in our `game.h`. To do this we need to create what is called **function pointers** — a function pointer is a way of passing a function as a variable.

`typedef` allows us to take the structure of our specific functions, meaning their return type and parameters and allows us to store them with a more easily typed name, like `Function_Initialize`.

```cpp
// main.cpp
constexpr const char* NAME_OF_FUNC_INIT = "Initialize";
constexpr const char* NAME_OF_FUNC_HANDLE_EVENT = "HandleEvents";
constexpr const char* NAME_OF_FUNC_UPDATE = "Update";
constexpr const char* NAME_OF_FUNC_DRAW = "Draw";
constexpr const char* NAME_OF_FUNC_QUIT = "OnQuit";
```

```cpp
// main.cpp
struct LIB_INFO{
  void* handle;
  time_t timestamp;
  Function_Initialize initialize;
  Function_HandleEvents handleEvents;
  Function_Update update;
  Function_Draw draw;
  Function_OnQuit quit;
};
```

> **Linux:** We use `void*` as the handle type (from `dlopen`) instead of Windows `HMODULE`. For timestamps we use `time_t` from `<sys/stat.h>`.

We create a struct that holds all the data we will need to work with our shared library.

```cpp
// main.cpp
time_t GetTimestamp(){
  struct stat attr;
  stat(NAME_OF_LIB, &attr);
  return attr.st_mtime;
}
```

> **Linux:** We use `stat()` and `st_mtime` to get the last modification time of the file. This is the Linux equivalent of `FindFirstFile` + `ftLastWriteTime`.

```cpp
// main.cpp
bool LoadLib(LIB_INFO* info, int depth = 0){
  printf("loading lib");
  if(depth > 20){
    printf("failed to write temp lib");
    return false;
  }
  // Copy the file using shell command
  char cmd[256];
  snprintf(cmd, sizeof(cmd), "cp %s %s", NAME_OF_LIB, NAME_OF_TEMP_LIB);
  int success = system(cmd);
  if(success != 0){
    usleep(50000);
    return LoadLib(info, depth + 1);
  }
  info->handle = dlopen(NAME_OF_TEMP_LIB, RTLD_NOW);
  if(info->handle == nullptr){
    printf("could not load lib: %s", dlerror());
    return false;
  }
  info->initialize = (Function_Initialize)dlsym(info->handle, NAME_OF_FUNC_INIT);
  info->handleEvents = (Function_HandleEvents)dlsym(info->handle, NAME_OF_FUNC_HANDLE_EVENT);
  info->update = (Function_Update)dlsym(info->handle, NAME_OF_FUNC_UPDATE);
  info->draw = (Function_Draw)dlsym(info->handle, NAME_OF_FUNC_DRAW);
  info->quit = (Function_OnQuit)dlsym(info->handle, NAME_OF_FUNC_QUIT);
  info->timestamp = GetTimestamp();
  return true;
}
```

> **Linux:** We use `dlopen` instead of `LoadLibrary`, `dlsym` instead of `GetProcAddress`, and `cp` instead of `CopyFile`. The `RTLD_NOW` flag loads all symbols immediately.

This function has the job of finding our compiled .so, create a copy of it called `NAME_OF_TEMP_LIB` then store that .so in our provided `LIB_INFO` struct. We then take the function pointers in our `LIB_INFO` struct and point them to the functions we exported.

The function uses a **recursive function call** — meaning that the function has called itself. In case the copy fails, we wait for 50 milliseconds then we return the result of calling the function again, incrementing the `depth` parameter by 1. We only allow the function to call itself a total of 20 times before stopping.

```cpp
// main.cpp
void UnloadLib(LIB_INFO* info){
  dlclose(info->handle);
  info->handle = nullptr;
  remove(NAME_OF_TEMP_LIB);
}
```

> **Linux:** We use `dlclose` instead of `FreeLibrary` and `remove` instead of `DeleteFile`.

```cpp
// main.cpp
void* AllocateGameMemory(){
  void* blob = malloc(GAME_MEMORY_ALLOWANCE);
  if(blob == nullptr){
    printf("fatal error: could not allocate memory");
    return nullptr;
  }
  printf("memory succesfully allocated");
  return blob;
}
```

```cpp
// main.cpp
void SDL_Setup(){
  SDL_Init(SDL_INIT_EVENTS);
  window = SDL_CreateWindow("pilot", 650, 400, 0);
  renderer = SDL_CreateRenderer(window, NULL);
}
```

```cpp
// main.cpp
void CalculateDeltaTime(float& dt){
  NOW = SDL_GetTicksNS();
  dt = NOW - PREV;
  dt = SDL_NS_TO_SECONDS(dt);
  PREV = NOW;
}
```

This function does the exact same thing, taking a pointer to a float and setting it to the calculated delta time. Here we've used the `float&` rather than `float*` — this is what we called **passing by reference** rather than passing by pointer.

```cpp
// main.cpp
void Lib_CheckStatus(LIB_INFO* lib){
  time_t timestamp = GetTimestamp();
  bool is_timestamp_changed = timestamp != lib->timestamp;
  if(is_timestamp_changed){
    UnloadLib(lib);
    LoadLib(lib);
  }
}
```

> **Linux:** We compare `time_t` values directly instead of using `CompareFileTime`.

```cpp
// main.cpp
int main() {
  void* game_memory = AllocateGameMemory();
  if(game_memory == nullptr){
    return 1;
  }
  Memory::Arena* arena = new Memory::Arena();
  Memory::Initialize(arena, game_memory, GAME_MEMORY_ALLOWANCE);
  GameData* gameData = (GameData*)Memory::Allocate(arena, sizeof(GameData));
  LIB_INFO lib;
  bool lib_successfully_loaded = LoadLib(&lib);
  if(lib_successfully_loaded == false){
    return 2;
  }
  SDL_Setup();
  lib.initialize(gameData);
  bool running = true;
  float dt;
  while(running){
    Lib_CheckStatus(&lib);
    CalculateDeltaTime(&dt);
    SDL_Event event;
    while(SDL_PollEvent(&event)){
      running = lib.handleEvents(gameData, event);
      if(running == false){
        break;
      }
    }
    lib.update(gameData, dt);
    lib.draw(gameData, renderer);
  }
  lib.quit(renderer);
  SDL_Quit();
  return 0;
}
```

Here we do the same steps as in our previous lecture, but our `initialize()`, `update()` and `draw()` functions are all called from our `LIB_INFO` struct using the function pointers we set in `LoadLib()`.

As a summary, this is what our `main()` does:

1. We allocate a blob of memory
2. We create a memory arena and point it to the start of our memory blob
3. We load our shared library and set up our function pointers
4. We set up SDL, creating our window and assigning our renderer
5. We begin our core loop
6. Inside our core loop we compare timestamps for our shared library
7. Then we calculate delta time
8. We collect all SDL_Events and pass them to the `HandleEvent` function in our shared library
9. We then call `update` followed by `draw` in our shared library
10. And if our core loop ever terminates we do some cleanup and return 0

We are done! We have now successfully added the boilerplate code necessary to work with our executable and shared library setup, allowing us to start developing a game in our next lecture.

We can now do the following steps:

1. Build the game using `cmake --build build`
2. Run the game
3. As the game is running, change the colors of our `SDL_SetRenderDrawColor()`
4. Compile our shared library using `cmake --build build --target Heartburner_game`
5. Back in our still running program we can see that the changes we made to our `game.cpp` is directly visible in the game, without us having to close the game and running it again.

This ability to make live-changes to our game is such a huge win for us and will make development of any game so much more streamlined!
