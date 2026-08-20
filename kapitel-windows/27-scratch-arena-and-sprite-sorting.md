# 27 Scratch Arena and Sprite Sorting

You might have noticed an issue where the order entities are being drawn to the screen is sometimes wrong. With a lower entity being drawn behind an entity above it.
We are going to make a copy of our `EntityBuffer` and sort it. This in regular C++ would require us to create a new array then free it. If we don't free it we are causing a stackoverflow due to us having assigned memory that we never allow our computer to recapture and reuse. We'll fix this need to create->free all together by using a scratch arena
a scratch arena is a memory arena that we allocate then reset during each tick. Meaning that all memory in it is freed in bulk instead of individually.
creating the scratch arena is as simple as creating a subarena from `arena_main` then calling `Reset()` at the beginning of our game-loop.

```cpp
// gameState.h
struct GameData {
  // other variables hidden for clarity
  Memory::Arena* arena_scratch;
};
```

```cpp
// main.cpp
gameData->arena_main = arena_main;
// other code hidden for clarity
gameData->arena_scratch = Memory::CreateSubArena(arena_main, KILOBYTES(256));
```

then in our `while(running)` game loop we reset the arena.

```cpp
// main.cpp
while(running) {
  DLL_CheckStatus(&dll);
  Reset(gameData->arena_scratch);
  CalculateDeltaTime(&dt);
```

I also think it's time to simplify our `Memory::Allocate` with a handy macro.
Inside `arena.h` we'll add the following code

```cpp
// arena.h
#define ALLOC(arena, type) (type*)Memory::Allocate((arena), sizeof(type));
#define ALLOC_ARRAY(arena, type, count) (type*)Memory::Allocate((arena), sizeof(type) * count);
```

This will take `ALLOC(arena, type)` when written in our codebase and replace it with the code that follows it. We'll use `ALLOC` for single items and `ALLOC_ARENA` for when we want to allocate an array.
Inside `main.cpp` at all places where we call `Allocate()` we can now use our simplified macro.

```cpp
// main.cpp
// old
GameData* gameData = (GameData*)Memory::Allocate(arena_main, sizeof(GameData));
// new
GameData* gameData = ALLOC(arena_main, GameData);

// and for example
// old
gameData->fps_buffer = (float*)Memory::Allocate(arena_main, sizeof(float) * gameData->fps_buffer_count);
// new
gameData->fps_buffer = ALLOC_ARRAY(arena_main, float, gameData->fps_buffer_count);
```

This makes the code easier to read and reduces the amount of mindless typing we have to do each time we want to allocate to an arena. I've gone ahead and substituted all call sites for `Memory::Allocate` with this macro. You're free to do the same. But make sure to compile your program afterwards to ensure you didn't break something my mistyping.
Now we can refactor our `RenderEntities` to both fix a problem we had with accidentally copying data and to introduce our scratch arena to help with sorting. Previously we stored `levelData lvl` as the actual struct and not a pointer `LevelData* lvl` . This meant that we copied over the content each time, which will contribute to a potential stack overflow . We also made the same mistake when fetching the specific entity with `Entity entity = lvl.entityBuffer[i];` this should also have been a pointer instead.
With this in mind, lets look at the updated `RenderEntities`

```cpp
// levelRenderer.cpp
void RenderEntities(GameData* data, SDL_Renderer* renderer) {
  LevelData* lvl = &data->levels[data->currentLevelIndex];
  Entity** SortedEntities = ALLOC_ARRAY(data->arena_scratch, Entity*, lvl->entityCount);
  for (int i = 0; i < lvl->entityCount; i++) {
    SortedEntities[i] = &lvl->entityBuffer[i];
  }
  std::sort(SortedEntities, SortedEntities + lvl->entityCount, IsEntityBelowOtherEntity);
  for (int i = 0; i < lvl->entityCount; i++) {
    Entity* entity = SortedEntities[i];
    if(entity->id == ID::NONE) {
      continue;
    }
    Sprite* sprite = GetSprite_FromEntityState(entity, data->spriteBuffer);
    if(HasBehaviour(entity, Behaviour::IS_PETRIFIED)) {
      sprite = GetSpriteFromID(ID::ROCK, data->spriteBuffer);
    }
    float x_animated = std::lerp(entity->x_prev, entity->x, entity->progress_01);
    float y_animated = std::lerp(entity->y_prev, entity->y, entity->progress_01);
    float dropshadow_y = y_animated;
    if(HasBehaviour(entity, Behaviour::JUMPS) && !HasBehaviour(entity, Behaviour::IS_PUSHING)) {
      y_animated -= 0.5 * sinf(entity->progress_01 * 3.14);
    }
    Sprite* dropshadow = &data->spriteBuffer[(int)SPRITE_ID::Dropshadow];
    RenderEntity_OnTile(dropshadow, lvl, renderer, &data->camera, x_animated, dropshadow_y, 1, 0.4, false);
    RenderEntity_OnTile(sprite, lvl, renderer, &data->camera, x_animated, y_animated, 1, 1, entity->facing == Direction::RIGHT);
  }
}
```

We will be creating a new type of variable a pointer to a pointer. A bit strange, but all it is is a pointer that points to a place in memory where another pointer exists. We are going to be sorting pointers and to sort pointers we need an ordered list that points to them that we can sort.
The original pointers are layed out sequentially in our memory, but the correct draw order is not the same order as they are in memory. This is why another array of pointers exist where each pointer-pointer points at a specific entry in the original array. Allowing for a remapping

```
// our original entity pointers in memory
1-2-3-4-5
// our Entity** pointer-pointers in memory
1-2-3-4-5
but the `1` pointer-pointer "points" to original entity pointer `4` like so
(1)4-(2)2-(3)1-(4)5-(5)3
```

this allows us to draw the entity with the lowest y-value (after sorting) first even though it was the fourth entity in the original memory block.
`std::sort` comes from `#include <algorithm>` . This is a standard library in C++ that give us a handy way of sorting a known array.
`std::sort` accepts 3 parameters 1) the first entry in the array 2) the last entry in the array 3) the way we want to sort them
We're creating a small function inside `levelRenderer.cpp` that we pass as an argument to the `std::sort`

```cpp
// levelrenderer.cpp
bool IsEntityBelowOtherEntity(Entity* a, Entity* b) {
  return a->y < b->y;
}
```

note, this has to be placed above our `RenderEntities()` as it is not defined in our .h file.
in our `std::sort` the second parameter `SortedEntities + lvl->entityCount` takes the known `Entity**` then moves down our memory block a number of `Entity*` long steps equal to `entityCount` . To arrive at the last element in the array.
We pass `IsEntityBelowOtherEntity` as the function itself, that's why we don't add `()` and parameters. We're not calling the function we're telling `sort` to call and use it. The function compares two `Entity*` and because this is what the array points to our compiler knows how to work with this.
With this we've added our scratch arena and added draw order to our entities!