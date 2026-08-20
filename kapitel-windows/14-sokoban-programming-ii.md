# Sokoban Programming II

It's time to get a player character moving on the screen. For this we will need to work with our data inside of our `Update()` inside `Game.cpp`. We will add behaviour as flags to our entities then based on those behaviours we will treat them differently.

Lets look at our updated `Entity.h`:

```cpp
#pragma once
#include <cassert> // So we can use assert()
#include <cstdint> // So we have access to uint8_t
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

We're working with a new concept here — `enum` — and right from the start we're using two different versions: `enum` and `enum class`. An `enum` is a named number. Looking at `ID` we can see that each of our tiles have been designated a number. We do this to sync the id from our .tmj file with our code, so that they match. This is a pretty flimsy setup because if we change the order of our tiles in Tiled then their id will change and this will no longer match up. We will look at improving this system later, but for now, as long as we keep the id-to-enum setup correct we are good to go.

By adding the `class` attribute we make it so we can only access our enums by first specifying the class like so: `ID::GROUND`. This is very similar to a namespace.

We also have a new operator `<<` used for our `Behaviour` — it's known as one of many **bitwise operators**. A `uint32_t` holds 32 bits to create its number as opposed to a `uint8_t` that holds 8 bits:

```
00000000
```

This is the bits of a `uint8_t` — they can either be 0 or 1. We can turn a number of these on/off by setting them to 1:

```
01000101
```

This is like a list of 8 booleans, and that is how we're treating them. We are representing one unique behaviour of an entity with one of these bits. For example if the second bit (right to left) has a value of 1 then `IS_PLAYER` is `true`. If the value is 0 then `IS_PLAYER` is `false`. With a `uint32_t` we can store 32 booleans in a single location, using the enum name to fetch them.

Lets look at the numbers 0, 1, 2, 3, 5, 10 and 32 in bit-format:

```
00000000 = 0
00000001 = 1
00000010 = 2
00000011 = 3
00000101 = 5
01000000 = 32
```

Each time we add 1 we flip the rightmost bit to 1. If it was already 1 we flip it back to 0 then flip the bit to the left of it to 1. If that was also a 1 already we flip its leftside bit (and so on). This means that each bit to the left of the previous is tasked with holding a number twice as large:

```
128 64 32 16 8 4 2 1
 0  0  0  0 0 0 0 0
```

If all the bits are set to 1 in a `uint8_t` then the total value would be 256. A value of 255 would have all bits except the rightmost set to a 1. As our `ID` enum is not a series of flags but an indicator of the type of tile from Tiled we don't need more than a `uint8_t` as it will be quite a long time before we need more than 256 tiles.

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

Keep in mind that we did not "miss" 3 — we are not allowed to use that number as it could be created by combining 1 and 2 together, meaning that it is not uniquely represented by a single bit. Therefore the sequence is: 0, 1, 2, 4, 8, 16, 32, 64, 128... and so on.

Using the bitwise operator `<<` we can learn our first action that manipulates bits. This is a **left-shift** operation that pushes a bit a certain amount of steps to the left:

```cpp
IS_PLAYER = 1 << 1
```

This moves a 1 toggled bit 1 step to the left. Resulting in `00000010` aka a 2.

```cpp
RESPOND_TO_INPUT = 1 << 2
```

This moves a 1 toggled bit 2 steps to the left. Resulting in `00000100` aka a 4.

Now we can see how our bitwise left shift and our simpler `= 0, 1, 2, 4, 8...` versions are the same.

Inside our `struct Entity {}` we've added a new variable as well as changing our `uint8_t id` to `ID id`:

```cpp
ID id;
int x;
int y;
Behaviour behaviour;
```

With the changes from `uint8_t` to our `ID` enum we need to update `levels.cpp` and `levelRenderer.cpp`:

```cpp
lvl_data->entityBuffer[index].id = (ID)entity_id;
```

In `levels.cpp` we need to cast our `entity_id` to `ID` so it can be set to our `id`.

```cpp
case ID::PLAYER:
```

In `levelRenderer.cpp` we switch from the hard-coded "3" to our `ID`, being much more specific. Avoiding putting what is known as **magic numbers** in our code.

Then inside our struct we add a series of functions. These live inside our struct so we can access them from an `Entity` like `entity->function()`:

```cpp
bool HasBehaviour(Behaviour flags){
  return (behaviour & flags) == flags;
}
```

`HasBehaviour()` takes a flag (or flags as they are collected in one single variable) and checks an `&` operation between them. This means that wherever there is a 1 in both sequences a 1 will be added to the output:

```
01001001
00001011
=
00001001
```

This boolean function only returns `true` if all the bits in `flags` were also set to 1 in `behaviour`. This allows us to check multiple flags at once.

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

From our `CreateEntities()` function in `levels.cpp` we call `InitializeBaseBehaviour`. This checks what `ID` we've assigned to the entity and then gives it its relevant starting behaviours. We are able to pass along multiple behaviour flags at once by adding them together using the **bitwise or** operator, making the resulting bit a 1 if either of the bits checked were a 1.

We also use `assert` from `<cassert>` — this allows our program to, if the assertion was false, to fail. This will let us learn that we have a critical error and halt our program. We use this to catch errors during development. And we can remove these once we build the final version of the game. The reasoning is that if our `assert` fails then we don't want to continue running our application as we've introduced a fatal error that needs to be addressed. In this case the assert checks that all entities that we initialize have had their `ID` set.

```cpp
void SetBehaviour(Behaviour flags){
  behaviour = flags;
}
```

Our `SetBehaviour()` function overwrites the current flags with the flags added as a parameter.

```cpp
void AddBehaviour(Behaviour flags){
  behaviour = (Behaviour)(behaviour | flags);
}
```

`AddBehaviour` combines the current flags with another flag (or multiple flags) using the bitwise or.

```cpp
void RemoveBehaviour(Behaviour flags){
  behaviour = (Behaviour)(behaviour & ~flags);
}
```

The `RemoveBehaviour()` function uses the **bitwise and** `&` that sets the bit to 1 if both the bits checked were 1. And the **bitwise not** `~` that inverts all 1's and 0's — turning ones into zeroes and vice-versa. The combination of these bitwise operators will remove flags from `behaviour`.

Lets look at how we've updated `CreateEntities()`:

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

We've added a call to `InitializeBaseBehaviour()` as well as fixed a cast from `entity_id` to `ID`, a necessary change to compile our project now that `id` is an enum.

The next step is writing code inside `game.cpp` fetching the currently pressed, held and released keyboard keys then using that info to manipulate our entities.

In our `GameData` struct we need to store an array of the status of all keys on the previous tick. We will compare this previous key array to the current one:

- If a key was pressed in both previous and current then it will be treated as **held**
- If a key was pressed this frame but not last frame then it will be treated as **pressed**
- If a key was pressed last frame but not on the current frame then it will be treated as **released**

The `SDL_SCANCODE` enum has a `SDL_SCANCODE_COUNT` — this is not a key on the keyboard but instead the last value, letting us know how many keys are present in total. We can now allocate a block of memory to hold, in sequence, this amount of `bool`s.

We add a pointer inside `GameData`:

```cpp
bool* keys_previous;
```

Then allocate this block of memory in our `main.cpp`:

```cpp
gameData->keys_previous = (bool*)Memory::Allocate(gameData->arena_levels, sizeof(bool) * SDL_SCANCODE_COUNT);
```

Inside our `update()` in `game.cpp` we can fetch the current array of bools and at the end of the function we can assign the now old values to `keys_previous`:

```cpp
// at the top of the Update function
const bool* keys = SDL_GetKeyboardState(nullptr);
...
// at the bottom of the Update function
memcpy((void*)data->keys_previous, keys, SDL_SCANCODE_COUNT * sizeof(bool));
```

Our `keys` pointer points to the place in memory where SDL upon initialization holds the state of the keyboard. We could pass an optional `int*` pointer to this function to retrieve the length of the returned array, this is not necessary for us, so we pass `nullptr` instead.

The `memcpy` function simplifies what would otherwise be a for-loop. Instead of looping over all our `SDL_SCANCODE_XXXX` we take a destination and a location in memory as well as a size to copy. This takes the data in `keys` and copies those values to the position in memory starting from `keys_previous`. If we had just assigned `keys_previous = keys` these pointers would both point to the same place in memory, both updating simultaneously as they are actually referencing the same bit of memory. With this `memcpy` we are just grabbing the values. And as we know that we allocated `SDL_SCANCODE_COUNT * sizeof(bool)` amount of memory and that holds every single key, then we can safely use it here to tell `memcpy` how much memory to read and copy.

Now we can use `keys` and `keys_previous` to create our functions that can tell us if a key was pressed, released or held.

We will open `game.h` and, outside of our `extern "C"` block we will create the function signatures for 3 functions. These three functions are not meant to be called from outside the DLL so they don't use any of those tags for them.

```cpp
bool KeyPressed(SDL_Scancode key, const bool* current, const bool* previous);
bool KeyHeld(SDL_Scancode key, const bool* current, const bool* previous);
bool KeyReleased(SDL_Scancode key, const bool* current, const bool* previous);
```

Each of these functions have the same parameters passed into them. First the `SDL_SCANCODE` that we are interested in, followed by the current values of each key, lastly the state of each key on the previous frame.

Back in `game.cpp` we can create the functions, once again outside of our `extern "C"` block:

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

When we use the and operator `&&` we can do some simple boolean logic: `false && true = false` and `true && true = true`. Our `KeyPressed` checks if `current[key]` is true and `previous[key]` is false using the not operator `!`.

We could also write it as:

```cpp
return current[key] == true && previous[key] == false;
```

Each of the three functions also have a defensive part where we first check if `previous` was `nullptr`. This is true if:

a) We have messed something up
b) It's the very first frame of the game, and nothing has been stored in `keys_previous` yet.

We will be accessing the `entityBuffer` and `cells` in the `currentLevel` a lot. At this point the only way we can access the level we're on is by the following code:

```cpp
data->levels[data->currentLevel]
```

Having to write this long-ish line, referring to `data` twice is a bit clumsy. Lets simplify our lives a bit by adding a `GetCurrentLevel()` function to our `LevelData` struct:

```cpp
struct GameData {
  // non-relevant variables omitted
  LevelData* levels;
  int currentLevelIndex;
  LevelData* GetCurrentLevel(){
    return &levels[currentLevelIndex];
  }
```

It's important that we return the specified level as a pointer reference using `&` and making the return type a `LevelData*` pointer. Otherwise we would be returning not the specified level but a copy of it. This copy would be discarded as soon as it goes out of scope. Meaning that any changes will not be reflected in the actual level.

With our keyboard logic set up as well as our entity behaviour we can start asking questions about our entities and the state of our inputs to drive our game.

Lets look at our `update()` in `game.cpp`:

```cpp
void Update(GameData* data,float dt){
  const bool* keys = SDL_GetKeyboardState(nullptr);
  for (int i = 0; i < data->GetCurrentLevel()->entityCount; i++) {
    Entity* entity = &data->GetCurrentLevel()->entityBuffer[i];
```

The first bit of code stores the current keyboard keys, we then use our new `GetCurrentLevel()` to fetch the `entityCount`. We use this to loop over our entities by going from 0 to `entityCount`. Inside our for-loop we fetch a pointer to an entity by taking the entity stored in the current `i` location. Both `entityCount` and `entityBuffer` are part of `LevelData` and is fetched from our `GameData* data`.

```cpp
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
```

Looping over all entities we check if the flags `RESPOND_TO_INPUT` and `CAN_MOVE` are both set to 1 using `HasBehaviour()`. Only if both of these are 1 do we continue.

We then create 2 variables, meant to store the direction we will be travelling in based on keyboard inputs. We use `if-else` statements so that we can't accidentally go diagonally if we pressed up and right on the same exact frame. We send in the correct `SDL_SCANCODE` and our keyboard data to our `KeyPressed()` function.

```cpp
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

We then check if either our x or y position should change. If yes, then we start fetching relevant data, like what the new cell position would be using `stepInto_x/y`. We then check if we have an entity already standing on the new cell with `stepInto_entity`. We also get the type of tile we're aiming for with `stepInto_tile_id`. We do this by using the helper functions we've added to our `LevelData` struct.

Then if no entity was found in the new position and the type of tile we're moving into was `GROUND` and not `WALL` then we update the x and y value of the specified entity.

Lastly, after having looped over all entities, we store the keyboard state in our `keys_previous`.

With this our player entity can move around the level using the arrow keys!
