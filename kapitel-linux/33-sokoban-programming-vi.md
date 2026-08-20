# 33 Sokoban Programming VI

Before we add goal squares to our project we will be needing code that checks a new layer of our .TMJ file. Currently we have 2 places where we loop over every layer in our `auto result` from `CreateLevel()` and `CreateEntities()` . By adding a third place where we need to type this same for-loop structure we've hit a good benchmark for a helper function.

```cpp
// levels.h
#include "Parsers/json.hpp"

namespace AssetManagement {
  std::vector<uint16_t> GetCellDataFromJsonLayer(nlohmann::json& parsedJson, const char* layerName, bool* wasFound);
  int GetFirstNonZeroCell(std::vector<uint16_t>* list);
}
```

So we'll create two helper functions inside our `AssetManagement` namespace . These will be created then substituted in the two currently places where we currently duplicate our code.

```cpp
// levels.cpp
namespace AssetManagement {
  std::vector<uint16_t> GetCellDataFromJsonLayer(nlohmann::json& parsedJson, const char* layerName, bool* wasFound) {
    std::vector<uint16_t> result;
    *wasFound = false;
    for (const auto& layer : parsedJson["layers"]) {
      if (layer["name"] == layerName) {
        result = layer["data"].get<vector<uint16_t>>();
        *wasFound = true;
        break;
      }
    }
    return result;
  }
}

int GetFirstNonZeroCell(std::vector<uint16_t> *list) {
  for (int id : *list) {
    if(id != 0) {
      return id;
    }
  }
  assert(false);
  return -1;
}
```

The code is lifted almost entirely from `levels.cpp` but now we pass in a `bool` by pointer to store whether or not we found what we were looking for. We pass `parsedJson` with a `&` instead of a `*` . When working with our json object doing this lets us keep the more familiar `parsedJson["layers"]` syntax rather than the noisier `(*parsedJson)["layers"]` that unfortunately would be required when using an explicit pointer. the `&` forces the variable to exist at the callsite and we pass it just by name. meaning that we don't know if its a copy or a reference before we look at the function we're using. I don't like this type of coding for our own code, but as we're working with this json parser and it has this c++ structure we're sorta forced into doing the same if we want to have a relatively clean code layout.
we also have a few callsites where we do a convertion from 1D to 2D by using divided by and modulo operators. This is not the simplest code to remember and a small helper function would be very beneficial.

```cpp
// common.h
inline void Expand1DTo2D(int flatIndex, int width, int* x, int* y) {
  *x = flatIndex % width;
  *y = flatIndex / width;
}

inline void Expand1DTo2D(int flatIndex, int width, float* x, float* y) {
  *x = (float)(flatIndex % width);
  *y = (float)(flatIndex / width);
}
```

This does the same work but we can just pass the required parameters and then it sets x and y to the correct values. As these are pointers the value will be updated for the variable living at the callsite.
now we can simplify `CreateLevel()` and `CreateEntities()`

```cpp
// levels.cpp
void CreateLevel(Arena* arena, LevelData* level, Tileset* tileset, const char* level_name) {
  fstream stream(level_name);
  auto result = json::parse(stream);
  bool found = false;
  vector<uint16_t> levelData = AssetManagement::GetCellDataFromJsonLayer(result, "level", &found);
  assert(found);
  int first_non_zero_id = AssetManagement::GetFirstNonZeroCell(&levelData);
  int id_offset = Get_Tileset_ID_Offset_From_Tilemap(first_non_zero_id, result);
  level->w = result["width"].get<int>();
  level->h = result["height"].get<int>();
  level->level_path = level_name;
  level->tileset = tileset;
  level->cells = ALLOC_ARRAY(arena, uint16_t, level->w * level->h);
  for (int i = 0; i < level->w * level->h; i++) {
    int local_id = levelData[i] - id_offset;
    if(local_id < 0) {
      local_id = 0;
    }
    level->cells[i] = local_id;
  }
}
```

It's the same function in all regards but we now use our helper functions to reduce to volume of code.

```cpp
// levels.cpp
void CreateEntities(LevelData* lvl_data, Arena* arena) {
  Reset(arena);
  lvl_data->entityCount = 0;
  lvl_data->entityBuffer = (Entity*)Memory::Allocate(arena, sizeof(Entity) * 256);
  fstream stream(lvl_data->level_path);
  auto result = json::parse(stream);
  bool found = false;
  vector<uint16_t> entities = AssetManagement::GetCellDataFromJsonLayer(result, "entities", &found);
  if(!found) {
    return;
  }
  for (int i = 0; i < lvl_data->w * lvl_data->h; i++) {
    if(entities[i] == 0) {
      continue;
    }
    uint16_t entity_id = GetLocalTileID(entities[i], result);
    int x;
    int y;
    Expand1DTo2D(i, lvl_data->w, &x, &y);
    AddEntity((ENTITY_ID)entity_id, x, y, lvl_data);
  }
}
```

The same goes for our `CreateEntities()` function.
Then in `RenderSprite_World()` we use our helper function as well - not because it saves a lot of space, but it feels like a consistent practice.

```cpp
// rendering.cpp
// Expand1DTo2D(frame, sprite->sprite_count_x, &tilesetRect.x, &tilesetRect.y);
tilesetRect.x *= width;
tilesetRect.y *= height;
```

in the supplied assets `chapter 34 assets.zip` you'll find a new .tmj file called `testing_goal.tmj` . This level has a new layer inside Tiled called `goals` . On this layer we exclusively place a new goal tile . This sits ontop of the rendered level and in our game the victory condition for a level will be that each goal has a player entity standing on it. We place this not in our `level` layer as we want the graphics to sit ontop of our level. And we don't place it in our `entity` layer as we want to have both a goal and an entity being able to overlap. We could label this layer `markers` and maybe have more than one thing in it. But as we live by the practice of just programming what we need and refactoring later we'll just have our goal tile in this layer.
We want to store these goals in a separate array inside our `LevelData`

```cpp
// levels.h
struct Goal {
  int x;
  int y;
  float blink_timer;
};

struct LevelData {
  int w;
  int h;
  uint16_t* cells;
  Goal* goals;
  int goalCount;
  const char* level_path;
  Entity* entityBuffer;
  int entityCount;
  const Tileset* tileset;
};
```

Right now our `Goal` struct is just a x and y position and a timer. But it's still a valuable struct that might expand in the future. Another solution would be to have a `Position` struct and for all callsites, structs and functions where we currently pass x and y separately we instead pass in a `Position` . But this has little value currently so lets hold off. `blink_timer` will increase by `deltaTime` each frame for as long as an entity is standing on the goal square. Otherwise we'll reset it to 0.
With our helper functions we'll add more logic to the end of `CreateLevel()` to allocate our goals array.

```cpp
// levels.cpp
// at the end of the CreateLevel() function
vector<uint16_t> onLevel = AssetManagement::GetCellDataFromJsonLayer(result, "on_level", &found);
if(found) {
  int firstNonZero = AssetManagement::GetFirstNonZeroCell(&onLevel);
  id_offset = Get_Tileset_ID_Offset_From_Tilemap(firstNonZero, result);
  level->goalCount = 0;
  for (int i = 0; i < level->w * level->h; i++) {
    int local_id = onLevel[i] - id_offset + 1;
    if(local_id > 0) {
      level->goalCount++;
    }
  }
}
level->goals = ALLOC_ARRAY(arena, Goal, level->goalCount);
int index = 0;
for (int i = 0; i < level->w * level->h; i++) {
  int local_id = onLevel[i] - id_offset + 1;
  if(local_id < 0) {
    local_id = 0;
  }
  if(local_id != 0) {
    int x;
    int y;
    Expand1DTo2D(i, level->w, &x, &y);
    level->goals[index].x = x;
    level->goals[index].y = y;
    index++;
  }
}
```

we find how many goals are on the layer by finding all non-zero cells. This means that we could in theory add any tile to the grid and it would get converted to a goal. This is not ideal but we can live with it for now. If we ever need more markers we can think about refactoring. With only one single tile in the tileset used for our goal we need to add +1 to the `id_offset` as our tilemap doesn't start with an empty zero-indexed tile. So in this example .tmj the only non-zero cell in our goals layer has an id of 90 and the `firstgid` is also 90 . Meaning that if we remove it outright those cells would turn into 90 - 90 = 0 and just show up as all empty cells do. This is a bit messy but with this being the only callsite we can afford it.
We call `Expand1DTo2D()` to get the x and y coordinates of the goal cell before assigning it to the relevant `level->goals[index]` . Note how we only advance `index` after we've found a goal cell in the level grid.
Now we can create our `SPRITE_ID` for our goal as well as making sure we render them during `RenderLevel()`

```cpp
// spriteLibrary.h
enum class SPRITE_ID {
  // other hidden for brevity
  Goal
};
```

```cpp
// spriteLibrary.cpp
static const SpriteDataEntry all_sprite_data[] = {
  {SPRITE_ID::Goal, "assets/sprites/goal.png", 8, 8, 8, 1}
};
```

As we can see, the goal sprite is a spritesheet with 8 frames in total, layed out in a single row. It also has it's pivot in the center of its 16x16 frame. You can inspect the `SpriteDataEntry` struct to more clearly see which number corresponds to which variable.
At the end of `RenderLevel()` in `levelRenderer` we can make sure we render all of our goals

```cpp
// levelRenderer.cpp
for(int i = 0; i < level->goalCount; i++) {
  Goal goal = level->goals[i];
  Sprite* sprite = GetSprite(SPRITE_ID::Goal, gameData->spriteBuffer);
  int frame = (int)(goal.blink_timer / 0.2) % (sprite->sprite_count_x * sprite->sprite_count_y);
  RenderSprite_OnTile({frame, sprite}, level, renderer, &gameData->camera, goal.x, goal.y);
}
```

We fetch our goal sprite and select the appropriate frame by taking `blink_timer` and keeping it within the spritesheet bounds using `%` then we divide it by `0.2` , making to so it changes frame every `0.2` seconds.
`blink_timer` won't update on its own, inside our `UpdateGame()` we have to loop over all goals

```cpp
// game.cpp
for (int i = 0; i < level->goalCount; i++) {
  Entity* entity = GetEntity(level, level->goals[i].x, level->goals[i].y);
  if(entity != nullptr && !IsActing(entity)) {
    level->goals[i].blink_timer += dt;
  }
  else {
    level->goals[i].blink_timer = 0;
  }
}
```

This resets our goal back to frame 0 as soon as there is no entity on it. and we only increment `blink_timer` if an entity existed and it has finished whatever action it might have been taking. For example, walking onto the goal square.
Lastly, lets make sure we actually play the new level

```cpp
// game.cpp
void InitializeGame(Gameplay* gameplay, Arena* arena_levels, Tileset* tilesetBuffer) {
  assert(gameplay->initialized == false);
  gameplay->currentLevelIndex = 0;
  CreateLevel(arena_levels, &gameplay->levels[0], &tilesetBuffer[(int)TILESETS::Dungeon], "assets/levels/testing_goal.tmj");
  gameplay->initialized = true;
}
```

Before moving forward, we have to fix a very very small issue. Our `CalculateDeltaTime()` function in `main.cpp` should be the first code that runs each while-loop

```cpp
// main.cpp
while(gameData->running) {
  CalculateDeltaTime(&dt, dt_scaler);
  // the rest of the while loop
}
```

otherwise our fps will be off by a very small fraction as `DLL_CheckStatus()` and `Reset()` take a small amount of time to run and were previously not "seen" by the deltatime function.
Now we can see our goals rendered and they pulse if we stand on them. The next step is actually changing level when all entities are standing on a goal spot.
We'll look over all goals and if we have met the condition for each we'll advance our `CurrentLevelIndex` and call `StartLevel()`

```cpp
// game.cpp
// inside UpdateGame()
if(level->goalCount > 0) {
  int goals_reached = 0;
  for (int i = 0; i < level->goalCount; i++) {
    Goal goal = level->goals[i];
    Entity* entity = GetEntity(level, goal.x, goal.y);
    if(entity == nullptr) {
      break;
    }
    else if(HasBehaviour(entity, Behaviour::IS_PLAYER)) {
      goals_reached++;
    }
  }
  if(goals_reached == level->goalCount) {
    gameplay->currentLevelIndex++;
    StartLevel(gameplay, arena_commands, arena_entities);
    return;
  }
}
```

We only allow for the level to actually change if we found a goal. Otherwise the level would just blink past. This will create a level that is unwinnable, but if we want later we can assert that `goalCount` is atleast a 1 . Note how we call `return` after `StartLevel()` as we don't want anything in `UpdateGame()` to be called this frame if we have changed level.
We loop over each `Goal` then we try and find a relevant entity ontop of it. Finally if we found an equal amount of entities to the `goalCount` we can progress `currentLevelIndex` and call `StartLevel()` .
Because `StartLevel()` requires us to pass along `arena_commands` and `arena_entities` we need to include these as parameters in `UpdateGame()`

```cpp
// game.cpp
void UpdateGame(Gameplay* gameplay, Input* input, Arena* arena_scratch, Arena* arena_commands, Arena* arena_entities, const float dt) {
  // ...
}
```

as well as pass these parameters in at the callsite in `Update()`

```cpp
// game.cpp
case SCENE_TYPES::GAME:
  UpdateGame(gameplay, &data->input, data->arena_scratch, data->arena_commands, data->arena_entities, dt);
  break;
```

We also need to cleanup our `CommandBuffer` between levels

```cpp
// command.h
void ResetCommandBuffer(CommandBuffer* buffer);
```

```cpp
// command.cpp
void ResetCommandBuffer(CommandBuffer* buffer) {
  buffer->index = 0;
  buffer->head = 0;
  buffer->timestamp = 0;
}
```

we then call this new function at the top of `StartLevel()`

```cpp
// game.cpp
void StartLevel(Gameplay* gameplay, Arena* arena_commands, Arena* arena_entities) {
  ResetCommandBuffer(gameplay->commandBuffer);
  Reset(arena_commands);
  CreateEntities(&gameplay->levels[gameplay->currentLevelIndex], arena_entities);
  gameplay->activePlayerIndex = 0;
}
```

we make sure to call `Reset` on our `arena_commands` to purge that memory block from containing stale data. Now it's all neatly zeroed-out.
The last step is to change what levels we have in the game. Included in the same assets .ZIP that you unpacked earlier are `level_01.tsj` and `level_02.tsj` .
Let's make those the levels we initialize

```cpp
// game.cpp
void InitializeGame(Gameplay* gameplay, Arena* arena_levels, Tileset* tilesetBuffer) {
  assert(gameplay->initialized == false);
  gameplay->currentLevelIndex = 0;
  CreateLevel(arena_levels, &gameplay->levels[0], &tilesetBuffer[(int)TILESETS::Dungeon], "assets/levels/level_01.tmj");
  CreateLevel(arena_levels, &gameplay->levels[1], &tilesetBuffer[(int)TILESETS::Dungeon], "assets/levels/level_02.tmj");
  gameplay->initialized = true;
}
```

With this we can now start the game, playing `level_01` then as soon as our character steps on the goal space we instantly load `level_02` . We be beat that level we crash the game as we don't currently handle `currentLevelIndex` going beyond the amount of levels we've added. We'll refactor this in an upcoming chapter.
And as an added bonus we'll quickly add R to reset the current level

```cpp
// game.cpp
void UpdateGame(Gameplay* gameplay, Input* input, Arena* arena_scratch, Arena* arena_commands, Arena* arena_entities, const float dt) {
  if(KeyPressed(input, SDL_SCANCODE_R)) {
    StartLevel(gameplay, arena_commands, arena_entities);
    return;
  }
  // other code hidden
}
```
