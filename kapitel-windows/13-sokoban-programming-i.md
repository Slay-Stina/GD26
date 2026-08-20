# Sokoban Programming I

We don't yet have for example architecture for working with sound implemented, but we'll hold off on that for a moment. Focusing instead on making some progress on game logic.

It's time we start implementing some gameplay logic. In this course we will be making a Sokoban style game. This is a grid-based game where you push blocks onto target cells. But(!) that is just the basics. The Sokoban base formula has been turned inside out creating some absolutely fantastic puzzle games with rich mechanics and surprising gameplay. To name a few favorites:

- A Monsters Expedition
- A Good Snowman Is Hard To Build
- Steven Sausage Roll
- Baba Is You
- Void Stranger
- Skipping Stones to Lonely Homes

And in 2026 we will be getting the release of Order of the Sinking Star, poised to become the largest and probably most influential Sokoban game to date. Time will tell.

To make a Sokoban game we need to:

1. Have a grid-based world that has floor and walls
2. Have entities on that grid that can move and be interacted with
3. Load a level and populate it with the relevant entities

I will be creating three .PNG files: `ground.png`, `player.png` and `wall.png` — all are 32x32px squares. The ground will be brown, the player ice-blue and the walls grey. We'll add these to our `assets/sprites` folder.

We will be using a software called **Tiled** to create our levels. We could represent our levels in code directly, but this is not a smart way of handling level creation. Instead we'll download Tiled from tiled.com.

Inside Tiled we'll create a new tileset importing our three PNGs. Then we create two layers: `level` and `entities`. In `level` we'll place `ground` and `wall` tiles. And in `entities` we'll place our `player`.

We can then create a map and using our tileset we can draw our level. Once we are happy with our test level we can export it from File > Export As, give it a name and export it as a JSON file that will have the file extension of `.tmj`. A `.tmj` file is just a .json and is used in the exact same way. The name just indicates that it is from Tiled and I would bet that the file acronym stands for "TileMapJson".

Opening our exported `.tmj` file inside Sublime Text we can look at the different fields. The json element `layers` has two sub-elements, each with a couple of fields — `data` and `id` are the most important to consider at the moment. Now that we know the structure of our .json file we can parse it.

But Windows or SDL does not have a native JSON parser. Instead it is expected that we write our own or use one that someone else wrote. A very good JSON parser comes from nlohmann and is a single .h file that has all the relevant functionality all in the same single location.

We download the `json.hpp` file from https://github.com/nlohmann/json. I've placed this .hpp file in `include/Parsers/`.

> [!NOTE]
> .hpp is just the dogmatic C++ way of labeling a header file. The C-standard is .h. So just remember that when working with C++, .hpp and .h are interchangeable.

With our new parser added to our include folder we can begin working with it. First we will need a struct that can hold the level data once we have deserialized it from JSON. We'll create this struct inside a new .h file `levels.h`:

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

Ok, lets break down each part in order:

- `int w` and `int h` — the variables holding the width and height of our level. We get these from the `width` and `height` elements in our .tmj file that we exported from Tiled.
- `uint8_t* cells` — a pointer to the first cell stored in memory. This `LevelData` has a sequence of these laid out in memory one after the other. We use the array indicators `[]` along with a specified width and height index to find the cell we're looking for in the grid.
- `uint8_t` holds, like `char`, a 1 byte element. In this case numbers between 0 and 255. This allows us to have up to 255 unique cell types before we would need to expand to a `uint16_t` that can hold over 65 thousand unique numbers.
- `const char* level_path` — this will hold the name of the level, so we can reference it in other functions.
- `Entity* entityBuffer` — we will store a sequence of entities in memory. Using this pointer to the first entity along with `int entityCount` we can loop over each entity using the array indicator `[]`.
  > [!NOTE]
> We'll look at the `Entity` struct in just a moment. Until we have added all the relevant code our program won't compile for a while.

We've also created a helper function right inside of the struct. This means that when we're using the struct we can access this function like we could any of the variables stored within it. This function takes an `x` and `y` parameter and uses them to return what type of cell is stored at that grid position.

Our grid is laid out as a 2D grid:

```
00000
01110
01010
01110
00000
```

But in memory each cell is laid out in sequence: `000000111001010011100000`. We use a handy calculation to get the correct element calculated from its 2D representation into 1D:

```
y * w + x
```

`y * w` sets us at the correct row by skipping forward by the full width, then we walk down the row the specified `x` steps. Remember that multiplication is executed before addition.

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

That's it, just position and ID — neat!

Including `cstdint` is what gives us access to `uint8_t`. With this data-oriented programming methodology (as compared to object oriented) we operate on data through functions and model our data as only the relevant information, as compared to the operations being tied to the data itself.

Back in our `levels.h` we will look at the included headers and two function declarations:

```cpp
#pragma once
#include "arena.h"
#include "entity.h"
#include <cstdint>
using namespace Memory;
struct LevelData{
  // @MAX: Here's where we had our LevelData variables
  // I've just removed it to make the code block smaller.
};
void CreateLevel(Arena* arena, LevelData* level, const char* level_name);
void CreateEntities(LevelData* lvl_data, Arena* arena);
```

We include the `Memory` namespace so we can avoid typing `Memory::` before we can use `Arena`. This helps us reduce the length of the code in terms of raw characters to type.

`CreateLevel()` will grab a piece of memory within an arena to store the cells of the `LevelData` as well as assigning their IDs. This is done by parsing the .tmj JSON that we will fetch from disk by using the `level_name` added to the function.

> [!NOTE]
> we are doing no safeguarding at this stage, meaning that yes, our program will crash if the specified .tmj file is not found. We'll look at adding these safety measures to a bunch of functions in a later part of the course.

`CreateEntities()` will loop over the entities layer in our .tmj and when it finds an entity it will add it to the `entityBuffer` inside `LevelData` at the next slot. The result will be that the entities are packed tightly next to each other in memory. For the game we will be making, we will have zero trouble looping over this array as much as we want, meaning that even if we have to look over the entire array many many times per frame it will cost close to no time at all.

Inside a newly created `levels.cpp` we will be adding the contents of these two functions, as well as adding a third function found only inside the .cpp. This means that this function is not accessible by another class that includes the `levels.h` header. This is good practice when we want to break functionality into more discrete and reusable chunks without exposing these to the larger codebase. This concept is known as **encapsulation**.

```cpp
// levels.cpp
#include <cstdint>
#include <fstream>
#include <vector>
#include "levels.h"
#include "arena.h"
#include "Parsers/json.hpp"
#include "entity.h"
using namespace std;
```

We start by including our headers:

- `cstdint` to get access to `uint8_t`
- `fstream` to allow reading from disk
- `vector` as this is the format of the JSON Data we get back from parsing
- `levels.h` included to get access to `LevelData`
- `arena.h` included so we can pass in an arena pointer as a parameter
- `Parsers/json.hpp` the location of our nlohmann JSON parser downloaded earlier
- `entity.h` to get access to the `Entity` struct
- `using namespace std;` to skip specifying the `std::` namespace everywhere

```cpp
// levels.cpp
const int LEVEL_INDEX = 0;
const int ENTITIES_INDEX = 1;
```

We then create two constants, each referencing the Tiled layer of our `level` and `entities` layers found in our .tmj file. Making them `const` is a defensive pattern to make sure we don't accidentally change these numbers in our code.

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

We create an `fstream` object that we named `stream` — this can, during creation, be given a path to the file it should stream data from. We will pass it the path of our .tmj file later: `"assets/levels/testLevel.tmj"`.

The `nlohmann` namespace contains another layered namespace `json` inside it. Once we have gone into the first namespace, deeper into the second can we find the `parse()` function. We store the result of parsing the stream into a variable of type `auto`.

The `auto` variable is a variable that we rely on the compiler to know the correct type of. In actuality the `auto` variable in this case expands to `nlohmann::basic_json<>` but all we want to know is that this `jsonResult` holds all the elements found in our .tmj file. The actual variable type is less interesting.

We can take our `jsonResult` and handle it like a nested set of arrays, each accessible by name. In our .tmj file we first had a list called `layers` — inside that we had different layers, 0 for `level` and 1 for `entities`. This is hard to remember, that's why we stored these two numbers in our constants earlier. Once we are in the right layer we need to find the `data` block — this held the 2D grid representation of the level we drew in Tiled. This data is stored in the JSON as a `vector` of `uint8_t`.

A `vector` is like an array, but the size of this can be changed at runtime. Meaning that we can add and remove from this list at runtime without causing errors. The `.get<type>()` function is what takes the contents of our json and puts it inside the right type. Before we do this, the type is not known.

We take our pointer to our `LevelData` and store the width and height from our .tmj. These are stored in our json as whole numbers (integers) and therefore we should specify this in our `.get<type>()` function as `.get<int>()`.

We store the path to the level in our `level_path` variable.

Then it's time to allocate our grid cells into memory. We take the size of a `uint8_t` aka 1 byte (this is the same as a `char` — we could even use these interchangeably) and multiply it by the combined width and height of our level. Giving us the memory footprint of all the cells. We then allocate this memory chunk to our arena and in the process fetch a pointer to the first cell in our `level->cells` pointer.

It is named `cells` and not `cell` even though the pointer only points to one cell — we can, as we know the size of each cell and the fact that they are laid out sequentially in memory, access them with the array indicator `[]`. If the name was singular this would be confusing.

We then loop an amount of times equal to the total number of cells (width times height of the level) and assign each of the cells in `level` their correct ID, that we can get from the `vector` (aka our list of the same size and order) that we parsed from our json earlier and stored in `dataField`.

With this we have read the contents of the JSON file and stored relevant information in our `LevelData` pointer that we passed into the function.

Our level represents our static game world, but our player, boxes and other objects will be stored as our `Entity` struct.

We will partition our main arena further so we can have an arena that holds just this data, making clearing it a simple action. But making arenas inside arenas is not something new — we already know how to slice up our main arena further. But with each arena we will have to perform the same few lines of code each time. And when we find code that we write over and over again, we've found a clear candidate for a function.

Inside our `Arena.h` we will add a new function `CreateSubArena()` — this function accepts a parent arena and a size, then carves out that size of memory from the parent arena and stores it as a new arena.

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

First we allocate the size of the arena struct which are just the three variables that construct the arena. Not to be confused with the memory given to the arena to hold.

We then allocate the space of the new arena to the parent arena, committing that chunk of memory so it is not overwritten. Finally we `Initialize` the `sub_arena` telling it what chunk of memory it has access to and then we return it (as a pointer).

With this change we can simplify our arena creation inside our `main()` function inside `main.cpp`:

```cpp
Memory::Arena* arena_main = new Memory::Arena();
Memory::Initialize(arena_main, game_memory, GAME_MEMORY_ALLOWANCE);
GameData* gameData = (GameData*)Memory::Allocate(arena_main, sizeof(GameData));
size_t IMAGE_ARENA_SIZE = sizeof(Image) * 100;
gameData->arena_images = Memory::CreateSubArena(arena_main, IMAGE_ARENA_SIZE);
gameData->arena_levels = Memory::CreateSubArena(arena_main, MEGABYTES(3));
gameData->arena_entities = Memory::CreateSubArena(gameData->arena_levels, MEGABYTES(1));
```

With this we have taken our main arena and carved out specific arenas, meant to be responsible for specific parts of our game data. We will return to further create and subdivide these arenas later.

But in order for these arenas to be assignable we need to actually set up our `GameData` struct to hold the games data, and not our test data that we used previously:

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

We have our sprites, references to our arenas, a pointer to the first level and the amount of levels and the current level index. With the pointer to `LevelData` and `levelCount` we can allocate the levels into our `arena_levels` and use the array symbols `[]` to fetch the correct level.

> [!NOTE]
> We do the same thing inside `LevelData` with our `Entity* entityBuffer` and `int entityCount`.

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

We loop over all `Entity` structs by fetching them one at a time and comparing both their `x` and `y` to the parameters provided. If an entity is found we return a pointer to it using `&`. If no entity is found at the specified position we return `nullptr`.

We now need to update a few places inside our code and create a few new .h and .cpp files. These changes are necessary to render our levels and entities. We will start by expanding `common.h` adding a few variables to help us know where on the screen we should draw our tiles as well as how big they should be:

```cpp
const int SCREEN_WIDTH = 650;
const int SCREEN_HEIGHT = 400;
const int UPSCALE_FACTOR = 2;
const int CELL_SIZE_PX = 32 * UPSCALE_FACTOR;
```

We use `UPSCALE_FACTOR` to draw our tiles larger than they actually are, meaning that with a `UPSCALE_FACTOR` of 2 every 1x1 pixel is now a 2x2 pixel grid. This is necessary to actually see pixel art as our HD monitors would make them very very tiny otherwise.

`CELL_SIZE_PX` is the width (or height) of our tiles, adjusted for upscaling. This value is used to position the tiles next to each other. If we didn't account for `UPSCALE_FACTOR` then our tiles when drawn larger would overlap each other by 50%.

Inside `void SDL_Setup()` in our `main.cpp` we update our `SDL_CreateWindow()` to use `SCREEN_WIDTH` and `SCREEN_HEIGHT`:

```cpp
window = SDL_CreateWindow("pilot", SCREEN_WIDTH, SCREEN_HEIGHT, 0);
```

We will also be passing along our `SDL_Renderer` to our `Initialize()` function inside `game.h/.cpp` — though this change, as it is part of the connection between our .EXE and .DLL, requires a bit more work.

Inside `main.cpp` we will update our `typedef` to take a `SDL_Renderer*` as a second parameter:

```cpp
typedef void (*Function_Initialize) (GameData* data, SDL_Renderer* renderer);
```

This addition needs to be added to `game.h` as well:

```cpp
__declspec(dllexport) void Initialize(GameData* data, SDL_Renderer* renderer);
```

Then lets add the changes to `game.cpp`:

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

We load and store our `Image` structs for `ground`, `wall` and `player` in our `GameData`. We hardcode our `currentLevel` to start at 0. Then we create level 0 and afterwards we create our entities related to the `currentLevel` (which is also level 0).

Lets add the contents of `CreateEntities` to `levels.cpp`:

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

This function starts very similarly to `CreateLevel` but instead of providing a path to our .tmj file we retrieve the path we stored inside our `LevelData` struct as `level_path`. But before we do anything else we call `Memory::Reset()` on our `arena_entities` meaning that all entities that existed previously are instantly freed. We also reset our `entityCount` variable to acknowledge this. The first time we run this function we are already at 0, but if we ever ran it again we would need to make sure everything was reset to the default.

We then find the JSON data from the layer related to entities (rather than the layer for the level as we did previously).

We then loop over all cells that we retrieved and whenever that cell is not 0 (meaning we found an entity) we increment our `entityCount`.

Then after we have our block of memory allocated as an array of `Entity` structs we loop over the cells one more time, and when we find an entity (non-zero value) we find the x and y positions using some clever math that can produce a 2D point from a 1D array given that we know the width of the grid.

- `x = i % lvl_data->w` finds the x coordinate by using the modulo operator to remove the width of the grid from `i` as many times as it can. Meaning that if the width is 5 and `i` is 11 then we remove 5 then we remove 5 again. Leaving an x value of 1.
- `y = i / lvl_data->w` we then find the y coordinate by taking the value of `i` and dividing it by the width. 11/5 this has the result of 2.2 but since an `int` can't store decimal values those are discarded, giving us a value of 2.

This means that at `i == 11` we get `x = 1` and `y = 2` and with arrays starting at 0 in C++ we know that our entity at `i = 11` is here:

```
00000
00000
01000
00000
00000
```

And laid out in its 1D representation we get:

```
0000000000010000000000000
```

We then take our calculated x, y and the id we found and store those at the position `index` which starts at 0 and increases by 1 each time we find an entity. This index shifts the array one step forward, filling each array element with the corresponding info. Lastly we increment `index` so that, during the next non-zero entity slot found, we put the corresponding data into the next array element.

With `LevelData` and its `EntityBuffer` loaded from our JSON and with the correct data filled we are ready to start rendering our level and entities.

We will render first the level then the entities on top of it. We will create `levelRenderer.h/.cpp`:

```cpp
#pragma once
#include "gameState.h"
void RenderLevel(GameData* gameData, SDL_Renderer* renderer);
void RenderEntities(GameData* gameData, SDL_Renderer* renderer);
```

Then inside our .cpp we will add our `#includes` and write the functions:

```cpp
#include "levelRenderer.h"
#include "common.h"
#include "rendering.h"
#include <cstdint>
```

```cpp
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

Lets break down this function:

```cpp
LevelData lvl = gameData->levels[gameData->currentLevel];
```

We fetch the levels pointer then using our array indicator we fetch the level stored at the `currentLevel` position.

```cpp
board_width_px_half = lvl.w * CELL_SIZE_PX / 2;
```

Takes the `w` meaning the amount of cells, multiplies it by the size of a cell `CELL_SIZE_PX` (which accounts for `UPSCALE_FACTOR`), then divides that total width of all the cells together by 2 to get half of this width. We need this and the corresponding "half-height" to push the rendered tiles back half the board width/height so that the level stays centered on the screen. Without adjusting our positions by these values a level would, as it grew, move down and to the right.

We nest a for-loop inside another for-loop as this will allow us to loop over each row one at a time, and each column in turn. We store the current row and column in the `x` and `y` variables.

```cpp
uint8_t cellType = lvl.GetCellID(x, y);
```

Our helper function found inside our `LevelData` struct now gives us a simple way of finding the ID of the tile on the specified coordinate.

```cpp
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
```

With this we can assign the correct `Image` to our `sprite` based on the ID we got back. The `default` case inside our switch-statement will be automatically selected if none of the other cases match. Meaning that instead of the program crashing we render our fallback sprite instead, letting us know that something should be on that coordinate but we have not added it yet.

```cpp
float xPos = x * CELL_SIZE_PX;
float yPos = y * CELL_SIZE_PX;
xPos += SCREEN_WIDTH / 2.0;
yPos += SCREEN_HEIGHT / 2.0;
xPos -= board_width_px_half;
yPos -= board_height_px_half;
```

As the comments suggest, these operations on `xPos` and `yPos` calculate the correct position for the tile.

```cpp
RenderSprite(sprite, renderer, xPos, yPos);
```

With all the required data fetched we pass it as arguments to our `RenderSprite()` function inside `Rendering.cpp`. But we need to update it to actually use our `UPSCALE_FACTOR` — if we don't we will need to pass it as a fifth parameter each time we want to render something:

```cpp
void RenderSprite(Image* sprite, SDL_Renderer* renderer, int xPos, int yPos, float scale){
  rect.h = sprite->height * UPSCALE_FACTOR * scale;
  rect.w = sprite->width * UPSCALE_FACTOR * scale;
}
```

Now `rect.h` and `rect.w` scale automatically by `UPSCALE_FACTOR` each time as well as still allowing us to supply `scale` as a fifth parameter to have custom scaling.

And our `RenderEntities()` function is in most ways very similar to our `RenderLevel()`. But instead of looping over the entire grid we loop over all entities in our `entityBuffer` then fetch the correct `Image*` from its `id`. We then perform the same position adjustments as in `RenderLevel()` but I have not added the same `board_width_px_half` and instead opted to just write the equation where it is needed. This is something you can easily refactor to either go with the more readable name or the shorter function call.

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

All that is left now is to rewrite the contents of our `Draw()` function inside `game.cpp`:

```cpp
void Draw(GameData* data, SDL_Renderer* renderer){
  SDL_SetRenderDrawColor(renderer, 120, 70, 120, 255);
  SDL_RenderClear(renderer);
  RenderLevel(data, renderer);
  RenderEntities(data, renderer);
  SDL_RenderPresent(renderer);
}
```

We set our DrawColor to any color we like, then calling `SDL_RenderClear` we fill the entire backbuffer with that color. Afterwards we render our level tiles using `RenderLevel` and then our entities on top of that backbuffer. Finally we finish by calling `SDL_RenderPresent` which takes everything we've drawn to our backbuffer and blits it to the window so it can be rendered.

With this we have our level rendering to the screen! In the next chapter we will start adding gameplay logic!
