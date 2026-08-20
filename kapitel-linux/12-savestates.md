> **Linux:** This chapter is adapted for Linux.

# Savestates

With our memory arenas set up and our game infrastructure being made almost entirely from scratch we can start to do some pretty impressive things. The first of these will be us saving and retrieving a complete snapshot of the state of the game.

To accomplish this we will need to:

1. Have a function that converts the game state into binary
2. Write that binary data into a text file
3. Read the binary data from the text file
4. And then... overwrite the binary data in our memory arena to the binary data we read

For now we'll copy over the entire block of memory into a .bin file. It's worth noting that this system is not currently equipped to handle being our official save/load system meant for consumers. But that is an issue we'll tackle later. A .bin file is just like a .txt but the file type indicates to us humans that it's holding binary data only.

We will be adding:

1. One new `#include`
2. Two new functions, both just 2 lines long
3. Two if-statements inside our `while(running)` loop

That is it.

In our `#include` section we will add:

```cpp
#include <fstream>
```

This gives us access to built-in functionality that lets us read and write files.

We then create two functions:

```cpp
void StoreGameState(Memory::Arena* arena){
  std::ofstream file("temp_state.bin", std::ios::binary);
  file.write(reinterpret_cast<const char*>(arena->base), arena->size);
}
void RetrieveGameState(Memory::Arena* arena){
  std::ifstream file("temp_state.bin", std::ios::binary);
  file.read(reinterpret_cast<char*>(arena->base), arena->size);
}
```

`StoreGameState()` writes the contents of the provided memory arena to a file. `RetrieveGameState()` reads the content of a file and overwrites the contents of the memory arena.

The nice part with our memory arena being a simple struct with a pointer to the first byte and a `size_t` for the total size of the arena is that this is precisely the two parameters needed to read the contents of the file into a place in memory.

With `<fstream>` included we get access to `ofstream` and `ifstream` responsible for writing and reading a file respectively.

Both an `ofstream` and an `ifstream` accept 2 optional parameters. The first being a `char*` for the name and the second an option for how the data is supposed to be interpreted. In our case we store it as binary (0's and 1's) because that is the exact same thing our memory actually is!

We then call `.write` and `.read` and because `.write` expects to get a `const char*` and our `arena->base` is an `unsigned char*` we need to use `reinterpret_cast<...>` to tell our program to treat it as if it were of the correct type.

We need to call `file.close()` at the end of each function as the filestreams we've created have allocated memory on our computer and needs to be freed so other memory can be allowed to overwrite it.

With these functions set up we just have to call them when we press the keyboard. Inside our `while(running)` we expand our `while(SDL_PollEvent(&event))` to include two if-statements -- one for pressing F9 and one for pressing F10:

```cpp
while(SDL_PollEvent(&event)){
  running = dll.handleEvents(gameData, event);
  if(running == false){
    break;
  }
}
if(event.type == SDL_EVENT_KEY_DOWN){
  if(event.key.key == SDLK_F9){
    StoreGameState(arena_main);
  }
  if(event.key.key == SDLK_F10){
    RetrieveGameState(arena_main);
  }
}
```

We first check if the event was a `SDL_EVENT_KEY_DOWN` to not try and get the keystroke info from another event entirely. Then we check `event.key.key` -- the first `key` in `event.key` is a struct `SDL_KeyboardEvent`, the second `key` in `event.key.key` is a variable inside `SDL_KeyboardEvent` that is of the type `SDL_Keycode` also unfortunately named `key`. We compare the keycode to the specified `SDLK` enum and if we get a match we call the specified function.

And with that we're actually done. We have everything needed to save and load our gamestate. Now we can go into our game, make any changes we want, save the gamestate with F9 then whenever we press F10 we are instantly back at that exact point again.

Boom!