# DLLs Memory and Hot Reloading - Part II

It's time to head back to our SDL3 project to set up the boilerplate necessary to use our .EXE + .DLL system.

In our previous example we had everything in one placeholder practice example.cpp. Now we will start breaking things into separate .cpp files along with corresponding .h files.

At the end of this lecture we will have the following files in our src folder:

- `arena.cpp` — holds the implementation of functions from arena.h
- `arena.h` — holds the declaration of functions for our memory arena as well as the arena struct
- `common.h` — A helper .h file containing some helpful macros to figure out memory sizes in kb, mb and gb
- `game.cpp` — Acts as the "entry point" for the DLL and performs our Input, Update, Draw routines
- `game.h` — Holds the definitions for the functions used in game.cpp and has them tagged in such a way that we can find them from our main.exe
- `gameState.h` — a .h file containing the struct with all variables used inside the game
- `main.cpp` — our .exe entry point, initializes everything, sets up memory and the game loop. Calls into our DLL through functions found in game.h

The process of breaking out parts of code into its own files is industry standard, as it allows clearer boundaries between files and makes reasoning about them simpler.

As our .EXE has no access to game.cpp directly we need to specify where it can find each of its relevant functions:

- `Initialize`
- `HandleEvents`
- `Update`
- `Draw`

This must be done in a few steps:

1. Flag the functions inside game.h in such a way as to be usable in this way
2. For each function, create a 'function pointer'
3. Create a struct to hold these 'function pointers' in one location
4. Connect each 'function pointer' to the right function in the DLL
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

At the top of this .h file we write `#pragma once` — we will be doing this for ALL .h files we ever write. This is a not-so-nice feature of C++ where without it our .h file will be copied above all files that implement it, meaning that our .DLL or .EXE bloats unnecessarily. By adding `#pragma once` our compiler knows to only add these once, which is enough.

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

This code makes sure that the block of memory that we've allocated here is free from garbage data by putting the value 0 across the board. This will stop undefined behaviour when — if we've not been careful — we try and access data before we've set its values. This defensive pattern is called **zero-allocation** — it makes all numbers 0 and all pointers become `nullptr`. Without this code we could find ourselves working with data that has been given random values.

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

Once again we have our `#pragma once` at the top. What follows are three macros that simplify getting the correct `size_t` for different sizes of memory. Making it easier to remember that `1024 * 1024 * 4` means we're working with 4mb. Now we can write `MEGABYTES(4)` and it is compiled into the same thing. What `define` actually does is specify a preprocessor instruction that substitutes the text on the left in our project with that on the right. Meaning that for the compiler it was always the full text, but we, the programmers, were allowed a bit of a smoother experience, having to type less each time we want to use the specific piece of code.

Lastly we have set up a variable that can not change at runtime (because of the `constexpr`) — this, whenever used, will be substituted by 10 megabytes. This part is not strictly necessary, but if we imagine our `common.h` to eventually hold a bunch of settings for our project. Then keeping the memory budget here starts to look more reasonable. We will be adding `constexpr` to values which have the 2 following conditions:

1. The value is not meant to change
2. We know the value at compile time

We only use this header file in our `main.cpp` currently to simplify our `malloc` (memory allocation) and our minimal `gameState.h`:

```cpp
// gamestate.h
#pragma once
#include "SDL3/SDL_rect.h"
struct GameData {
  SDL_FRect rect;
  float move_speed;
};
```

And just to hammer it home, we have `#pragma once` at the very top of our .h file. Then we include `SDL3/SDL_rect.h` so we can use the `SDL_FRect` struct.

Inside our `GameData` struct we currently just specify two variables, our rectangle and how fast we are going to want it to move. Note how we don't do any setup or assign any value to these variables — that will be done in our actual game logic.

Ok, we've looked at four out of seven files, but those were the short and simple ones. `game.cpp` and `game.h` have some new logic but 90% of our boilerplate code lives inside our `main.cpp`. Lets tackle `game.h` and `game.cpp` next:

```cpp
// game.h
#pragma once
#include "SDL3/SDL_render.h"
#include "gameState.h"
extern "C" {
  __declspec(dllexport) void Initialize(GameData* data);
  __declspec(dllexport) bool HandleEvents(GameData* data, SDL_Event event);
  __declspec(dllexport) void Draw(GameData* data, SDL_Renderer* renderer);
  __declspec(dllexport) void Update(GameData* data, float dt);
  __declspec(dllexport) void OnQuit(SDL_Renderer* renderer);
}
```

Note the `#pragma once`.

If we look past the very strange looking `extern "C"` and the `__declspec(dllexport)` prefix attached to each function declaration then this .h file looks very standard. Here it is just for posterity:

```cpp
// just an example without __declspec
#pragma once
#include "SDL3/SDL_render.h"
#include "gameState.h"
void Initialize(GameData* data);
bool HandleEvents(GameData* data, SDL_Event event);
void Draw(GameData* data, SDL_Renderer* renderer);
void Update(GameData* data, float dt);
void OnQuit(SDL_Renderer* renderer);
```

`extern "C"` ensures that during compilation, our functions won't have their names modified in any way. This is essential when we want to access them later by referencing their names exactly, otherwise the name will be changed in a linking process called [**name-mangling**](https://en.wikipedia.org/wiki/Name_mangling).

`__declspec(dllexport)` has that very strange syntax as it is not part of normal C++ but an addition by Microsoft to flag the function as relevant to the compiler, in this case making sure the function is made available to other programs that are interested in calling it. The syntax is very strange, but fortunately we will basically only use it in this specific case. Meaning that as long as we can remember that the header file required some strange additions then we can search for those again later if we forget the exact syntax — that's very common.

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

Our `game.cpp` is similar to last time. The changes are we work directly on the `SDL_FRect` struct instead of our own `xPos` and `yPos` variables that we later assigned to the `FRect`. We also wrap each function inside the `extern "C"` scope to mirror the change in the .h file, which is necessary to ensure that both declaration and implementation avoid having the function names be name-mangled and no longer have the same name as each other to the linker. If the names were to differ to the linker then we would not be able to call them by `#include` the header.

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

Now that we know more about pointers we can see that our `keys` variable holds a pointer to the first byte of the place in memory where all keys (bool's) are laid out sequentially. The `const` keyword means that the variables we find at the memory being pointed to can not be changed by us accidentally in code. It has become read-only — meaning that we can read the value but not write to it.

Then we can use the array syntax `[]` along with an enum to handle the pointer offset to fetch the memory address of the specific keyboard key we are interested in. The enum `SDL_SCANCODE_XXX` is built into SDL and is made for this exact purpose.

Meaning that our if-statement is true if the specific key is being held down this tick.

> [!NOTE]
> we would need to store the result of the keys from the previous tick if we want to know if a key has been released or just pressed down this tick. But that is for a later lecture

Enough stalling, lets start digging into our new `main.cpp`:

```cpp
// main.cpp
#include <windows.h>
#include <fileapi.h>
#include <cstdio>
#include "SDL3/SDL_init.h"
#include "SDL3/SDL_render.h"
#include "SDL3/SDL_timer.h"
#include "common.h"
#include "arena.h"
#include "gameState.h"
```

We're starting to include quite a lot of headers. And some of these headers will eventually start to clash with each other. You'll no doubt start getting messy error messages as your program grows. We will be looking into more robust solutions later, but for now we can resolve this inclusion chain by sticking to best practices when it comes to ordering our .h files:

1. Start with the global headers from Windows and the standard library (that are passed by using `<>`)
2. After those we include all headers we have not written ourselves. In this case the SDL3 headers.
3. After that we add our custom made headers

```cpp
// main.cpp
SDL_Window* window;
SDL_Renderer* renderer;
```

We will be passing the pointer to the renderer to our DLL, we also need to have an `SDL_Window` to assign the renderer to.

```cpp
// main.cpp
Uint64 NOW = 0;
Uint64 PREV = 0;
```

Unsigned integers to store the time between ticks so we can calculate our delta time.

```cpp
// main.cpp
constexpr const char* NAME_OF_DLL = "Heartburner_game.dll";
constexpr const char* NAME_OF_TEMP_DLL = "Heartburner_temp.dll";
```

A `char*` is a pointer that points to the first byte of a series of individual characters. The block in memory that holds the full character-sequence is automatically followed by a terminator, meaning that just by pointing at the first byte, then reading until we hit the terminator we can get a hold of the full character sequence.

> [!NOTE]
> another data type that stores text is a `string` but many of the functions we are supplying `NAME_OF_DLL` to only accepts a `char*` and we would have to convert our string in order to pass it as a parameter.

A `const char*` means that the characters stored at the memory address are not allowed to change. In other words, the text itself is read-only. However, without additional qualifiers, the pointer itself could still be reassigned to point somewhere else. That means the variable could later be made to reference a different string.

The `constexpr` keyword makes the variable a **compile-time constant**. This implies that the pointer itself cannot change after initialization, and the compiler knows its value during compilation.

Now if we try and write:

```cpp
NAME_OF_DLL = "a_new_name";
```

We get this error:

```
Cannot assign to variable `NAME_OF_DLL` with const-qualified type 'const char *const'
```

If we never make any mistakes when coding, then all of these qualifiers are unnecessary. But we are adding these to:

1. Safeguard against screwing things up
2. Write hireable C++
3. Learn about `const` and `constexpr`
4. Help other programmers know the intention of our program

```cpp
// main.cpp
typedef void (*Function_Initialize) (GameData* data);
typedef bool (*Function_HandleEvents) (GameData* data, SDL_Event event);
typedef void (*Function_Update) (GameData* data, float dt);
typedef void (*Function_Draw) (GameData* data, SDL_Renderer* renderer);
typedef void (*Function_OnQuit) (SDL_Renderer* renderer);
```

We need a way of accessing the functions we've set as `__declspec(dllexport)` in our `game.h`. To do this we need to create what is called **function pointers** — a function pointer is a way of passing a function as a variable. Meaning that we can store the address of a function and call it later.

These five function pointers have the same exact return type and parameters as the functions inside `game.h`.

`typedef` allows us to take the structure of our specific functions, meaning their return type and parameters and allows us to store them with a more easily typed name, like `Function_Initialize`. Without our `typedef` we would need to write:

```cpp
void (*initFunc)(GameData* data) = MyInitFunction;
```

With our top-level `typedef` we can just use the name we've provided as a substitute for all the behind-the-scenes stuff:

```cpp
Function_Initialize initFunc = MyInitFunction
```

```cpp
// main.cpp
constexpr const char* NAME_OF_FUNC_INIT = "Initialize";
constexpr const char* NAME_OF_FUNC_HANDLE_EVENT = "HandleEvents";
constexpr const char* NAME_OF_FUNC_UPDATE = "Update";
constexpr const char* NAME_OF_FUNC_DRAW = "Draw";
constexpr const char* NAME_OF_FUNC_QUIT = "OnQuit";
```

This series of `char*` stores the exact names of the functions found in `game.h` so we only have to write them correctly once, here in the variable declaration. Then we can use the variable anywhere and be sure that we didn't write a typo anywhere. If we never wrote any typos and always remembered exactly the names of functions then we would not need these variables. But since we rarely remember things as well as we imagine we do, then we use these types of safeguards.

```cpp
// main.cpp
struct DLL_INFO{
  HMODULE dll;
  FILETIME timestamp;
  Function_Initialize initialize;
  Function_HandleEvents handleEvents;
  Function_Update update;
  Function_Draw draw;
  Function_OnQuit quit;
};
```

We create a struct that holds all the data we will need to work with our .DLL. The `HMODULE` is, behind the scenes, a pointer to either a .DLL or a .EXE. Now by having access to it in memory we can tell it to do things, like calling functions.

`FILETIME` is a data type "Containing a 64-bit value representing the number of 100-nanosecond intervals since January 1, 1601 (UTC)." This might seem like insane precision for us, but we use it because there are handy Windows functions that we can use to compare two `FILETIME` variables and we can also easily get the time a file was changed in the form of a `FILETIME` removing the need to convert it from another data type. Our timestamp will be used to check if our .DLL has been recompiled after we started running our game, and if it has been we re-link our dll and start sending the games data to it instead.

Then we have our function pointers that we will be pointing to the functions inside the .DLL and calling from our .EXE.

```cpp
// main.cpp
FILETIME GetTimestamp(){
  WIN32_FIND_DATA data;
  HANDLE handle = FindFirstFile(NAME_OF_DLL, &data);
  FILETIME time_of_last_change = data.ftLastWriteTime;
  FindClose(handle);
  return time_of_last_change;
}
```

We use this function to look for a file with the name we specified (`NAME_OF_DLL`) and then we can fetch the built in `LastWriteTime`. This comes in the `FILETIME` format by default. We pass along a pointer to the `WIN32_FIND_DATA` so the `FindFirstFile` function can store the information about the file it found in that variable. That's why we're required to pass it by pointer using the address-of operator `&` — meaning that we pass not the value but instead the place in memory. Once the `FindFirstFile()` function has filled the data struct with information we can get the `FILETIME`.

`FindFirstFile` allocates some memory when called. This is stored in a `HANDLE` variable — this is done no matter what by our operating system when it is asked to look for a file like this. This little chunk of memory needs to be told that it is no longer necessary to keep around by using the `FindClose()` function. If we don't free the memory from our `HANDLE` we will eventually run out of memory as every possible address in our memory is filled with an old no longer relevant `HANDLE`. Freeing memory is a core part of working with C++, but our memory arena lets us do most of that in one place rather than putting allocations and freeing calls all over our codebase.

```cpp
// main.cpp
bool LoadDLL(DLL_INFO* info, int depth = 0){
  printf("loading dll");
  if(depth > 20){
    printf("failed to write temp DLL");
    return false;
  }
  bool success = CopyFile(NAME_OF_DLL, NAME_OF_TEMP_DLL, false);
  if(!success){
    Sleep(50);
    return LoadDLL(info, depth + 1);
  }
  info->dll = LoadLibrary(NAME_OF_TEMP_DLL);
  if(info->dll == nullptr){
    printf("could not load dll");
    return false;
  }
  info->initialize = (Function_Initialize)GetProcAddress(info->dll, NAME_OF_FUNC_INIT);
  info->handleEvents = (Function_HandleEvents)GetProcAddress(info->dll, NAME_OF_FUNC_HANDLE_EVENT);
  info->update = (Function_Update)GetProcAddress(info->dll, NAME_OF_FUNC_UPDATE);
  info->draw = (Function_Draw)GetProcAddress(info->dll, NAME_OF_FUNC_DRAW);
  info->quit = (Function_OnQuit)GetProcAddress(info->dll, NAME_OF_FUNC_QUIT);
  info->timestamp = GetTimestamp();
  return true;
}
```

This function has the job of finding our compiled .DLL, create a copy of it called `NAME_OF_TEMP_DLL` then store that .DLL in our provided `DLL_INFO` struct. We then take the function pointers in our `DLL_INFO` struct and point them to the functions we tagged as `__declspec(dllexport)` in our `game.h`.

The first thing, when reading the function, that might seem strange is that the function does something bizarre that we haven't encountered before. In case the `CopyFile` function fails and the `success` bool is set to `false`, then we wait for 50 milliseconds then we return the result of calling the function again, but incrementing the `depth` parameter by 1 as it is being passed to the re-run of the function. This is called a **recursive function call** — meaning that the function has called itself.

When trying to copy a .DLL there can, if we are unlucky, be background operations being performed by our operating system that locks the file, making reading it temporarily impossible. To make sure that we:

1. Are allowed to eventually make the copy
2. Stop in case some dramatic error has happened that will never allow the file to be copied

We only allow the function to call itself a total of 20 times before stopping. What the recursive function does is this:

1. It tries to copy the .DLL. If it is successful then it just moves along as normal, doing the rest of the variable allocations we need.
2. But if it failed to copy it, we wait 50ms then the function returns, meaning that everything after the `return` will not be executed. We have aborted the function early. But we are at the same time starting up another instance of the function. This new instance will run from the top again, trying once more to copy the .DLL. If this once again is unsuccessful it will call another instance of the function and return whatever result that function gave back in its own `return`.
3. Hopefully now the value being returned is the `return true` at the end of the function meaning that we successfully allocated our `DLL_INFO`. Each time a function calls itself we pass along `depth` but first we add 1 to it. Meaning that if ten of these functions call one another still trying to get a successful copy, then `depth` will have a value of 10. If it ever reaches 20 the function returns `false` without calling itself again. This will stop the recursive loop.

This means that the function will keep calling itself unless one of two things happen:

- The .DLL successfully gets copied and its values stored in our `DLL_INFO` pointer.
- The function calls itself for the 20th time and returns `false` and not calling itself for a 21st time again.

All recursive functions could be substituted by a `while`-loop or a `for`-loop and vice-versa. And recursive functions are powerful but can be a bit hard to conceptualize at first. I tend to find it helpful to think about each function digging deeper then once we reach one of the two stop conditions we crawl back out of the stack.

```cpp
// main.cpp
void UnloadDLL(DLL_INFO* info){
  FreeLibrary(info->dll);
  info->dll = nullptr;
  DeleteFile(NAME_OF_TEMP_DLL);
}
```

This function accepts a pointer to our `DLL_INFO` struct, it then clears the memory holding the `HMODULE` aka the pointer to our .DLL — it takes the `HMODULE` pointer and nulls it. Meaning that the pointer no longer points at anything at all. It's important to note that just because we've freed the memory associated with our `HMODULE` that doesn't mean that the memory is gone — we've just told our computer that it is allowed to overwrite it. Until it does we could theoretically still use it. Then lastly we take the temporary copy of our .DLL created in the `LoadDLL` function and deletes the file.

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

This function uses `malloc` to tag a block of memory, it then returns a pointer to the first byte of that memory so it can be referenced by our memory arena. If `malloc` was unsuccessful then it returns `nullptr` and if it does we have reached a fatal error and we will be terminating our program.

```cpp
// main.cpp
void SDL_Setup(){
  SDL_Init(SDL_INIT_EVENTS);
  window = SDL_CreateWindow("pilot", 650, 400, 0);
  renderer = SDL_CreateRenderer(window, NULL);
}
```

This function as well as `AllocateGameMemory` are just functions that bundle some code we used to have one after the other in our `main()` function. We've just collected them in reasonable functions to make the `main()` function shorter and more descriptive. We could take the content of these functions and add them back into our `main()` as we had in an earlier lecture.

```cpp
// main.cpp
void CalculateDeltaTime(float& dt){
  NOW = SDL_GetTicksNS();
  dt = NOW - PREV;
  dt = SDL_NS_TO_SECONDS(dt);
  PREV = NOW;
}
```

This function does the exact same thing, taking a pointer to a float and setting it to the calculated delta time. Here we've used the `float&` rather than `float*` — this is what we called **passing by reference** rather than passing by pointer. It is just syntactic sugar that lets us work with the value found at the memory address without the `->` and dereferencing it. If this looks confusing then rewriting the function to accept a pointer instead of a reference is as simple as this:

```cpp
// main.cpp
void CalculateDeltaTime(float* dt){
  NOW = SDL_GetTicksNS();
  *dt = NOW - PREV;
  *dt = SDL_NS_TO_SECONDS(*dt);
  PREV = NOW;
}
```

Just a bit more to keep straight but as you can see it is very similar.

```cpp
// main.cpp
void DLL_CheckStatus(DLL_INFO* dll){
  FILETIME timestamp = GetTimestamp();
  bool is_timestamp_changed = CompareFileTime(&dll->timestamp, &timestamp) != 0;
  if(is_timestamp_changed){
    UnloadDLL(dll);
    LoadDLL(dll);
  }
}
```

This function uses three of the functions we wrote earlier to fetch the `FILETIME`, then we compare it and if we find that the .DLL has changed we unload the old .dll with `UnloadDLL()` then copy and create the new one using `LoadDLL()`.

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
  DLL_INFO dll;
  bool dll_successfully_loaded = LoadDLL(&dll);
  if(dll_successfully_loaded == false){
    return 2;
  }
  SDL_Setup();
  dll.initialize(gameData);
  bool running = true;
  float dt;
  while(running){
    DLL_CheckStatus(&dll);
    CalculateDeltaTime(&dt);
    SDL_Event event;
    while(SDL_PollEvent(&event)){
      running = dll.handleEvents(gameData, event);
      if(running == false){
        break;
      }
    }
    dll.update(gameData, dt);
    dll.draw(gameData, renderer);
  }
  dll.quit(renderer);
  SDL_Quit();
  return 0;
}
```

Here we do the same steps as in our previous lecture, but our `initialize()` `update()` and `draw()` functions are all called from our `DLL_INFO` struct using the function pointers we set in `LoadDLL()`. We also run the functions `SDL_Setup` and `AllocateGameMemory` to do initial setups that were previously written as-is in our `main()`. Whenever we call a function we could imagine taking all the code inside that function and just copy-pasting it into the call site. As a summary, this is what our `main()` does:

1. We allocate a blob of memory
2. We create a memory arena and point it to the start of our memory blob
3. We load our DLL and set up our function pointers
4. We set up SDL, creating our window and assigning our renderer
5. We begin our core loop
6. Inside our core loop we compare FILETIME for our DLL
7. Then we calculate delta time
8. We collect all SDL_Events and pass them to the `HandleEvent` function in our .DLL
9. We then call `update` followed by `draw` in our .DLL
10. And if our core loop ever terminates we do some cleanup and return 0

We are done! We have now successfully added the boilerplate code necessary to work with our .EXE and .DLL setup, allowing us to start developing a game in our next lecture. But before that we're going to see the magic of our system by doing a hot-reload of our .DLL.

Inside our `game.cpp` in the `Draw()` function we call `SetRenderDrawColor()`. We can now do the following steps:

1. Build the game using `build name_of_project` in PowerShell
2. Run the game using `run name_of_project`
3. As the game is running, change the colors of our `SDL_SetRenderDrawColor()`
4. Compile our .DLL using `reload name_of_project` prompting clang to just rebuild the .DLL
5. Back in our still running program we can see that the changes we made to our `game.cpp` is directly visible in the game, without us having to close the game and running it again.

This ability to make live-changes to our game is such a huge win for us and will make development of any game so much more streamlined!
