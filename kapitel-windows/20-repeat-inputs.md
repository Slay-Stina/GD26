# 20 Repeat Inputs

In this chapter we'll add the ability to hold down a key and get our entities to keep moving, and undos to keep undoing. Sparing us from having to press a key each time we want to perform an action (though that functionality will of course remain)
We're also going to do some housekeeping and move key press logic out of `game.cpp` and firmly into its own script - as this fits better as part of our boilerplate.
We're going to create a new struct that will live inside a newly created `input.h`

```cpp
// input.h
#pragma once
struct Input {
  const bool* keys_current;
  const bool* keys_previous;
  float* keys_held_time;
};
```

`keys_current/previous` are set to `const` as we are not looking to allow their contents to be changed individually. but `keys_held_time` will be updated on an individual level. Each of these variables are used as an array, indicated by the plural `keys` rather than `key` .
`keys_held_time` is an array of floats that track how long each key has been held down. We're going to use this to simplify checking how long a key has been held.
We'll add one of these structs to our `GameData` inside `gameState.h` . We're also removing the variables related to keys that lived directly inside `GameData` earlier. We also previously quite lazily allocated our key arrays into `arena_level` but we don't really want to free the memory of these arrays. But we might want to reset their values. We're also going to add a new `arena_input` and create this new subarena from `arena_main` .

```cpp
// gameState.h
#include "input.h"
struct GameData {
  // other variables hidden for clarity
  Input input;
  Arena* arena_input;
  // bool* keys_previous; <- removed
};
```

inside `main.cpp` we'll create this subarena and allocate our arrays to it.

```cpp
// main.cpp
GameData* gameData = (GameData*)Memory::Allocate(arena_main, sizeof(GameData));

size_t INPUT_ARENA_SIZE = 0;
INPUT_ARENA_SIZE += sizeof(bool) * SDL_SCANCODE_COUNT * 2;
INPUT_ARENA_SIZE += sizeof(float) * SDL_SCANCODE_COUNT;
INPUT_ARENA_SIZE += 128;
gameData->arena_input = Memory::CreateSubArena(arena_main, INPUT_ARENA_SIZE);
gameData->input.keys_current = (bool*)Memory::Allocate(gameData->arena_input, sizeof(bool) * SDL_SCANCODE_COUNT);
gameData->input.keys_previous = (bool*)Memory::Allocate(gameData->arena_input, sizeof(bool) * SDL_SCANCODE_COUNT);
gameData->input.keys_held_time = (float*)Memory::Allocate(gameData->arena_input, sizeof(float) * SDL_SCANCODE_COUNT);
```

We collect the total size for all our arrays and add them to `INPUT_ARENA_SIZE` , making sure two multiply by 2 to get the size of both our `bool*` . Then we allocate our SubArena and finally allocate the three arrays into that arena. We also lazily add on 128 bytes as our `Allocate()` function is a bit naive and does not take into account that sometimes a new allocation will be padded a little. creating a gap of a few bytes. This is only necessary because

1. our `Allocate()` is a bit naive
2. we are slicing of juuuuust enough memory to hold the three arrays

Now inside our `input.h` we'll add the functions that previously lived inside `game.cpp` (as well as 4 new ones). These functions live outside of the struct

```cpp
// input.h
bool KeyPressed(const Input* input, SDL_Scancode key);
bool KeyHeld(const Input* input, SDL_Scancode key);
bool KeyReleased(const Input* input, SDL_Scancode key);
bool KeyHeld_ForTime(const Input* input, SDL_Scancode key, float min_length);
void UpdateKeys(Input* input, float dt);
void ResetKeyHeldTime(Input* input, SDL_Scancode key);
void ResetAll(Input*);
```

Note that the parameters for each function have changed and that there are generally fewer. Previously we passed `keys` and `keys_previous` as separate variables each time. Now we send `Input` that holds both of these. we pass in `Input*` as a `const` as we are not letting those functions modify the arrays.
Inside `input.cpp` we create the content of each of these functions

```cpp
// input.cpp
#include "input.h"
#include <cstring>

bool KeyPressed(const Input* input, SDL_Scancode key) {
  if(input->keys_previous == nullptr) {
    return input->keys_current[key];
  }
  return input->keys_current[key] && !input->keys_previous[key];
}

bool KeyHeld(const Input* input, SDL_Scancode key) {
  if(input->keys_previous == nullptr) {
    return false;
  }
  return input->keys_current[key] && input->keys_previous[key];
}

bool KeyReleased(const Input* input, SDL_Scancode key) {
  if(input->keys_previous == nullptr) {
    return false;
  }
  return !input->keys_current[key] && input->keys_previous[key];
}
```

these functions are the same as the ones we used inside `game.cpp` but we access `keys_current/previous` from our `Input` parameter that was passed in.

```cpp
// input.cpp
bool KeyHeld_ForTime(const Input* input, SDL_Scancode key, float min_length) {
  return input->keys_held_time[key] >= min_length;
}

void UpdateKeys(Input* input, float dt) {
  for (int i = 0; i < SDL_SCANCODE_COUNT; i++) {
    if (input->keys_current[i]) {
      input->keys_held_time[i] += dt;
    }
    else {
      input->keys_held_time[i] = 0;
    }
  }
  memcpy((void*)input->keys_previous, input->keys_current, SDL_SCANCODE_COUNT * sizeof(bool));
}

void ResetKeyHeldTime(Input* input, SDL_Scancode key) {
  input->keys_held_time[key] = 0;
}

void ResetAll(Input* input) {
  memset((void*)input->keys_current, 0, sizeof(bool) * SDL_SCANCODE_COUNT);
  memset((void*)input->keys_previous, 0, sizeof(bool) * SDL_SCANCODE_COUNT);
  memset((void*)input->keys_held_time, 0, sizeof(float) * SDL_SCANCODE_COUNT);
}
```

For `KeyuHeld_ForTime()` we pass along a float then find the specified key from our `keys_held_time[]` array and check if the timer for that key exceeded the time we passed in.
`UpdateKeys()` is our consolidated function that runs over all keys and increased the held timer by deltatime / `dt` if it was held this frame. If not we reset the `held_timer` for that key. After that is done we take `keys_current` and copy over all of those values to `keys_previous` . We'll call this function from `main.cpp` after we've called `Update()`
`ResetKeyHeldTime()` accepts a key then sets the timer for that key to 0
`ResetAll()` uses the `memset()` function to fill every memory address for our three arrays with zeroes. This means that they still exist but nothing is stored. We'll use this later to clear the inputs as we load or reload a level. We have to cast our `bool*` and `float*` to `void*` as that is the parameter that `memset` uses.
Lets head to `main.cpp` and add this new functionality

```cpp
// main.cpp
gameData->input.keys_current = SDL_GetKeyboardState(nullptr);
dll.update(gameData, dt);
UpdateKeys(&gameData->input, dt);
dll.draw(gameData, renderer);
// memcpy((void*)gameData->keys_previous, SDL_GetKeyboardState(nullptr), SDL_SCANCODE_COUNT * sizeof(bool)); // removed
```

So before `Update()` we collect the keys being pressed. then after `Update()` we update our `held_timers` and copy over `keys_current` to `keys_previous` in preparation for the next tick using our new `UpdateKeys()` function. We also remove the old `memcpy` we created last chapter as we do this inside `UpdateKeys()` now.
We also have to do a small amount of coding inside our `cmakelists.txt` . Currently our exe does not get access to `input.cpp` but it calls `UpdateKeys()` .

```cmake
set(SHARED_SOURCES
  ${CMAKE_SOURCE_DIR}/src/image.cpp
  ${CMAKE_SOURCE_DIR}/src/arena.cpp
  ${CMAKE_SOURCE_DIR}/src/input.cpp
)
```

Now we can remove the old `Pressed/Held/Released` function from `game.cpp` and instead `#include "input.h"` and our new simplified functions. We are also removing the old functions (`Pressed, Held, Released`) from `game.h`

```cpp
// game.cpp
if(KeyPressed(&data->input, SDL_SCANCODE_Z)) {
  if(KeyHeld(&data->input, SDL_SCANCODE_LSHIFT)) {
    Redo(data->commandBuffer);
  }
  else {
    Undo(data->commandBuffer);
  }
}
if(KeyPressed(&data->input, SDL_SCANCODE_RIGHT)) {
  data->input_buffer[data->input_buffer_write_count++ % data->input_buffer_capacity] = {1, 0};
}
else if(KeyPressed(&data->input, SDL_SCANCODE_LEFT)) {
  data->input_buffer[data->input_buffer_write_count++ % data->input_buffer_capacity] = {-1, 0};
}
else if(KeyPressed(&data->input, SDL_SCANCODE_UP)) {
  data->input_buffer[data->input_buffer_write_count++ % data->input_buffer_capacity] = {0, -1};
}
else if(KeyPressed(&data->input, SDL_SCANCODE_DOWN)) {
  data->input_buffer[data->input_buffer_write_count++ % data->input_buffer_capacity] = {0, 1};
}
```

With this we can clearly see how our housekeeping has simplified calling these functions. It currently does the same thing, but it's easier to digest.
Now we're going to expand these functions using the or evaluator written as `||` . This will make an if-statement be true if either of the conditions checked is true.

```cpp
// or-evaluator example
if((weapon.damage >= enemy->health) || cheats->max_damage) {
  KillEnemy(enemy);
}
```

This pseudo-code would kill the enemy if either our weapon had enough damage or the `max_damage` bool from cheats were set to true.

```cpp
// game.cpp
if(KeyPressed(&data->input, SDL_SCANCODE_Z) || KeyHeld_ForTime(&data->input, SDL_SCANCODE_Z, UNDO_REPEAT_TIME)) {
  ResetKeyHeldTime(&data->input, SDL_SCANCODE_Z);
  if(KeyHeld(&data->input, SDL_SCANCODE_LSHIFT)) {
    Redo(data->commandBuffer);
  }
  else {
    Undo(data->commandBuffer);
  }
}
if(KeyPressed(&data->input, SDL_SCANCODE_RIGHT) || KeyHeld_ForTime(&data->input, SDL_SCANCODE_RIGHT, (1 / MOVE_SPEED) * 1.15)) {
  ResetKeyHeldTime(&data->input, SDL_SCANCODE_RIGHT);
  data->input_buffer[data->input_buffer_write_count++ % data->input_buffer_capacity] = {1, 0};
}
else if(KeyPressed(&data->input, SDL_SCANCODE_LEFT) || KeyHeld_ForTime(&data->input, SDL_SCANCODE_LEFT, (1 / MOVE_SPEED) * 1.15)) {
  ResetKeyHeldTime(&data->input, SDL_SCANCODE_LEFT);
  data->input_buffer[data->input_buffer_write_count++ % data->input_buffer_capacity] = {-1, 0};
}
else if(KeyPressed(&data->input, SDL_SCANCODE_UP) || KeyHeld_ForTime(&data->input, SDL_SCANCODE_UP, (1 / MOVE_SPEED) * 1.15)) {
  ResetKeyHeldTime(&data->input, SDL_SCANCODE_UP);
  data->input_buffer[data->input_buffer_write_count++ % data->input_buffer_capacity] = {0, -1};
}
else if(KeyPressed(&data->input, SDL_SCANCODE_DOWN) || KeyHeld_ForTime(&data->input, SDL_SCANCODE_DOWN, (1 / MOVE_SPEED) * 1.15)) {
  ResetKeyHeldTime(&data->input, SDL_SCANCODE_DOWN);
  data->input_buffer[data->input_buffer_write_count++ % data->input_buffer_capacity] = {0, 1};
}
```

suddenly there is more code here, but remember for each direction we press the code is actually almost identical.
we've also just added `UNDO_REPEAT_TIME` to our `common.h`

```cpp
// common.h
const float UNDO_REPEAT_TIME = 0.15;
```

this new or evaluator in our undo if-statement will allow us to repeatedly undo once every `0.15` seconds as long as the Z key is being held down. Once the if-statement evaluates to true we call `ResetKeyHeldTime` to set the timer keeping track of the Z key back to `0` . So that we need to wait another `0.15` seconds for the next undo.
lets look at the first of the right/left/up/down input blocks (as the rest are just copies)

```cpp
// game.cpp
if(KeyPressed(&data->input, SDL_SCANCODE_RIGHT) || KeyHeld_ForTime(&data->input, SDL_SCANCODE_RIGHT, (1 / MOVE_SPEED) * 1.15)) {
  ResetKeyHeldTime(&data->input, SDL_SCANCODE_RIGHT);
  data->input_buffer[data->input_buffer_write_count++ % data->input_buffer_capacity] = {1, 0};
}
```

we use the same `||` to allow for a new input.
But the float we pass as a parameter to `KeyHeld_ForTime()` is a bit more complex. We actually pass in the time it takes for a move animation to finish but increased by 15%. This is honestly not a very good approach but it will do for now. It means that the next input will only be logged after a full movement has elapsed, (plus a little extra). We'll definitely return to this little equation later and improve it.
We pass `&data->input` using the pointer to symbol aka `&` we do this because our `data->input` is not a pointer but our `KeyPressed()` function expects a pointer. We pass by pointer to avoid copying over the array each time, as this creates unecesary overhead (CPU work).
If we are allowed inside the if-statement we reset the timer for the specific key then add the correct `Position*` to our ring buffer and increase `_write_count` by 1 afterwards using the increment operator aka `++` .
Now we can hold our undo and movement keys instead of clicking all the time. We have also put our input logic inside our boilerplate and simplified calling `Pressed/Held/Released` .