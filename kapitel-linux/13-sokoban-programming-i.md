> **Linux:** This chapter is adapted for Linux.

# Sokoban Programming I

We don't yet have for example architecture for working with sound implemented, but we'll hold off on that for a moment. Focusing instead on making some progress on game logic.

It's time we start implementing some gameplay logic. In this course we will be making a Sokoban style game. This is a grid-based game where you push blocks onto target cells.

To make a Sokoban game we need to:

1. Have a grid-based world that has floor and walls
2. Have entities on that grid that can move and be interacted with
3. Load a level and populate it with the relevant entities

I will be creating three .PNG files: `ground.png`, `player.png` and `wall.png` — all are 32x32px squares. We'll add these to our `assets/sprites` folder.

We will be using a software called **Tiled** to create our levels. Download Tiled from tiled.com.

Inside Tiled we'll create a new tileset importing our three PNGs. Then we create two layers: `level` and `entities`. We can then create a map and using our tileset we can draw our level. Once we are happy with our test level we can export it from File > Export As, give it a name and export it as a JSON file that will have the file extension of `.tmj`.

Opening our exported `.tmj` file inside your editor (e.g. nvim) we can look at the different fields. The json element `layers` has two sub-elements, each with a couple of fields — `data` and `id` are the most important to consider at the moment.

> **Linux:** Use your editor (e.g. nvim) to open `.tmj` files instead of `subl` on Windows.

We need a JSON parser. A very good JSON parser comes from nlohmann and is a single .h file. Download the `json.hpp` file from https://github.com/nlohmann/json. Place it in `include/Parsers/`.

We'll create a struct inside a new .h file `levels.h`:

```cpp
struct LevelData{
  int w;
  int h;
  uint8_t* cells;
  const char* level_path;
  Entity* entityBuffer;
  int entityCount;
  uint8_t GetCellID(int x, int y){
    return cells[y * w + x];
  }
};
```

Lets look at our `Entity` struct next, kept inside a newly created `entity.h`:

```cpp
#pragma once
#include <cstdint>
struct Entity{
  uint8_t id;
  int x;
  int y;
};
```

Back in our `levels.h`:

```cpp
#pragma once
#include "arena.h"
#include "entity.h"
#include <cstdint>
using namespace Memory;
struct LevelData{
  // ...
};
void CreateLevel(Arena* arena, LevelData* level, const char* level_name);
void CreateEntities(LevelData* lvl_data, Arena* arena);
```

Inside a newly created `levels.cpp`:

```cpp
#include <cstdint>
#include <fstream>
#include <vector>
#include "levels.h"
#include "arena.h"
#include "Parsers/json.hpp"
#include "entity.h"
using namespace std;
```

```cpp
// levels.cpp
const int LEVEL_INDEX = 0;
const int ENTITIES_INDEX = 1;
```

```cpp
//levels.cpp
void CreateLevel(Arena* arena, LevelData* level, const char* level_name){
  fstream stream(level_name);
  auto jsonResult = nlohmann::json::parse(stream);
  vector dataField = jsonResult["layers"][LEVEL_INDEX]["data"].get<vector<uint8_t>>();
  level->w = jsonResult["width"].get<int>();
  level->h = jsonResult["height"].get<int>();
  level->level_path = level_name;
  size_t size_of_cells = sizeof(uint8_t) * level->w * level->h;
  level->cells = (uint8_t*)Memory::Allocate(arena, size_of_cells);
  for (int i = 0; i < level->w * level->h; i++) {
    level->cells[i] = dataField[i];
  }
}
```

Inside our `Arena.h` we will add a new function `CreateSubArena()`:

```cpp
Arena* CreateSubArena(Arena* parent_arena, size_t size);
```

And inside our `arena.cpp` we create the function:

```cpp
Memory::Arena* Memory::CreateSubArena(Arena* parent_arena, size_t size){
  Memory::Arena* sub_arena = (Memory::Arena*)Allocate(parent_arena, sizeof(Memory::Arena));
  void* memory_start = Allocate(parent_arena, size);
  Memory::Initialize(sub_arena, memory_start, size);
  return sub_arena;
}
```

With this change we can simplify our arena creation inside our `main()` function:

```cpp
Memory::Arena* arena_main = new Memory::Arena();
Memory::Initialize(arena_main, game_memory, GAME_MEMORY_ALLOWANCE);
GameData* gameData = (GameData*)Memory::Allocate(arena_main, sizeof(GameData));
size_t IMAGE_ARENA_SIZE = sizeof(Image) * 100;
gameData->arena_images = Memory::CreateSubArena(arena_main, IMAGE_ARENA_SIZE);
gameData->arena_levels = Memory::CreateSubArena(arena_main, MEGABYTES(3));
gameData->arena_entities = Memory::CreateSubArena(gameData->arena_levels, MEGABYTES(1));
```

Our updated `GameData` struct:

```cpp
struct GameData {
  Image* fallback;
  Image* wall;
  Image* ground;
  Image* player;
  Memory::Arena* arena_levels;
  Memory::Arena* arena_entities;
  Memory::Arena* arena_images;
  LevelData* levels;
  int levelCount;
  int currentLevel;
};
```

We will create a helper function inside `LevelData` to help us fetch an `Entity`:

```cpp
Entity* GetEntity(int x, int y){
  for (int i = 0; i < entityCount; i++) {
    if(entityBuffer[i].x == x && entityBuffer[i].y == y){
      return &entityBuffer[i];
    }
  }
  return nullptr;
}
```

We will expand `common.h`:

```cpp
const int SCREEN_WIDTH = 650;
const int SCREEN_HEIGHT = 400;
const int UPSCALE_FACTOR = 2;
const int CELL_SIZE_PX = 32 * UPSCALE_FACTOR;
```

Inside `void SDL_Setup()` in our `main.cpp`:

```cpp
window = SDL_CreateWindow("pilot", SCREEN_WIDTH, SCREEN_HEIGHT, 0);
```

Inside `main.cpp` we will update our `typedef`:

```cpp
typedef void (*Function_Initialize) (GameData* data, SDL_Renderer* renderer);
```

In `game.h`:

```cpp
extern "C" {
  void Initialize(GameData* data, SDL_Renderer* renderer);
  // ...
}
```

In `game.cpp`:

```cpp
void Initialize(GameData* data, SDL_Renderer* renderer){
  data->ground = AssetManagement::LoadSprite(data->arena_images, renderer, "ground.png");
  data->wall = AssetManagement::LoadSprite(data->arena_images, renderer, "wall.png");
  data->player = AssetManagement::LoadSprite(data->arena_images, renderer, "player.png");
  data->currentLevel = 0;
  CreateLevel(data->arena_levels, &data->levels[0], "assets/levels/testLevel.tmj");
  CreateEntities(&data->levels[data->currentLevel], data->arena_entities);
}
```

`CreateEntities` in `levels.cpp`:

```cpp
void CreateEntities(LevelData* lvl_data, Arena* arena){
  Reset(arena);
  lvl_data->entityCount = 0;
  fstream stream(lvl_data->level_path);
  auto result = nlohmann::json::parse(stream);
  auto entityData = result["layers"][ENTITIES_INDEX]["data"].get<vector<uint8_t>>();
  for (int i = 0; i < lvl_data->w * lvl_data->h; i++) {
    unsigned char entity_id = entityData[i];
    if(entity_id != 0){
      entityCount++;
    }
  }
  lvl_data->entityBuffer = (Entity*)Memory::Allocate(arena, sizeof(Entity) * lvl_data->entityCount);
  int index = 0;
  for (int i = 0; i < lvl_data->w * lvl_data->h; i++) {
    unsigned char entity_id = entityData[i];
    if(entity_id != 0){
      int x = i % lvl_data->w;
      int y = i / lvl_data->w;
      lvl_data->entityBuffer[index].id = entity_id;
      lvl_data->entityBuffer[index].x = x;
      lvl_data->entityBuffer[index].y = y;
      index += 1;
    }
  }
}
```

We will create `levelRenderer.h/.cpp`:

```cpp
#pragma once
#include "gameState.h"
void RenderLevel(GameData* gameData, SDL_Renderer* renderer);
void RenderEntities(GameData* gameData, SDL_Renderer* renderer);
```

```cpp
#include "levelRenderer.h"
#include "common.h"
#include "rendering.h"
#include <cstdint>

void RenderLevel(GameData* gameData, SDL_Renderer* renderer){
  LevelData lvl = gameData->levels[gameData->currentLevel];
  int board_width_px_half = lvl.w * CELL_SIZE_PX / 2;
  int board_height_px_half = lvl.h * CELL_SIZE_PX / 2;
  for(int x = 0; x < lvl.w; x++){
    for (int y = 0 ; y < lvl.h; y++) {
      uint8_t cellType = lvl.GetCellID(x, y);
      Image* sprite;
      switch(cellType){
        case 1:
          sprite = gameData->ground;
          break;
        case 2:
          sprite = gameData->wall;
          break;
        default:
          sprite = gameData->fallback;
          break;
      }
      float xPos = x * CELL_SIZE_PX;
      float yPos = y * CELL_SIZE_PX;
      xPos += SCREEN_WIDTH / 2.0;
      yPos += SCREEN_HEIGHT / 2.0;
      xPos -= board_width_px_half;
      yPos -= board_height_px_half;
      RenderSprite(sprite, renderer, xPos, yPos);
    }
  }
}
```

Update `RenderSprite` to use `UPSCALE_FACTOR`:

```cpp
void RenderSprite(Image* sprite, SDL_Renderer* renderer, int xPos, int yPos, float scale){
  rect.h = sprite->height * UPSCALE_FACTOR * scale;
  rect.w = sprite->width * UPSCALE_FACTOR * scale;
}
```

`RenderEntities`:

```cpp
//levelRenderer.cpp
void RenderEntities(GameData* data, SDL_Renderer* renderer){
  LevelData lvlData = data->levels[data->currentLevel];
  for(int i = 0; i < lvlData.entityCount; i++){
    Image* img;
    Entity entity = lvlData.entityBuffer[i];
    switch(entity.id){
      case 3:
        img = data->player;
        break;
      default:
        img = data->fallback;
        break;
    }
    int xPos = 0;
    int yPos = 0;
    xPos += SCREEN_WIDTH / 2.0;
    yPos += SCREEN_HEIGHT / 2.0;
    xPos -= data->levels[data->currentLevel].w * CELL_SIZE_PX / 2;
    yPos -= data->levels[data->currentLevel].h * CELL_SIZE_PX / 2;
    xPos += entity.x * CELL_SIZE_PX;
    yPos += entity.y * CELL_SIZE_PX;
    RenderSprite(img, renderer, xPos, yPos);
  }
}
```

`Draw()` in `game.cpp`:

```cpp
void Draw(GameData* data, SDL_Renderer* renderer){
  SDL_SetRenderDrawColor(renderer, 120, 70, 120, 255);
  SDL_RenderClear(renderer);
  RenderLevel(data, renderer);
  RenderEntities(data, renderer);
  SDL_RenderPresent(renderer);
}
```

With this we have our level rendering to the screen! In the next chapter we will start adding gameplay logic!
