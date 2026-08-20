> **Linux:** This chapter is adapted for Linux.

# Sokoban Programming II

It's time to get a player character moving on the screen. For this we will need to work with our data inside of our `Update()` inside `Game.cpp`. We will add behaviour as flags to our entities then based on those behaviours we will treat them differently.

Lets look at our updated `Entity.h`:

```cpp
#pragma once
#include <cassert>
#include <cstdint>
enum Behaviour : uint32_t {
  NONE = 0,
  CAN_MOVE = 1 << 0,
  IS_PLAYER = 1 << 1,
  RESPOND_TO_INPUT = 1 << 2
};
enum class ID : uint8_t {
  GROUND = 1,
  WALL = 2,
  PLAYER = 3
};
```

We're working with a new concept here — `enum` — and right from the start we're using two different versions: `enum` and `enum class`. An `enum` is a named number. Looking at `ID` we can see that each of our tiles have been designated a number.

By adding the `class` attribute we make it so we can only access our enums by first specifying the class like so: `ID::GROUND`. This is very similar to a namespace.

We also have a new operator `<<` used for our `Behaviour` — it's known as one of many **bitwise operators**. A `uint32_t` holds 32 bits to create its number as opposed to a `uint8_t` that holds 8 bits.

Each time we add 1 we flip the rightmost bit to 1. If it was already 1 we flip it back to 0 then flip the bit to the left of it to 1. This means that each bit to the left of the previous is tasked with holding a number twice as large.

For our behaviour flags to work each number used has to be a unique bit. This means that we can store 8 enum behaviour flags in a `uint8_t` and 32 of them in a `uint32_t`.

Knowing the value of our bits when flipped to 1 we could write our enum `Behaviour` like this:

```cpp
enum Behaviour : uint32_t {
  NONE = 0,
  CAN_MOVE = 1,
  IS_PLAYER = 2,
  RESPOND_TO_INPUT = 4
};
```

Keep in mind that we did not "miss" 3 — we are not allowed to use that number as it could be created by combining 1 and 2 together.

Inside our `struct Entity {}` we've added a new variable as well as changing our `uint8_t id` to `ID id`:

```cpp
ID id;
int x;
int y;
Behaviour behaviour;
```

Then inside our struct we add a series of functions:

```cpp
bool HasBehaviour(Behaviour flags){
  return (behaviour & flags) == flags;
}
```

`HasBehaviour()` takes a flag (or flags as they are collected in one single variable) and checks an `&` operation between them. This boolean function only returns `true` if all the bits in `flags` were also set to 1 in `behaviour`.

```cpp
void InitializeBaseBehaviour(){
  assert(id != ID::NONE);
  switch (id) {
    default:
      SetBehaviour(NONE);
      break;
    case ID::PLAYER:
      SetBehaviour((Behaviour)(CAN_MOVE | IS_PLAYER | RESPOND_TO_INPUT));
      break;
  }
}
```

```cpp
void SetBehaviour(Behaviour flags){
  behaviour = flags;
}
void AddBehaviour(Behaviour flags){
  behaviour = (Behaviour)(behaviour | flags);
}
void RemoveBehaviour(Behaviour flags){
  behaviour = (Behaviour)(behaviour & ~flags);
}
```

Updated `CreateEntities()`:

```cpp
if(entity_id != 0){
  int x = i % lvl_data->w;
  int y = i / lvl_data->w;
  lvl_data->entityBuffer[index].id = (ID)entity_id;
  lvl_data->entityBuffer[index].InitializeBaseBehaviour();
  lvl_data->entityBuffer[index].x = x;
  lvl_data->entityBuffer[index].y = y;
  index += 1;
}
```

In our `GameData` struct we need to store an array of the status of all keys on the previous tick:

```cpp
bool* keys_previous;
```

Allocate this block of memory in our `main.cpp`:

```cpp
gameData->keys_previous = (bool*)Memory::Allocate(gameData->arena_levels, sizeof(bool) * SDL_SCANCODE_COUNT);
```

Inside our `update()` in `game.cpp`:

```cpp
// at the top of the Update function
const bool* keys = SDL_GetKeyboardState(nullptr);
...
// at the bottom of the Update function
memcpy((void*)data->keys_previous, keys, SDL_SCANCODE_COUNT * sizeof(bool));
```

In `game.h`:

```cpp
bool KeyPressed(SDL_Scancode key, const bool* current, const bool* previous);
bool KeyHeld(SDL_Scancode key, const bool* current, const bool* previous);
bool KeyReleased(SDL_Scancode key, const bool* current, const bool* previous);
```

In `game.cpp`:

```cpp
bool KeyPressed(SDL_Scancode key, const bool* current, const bool* previous){
  if(previous == nullptr){
    return current[key];
  }
  return current[key] && !previous[key];
}
bool KeyHeld(SDL_Scancode key, const bool* current, const bool* previous){
  if(previous == nullptr){
    return false;
  }
  return current[key] && previous[key];
}
bool KeyReleased(SDL_Scancode key, const bool* current, const bool* previous){
  if(previous == nullptr){
    return false;
  }
  return !current[key] && previous[key];
}
```

Add `GetCurrentLevel()` to `GameData`:

```cpp
struct GameData {
  LevelData* levels;
  int currentLevelIndex;
  LevelData* GetCurrentLevel(){
    return &levels[currentLevelIndex];
  }
```

Our `update()` in `game.cpp`:

```cpp
void Update(GameData* data,float dt){
  const bool* keys = SDL_GetKeyboardState(nullptr);
  for (int i = 0; i < data->GetCurrentLevel()->entityCount; i++) {
    Entity* entity = &data->GetCurrentLevel()->entityBuffer[i];
    if(entity->HasBehaviour((Behaviour)(Behaviour::RESPOND_TO_INPUT | Behaviour::CAN_MOVE))){
      int xChange = 0;
      int yChange = 0;
      if(KeyPressed(SDL_SCANCODE_RIGHT, keys, data->keys_previous)){
        xChange = 1;
      }
      else if(KeyPressed(SDL_SCANCODE_LEFT, keys, data->keys_previous)){
        xChange = -1;
      }
      else if(KeyPressed(SDL_SCANCODE_UP, keys, data->keys_previous)){
        yChange = -1;
      }
      else if(KeyPressed(SDL_SCANCODE_DOWN, keys, data->keys_previous)){
        yChange = 1;
      }
      if(xChange != 0 || yChange != 0){
        int stepInto_x = entity->x + xChange;
        int stepInto_y = entity->y + yChange;
        Entity* stepInto_entity = data->GetCurrentLevel()->GetEntity(stepInto_x, stepInto_y);
        uint8_t stepInto_tile_id = data->GetCurrentLevel()->GetCellID(stepInto_x, stepInto_y);
        if(stepInto_entity == nullptr){
          if(stepInto_tile_id == (uint8_t)ID::GROUND){
            entity->x = stepInto_x;
            entity->y = stepInto_y;
          }
        }
      }
    }
  }
  memcpy((void*)data->keys_previous, keys, SDL_SCANCODE_COUNT * sizeof(bool));
}
```

With this our player entity can move around the level using the arrow keys!
