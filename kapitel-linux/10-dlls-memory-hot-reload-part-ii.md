> **Linux:** This chapter is adapted for Linux.

# DLLs Memory and Hot Reloading - Part II

It's time to head back to our SEL3 project to set up the boilerplate necessary to use our executable + shared library (.so) system.

In our previous example we had everything in one placeholder practice example.cpp. Now we will start breaking things into separate .cpp files along with corresponding .h files.

At the end of this lecture we will have the following files in our src folder:

- `arena.cpp` -- holds the implementation of functions from arena.h
- `arena.h` -- holds the declaration of functions for our memory arena as well as the arena struct
- `common.h` -- A helper .h file containing some helpful macros to figure out memory sizes in kb, mb and gb
- `game.cpp` -- Acts as the "entry point" for the shared library and performs our Input, Update, Draw routines
- `game.h` -- Holds the definitions for the functions used in game.cpp and has them tagged in such a way that we can find them from our main executable
- `gameState.h` -- a .h file containing the struct with all variables used inside the game
- `main.cpp` -- our executable entry point, initializes everything, sets up memory and the game loop. Calls into our shared library through functions found in game.h

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

At the top of this .h file we write `#pragma once` -- we will be doing this for ALL .h files we ever write. This is a not-so-nice feature of C++ where without it our .h file will be copied above all files that implement it, meaning that our executable or shared library bloats unnecessarily. By adding `#pragma once` our compiler knows to only add these once, which is enough.

We have encapsulated our struct and function declarations in a namespace we've named `Memory`. This means that when we `#include "arena.h"` we can only access the struct and functions by first specifying the namespace -- for example: `Memory::Initialize()`

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

End of file : )