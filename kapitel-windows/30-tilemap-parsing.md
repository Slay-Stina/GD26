# 30 Tilemap parsing

Graphics is very much a non-trivial part of game development. We'll be doing quite extensive refactoring to our codebase to work with a less fragile and more expressive output from Tiled.
You'll find a copy of the chapters `assets` folder in the course material named `chapter 31 assets.zip` . Replace your `assets` folder with this new one.
Inside you'll find a new .tmj file as well as a new .tsj file stored in a new subdirectory called `tilesets` .
I've also included tiled project files `chapter 31.zip` . This is the entire Tiled project used to produce the contents of the assets folder.
The goal of this chapter is to stop referencing ground and wall tiles directly by individual `SPRITE_ID`s and instead we grab a large tileset with multiple tiles all layed out next to each other. Then we select the appropriate tile to use based on the `uint16_t` id of our `cells[]` in `LevelData*` .
Right now we only have a single tilset `Dungeon` but this architecture is made to simplify the addition of more tilesets and more worlds down the line.
Previously we checked against the ID of a tile to figure out if we were allowed to walk on it. We did this with the naive if-statement

```cpp
// example
if(cell_id == ID::GROUND) {
  // ...
}
```

We are removing both our `ID::GROUND` and `ID::WALL` from our code. then we'll change the name of `ID` to `ENTITY_ID` as we are no longer storing IDs for our terrain. Now they are exclusive for our Entities. This will touch a lot of our codebase. But at this point you should be familiar enough using Helix to search for, find and update these parts of the code once everything is set up in this chapter.
We can open Tiled project and inspect it. our `testing.tmx` is our level. It uses three tilesets to draw the level.

1. `automap rules` . This is a tileset from Tiled itself. It was used alongside a feature called `automapping` to help with level creation. The important thing to understand for us is that it is part of the project file.
2. `hell_of_a_time_dungeon_tileset` . Here we have all the tiles necessary to create our level. We have a few variations for walls but a lot of different ground tiles
3. `Entities` . This tileset has our `Medusa` , a blue box for `Demon` and our `Rock` . It should be noted that when we open `entities.tsx` we can find our three entities. In this chapter we will have a very brittle setup where the order of the entities in this .tsx file need to match the `ENTITY_ID` enum order. So if Medusa is the first `ENTITY_ID` then it also has to be the leftmost sprite in our `entities` tileset . We'll be creating a more robust connection between these at a later chapter.

Here we run across the fundamental increase in complexity as compared to what we did previously. We can no longer say that ID 5 is for example `Ground` as we have many many tiles that represent `Ground` . The same goes for walls. We need a more robust way of categorising these. We are also interested potentially in having more behaviour associated with a tile besides whether or not we can walk on it. Additionally we can not even know for sure if ID 5 is a ground tile or maybe a bush or a wall. This is because each tileset we add needs their own ids meaning that the first tileset starts from 0 then the next starts from the previous tilesets final id and goes from there.
We've decided to use Tiled in this project for two reasons

1. we want to work with JSON files
2. we want to show a pretty standard workflow for working with colleagues using other programs or file formats to produce content for our game. It would pretty simple to expand on our level editor to allow us to draw and create our levels right inside our game engine. But this would mean that we would need to take a whole detour into tools programming . This is the work of creating robust systems that anyone in the team can learn to use, demanding little to no programming knowledge. I mention this so your takeaway isnt that Tiled is some godsend software that we have no issues with.

Inside the project file `hell_of_a_time_dungeon_tileset.tsx` we have added what is called a custom property to each tile. This property labeled `walkable` is a bool that we set inside Tiled on a per tile basis to control if they should be walkable or not. When we export our `dungeon_tileset.tsj` file to our `assets` folder then this custom property will be stored inside alongside each `id` . We can reference this custom property when parsing our .tsj file in our game engine. the .tsj file is just a JSON file with a custom extension that Tiled has added. It's exactly the same as a normal JSON. The name is just for cataloguing.
We'll be setting up a `tilesetLibrary.h/.cpp` that will hook into our `AssetManagement` namespace and add a way for us to load each tileset

```cpp
// tilesetlibrary.h
#pragma once
#include "Parsers/json.hpp"

using namespace nlohmann;

namespace Memory {
  struct Arena;
}

enum class TILESETS {
  NONE = 0,
  Dungeon = 1,
  COUNT = 2
};

struct Tileset {
  TILESETS type;
  bool* walkableBuffer;
};

struct TilesetDataEntry {
  TILESETS type;
  const char* path;
};

uint16_t GetLocalTileID(uint16_t id_global, const json& tmj_result);
uint16_t Get_Tileset_ID_Offset_From_Tilemap(int id_limit, const json& tmj_result);

namespace AssetManagement {
  void LoadAllTilesets(Tileset* tilesetBuffer, Memory::Arena* arena_images);
  void LoadTileset(TilesetDataEntry* entry, Tileset* tilesetBuffer, Memory::Arena* arena_images);
}
```

We forward declare our `Arena` struct so that this .h file is allowed to add it as a parameter to our functions. We could also `#include "arena.h"` but doing it this way will help avoid circular dependencies. But as we currently don't have any of those you can decide if you find the forward declaration hard to understand. In that case just substitute it for our normal include.
we also specify `using namespace nlohmann` this is the namespace holding our Json parser by adding this `using` we dont have to write `nlohmann::` before being allowed to access the `json` struct.
We store our different tilesets in our `TILESETS` enum. Currently we just have `Dungeon` as well as some helpers values.
our `Tileset` struct is very basic. It knows its type then it stores an array of booleans. This array will live alongside our tileset's IDs to allow us to quickly and easily check if a tile is walkable.
`TilesetDataEntry` is a struct that we'll be manually filling out just as we did for `SpriteDataEntry` . We'll do this inside `tilesetLibrary.cpp` .
We also have two helper functions `GetLocalTileID()` and `Get_Tileset_ID_Offset_From_Tilemap()` (a bit of a mouthful...)
These are used to get a tiles ID/posiition within its own tileset, not caring about the fact that our .tmj might have had multiple tilesets used. This will help us find that the first tile in our tileset is a corner wall piece even if we normally can't know that based on the ID it was given inside our .tmj .
Then using our `AssetManagement` namespace we create two load functions. `LoadAllTilesets` will be calling `LoadTileset` once for each `TilesetDataEntry` that we've created inside `tilesetLibrary.cpp`

```cpp
// tilesetLibrary.cpp
#include "Parsers/json.hpp"
#include "tilesetLibrary.h"
#include "arena.h"
#include <cassert>
#include "fstream"

using namespace nlohmann;
using namespace std;

static const TilesetDataEntry all_tilesets_data[]{
  {TILESETS::Dungeon, "assets/tilesets/dungeon_tileset.tsj"}
};
```

here we simplify accessing `nlohmann` and `std` then we set up our `static` array of `TileSetDataEntry` . Right now there is just one. But its contents is the enum as well as a path to the .tsj file holding our tileset . Remember, our .tsj file has both our ids in the tilesets local 0-count range and our custom property called `walkable` added inside Tiled .

```cpp
// tilesetLibrary.cpp
uint16_t Get_Tileset_ID_Offset_From_Tilemap(int id_limit, const json& tmj_result) {
  int highest_tilemap_start_id = 0;
  for (const json& tileset : tmj_result["tilesets"]) {
    int first_id = tileset["firstgid"].get<int>();
    if(first_id <= id_limit && first_id > highest_tilemap_start_id) {
      highest_tilemap_start_id = first_id;
    }
  }
  return highest_tilemap_start_id;
}

uint16_t GetLocalTileID(uint16_t id_global, const json& tmj_result) {
  return id_global - Get_Tileset_ID_Offset_From_Tilemap(id_global, tmj_result);
}
```

Our helper functions are a bit hard to parse since we are doing a bunch of things related to parsing a JSON file that we don't normally do for the rest of our codebase. The first thing you need to know is that our .tmj file has an array inside it called `tilesets` and each of these tilesets has what is called a `firstgid` this is the id of the first tile in that tileset but it's based on the tilesets that came before it.

```cpp
// example
tileset_01 has 6 tiles and `firstgid` is 1
tileset_02 has 51 tiles and `firstgid` is 7
tileset_03 has 2 tiles and `firstgid` is 58
```

If the tilesets had been added in another order their `firstgid` values would also change.
`Get_Tileset_ID_Offset_From_Tilemap()` takes in a cell id as a parameter then finds the tileset with the highest `firstgid` that is still lower than the id we specified. This means that if we pass in id 44 then we find that `firstgid` for tileset_03 was too large and therefore the id doesn't belong to it. Then we have two tilesets left. One with `firstgid` of 1 and one with 7 . Both are lower than id (44) so we can safely select the highest one tileset_02 . This allows us to find out the `firstgid` of the tileset that the tile belonged to by just specifying its global id .
`GetLocalTileID` takes the `id_global` and subtracts the `firstgid` that its own tileset uses to get the id that the tile has inside its own tileset only.
inside `Get_Tileset_ID_Offset_From_Tilemap()` we use `const auto& tileset` it is very syntactically different from our normal code.
`const` means that we are not allowed to accidentally modify the values in `tileset` .

`auto` means that we let our compiler figure out the data type of our `tileset` . We do this because the `nlohmann::json` parser uses more complex data structures behind the scenes to help us work with our JSON file. So we'll let the compiler handle the mapping. `&` put after `auto` will make the `tileset` be a reference to the data from `tmj_result` instead of creating a whole new copy of the data. The size of the data behind the scenes is pretty large, so we wan't to avoid creating a copy of it when there is no need.
This is not my favorite type of programming, but this syntax style will be contained to the parsing of json files for our game engine. If we can swallow the fact that we don't have full knowledge of the underlying data type hidden by `auto` and that as long as we know how to fetch data from the json using `["name_of_data"]` and `.get<variableType>()` then we can move forward.
the strings `"tilesets"` and `"firstgid"` are the exact names that I found when opening the .tmj file in Sublime Text to inspect the data inside it.

```cpp
// tilesetLibrary.cpp
namespace AssetManagement {
  void LoadAllTilesets(Tileset* tilesetBuffer, Memory::Arena* arena_images) {
    for (TilesetDataEntry entry : all_tilesets_data) {
      LoadTileset(&entry, tilesetBuffer, arena_images);
    }
  }

  void LoadTileset(TilesetDataEntry* entry, Tileset* tilesetBuffer, Memory::Arena* arena_images) {
    assert(entry->type != TILESETS::COUNT);
    assert(entry->type != TILESETS::NONE);
    Tileset* tileset = &tilesetBuffer[(int)entry->type];
    tileset->type = entry->type;
    fstream stream(entry->path);
    auto jsonResult = json::parse(stream);
    int tile_count = jsonResult["tilecount"].get<int>();
    tileset->walkableBuffer = ALLOC_ARRAY(arena_images, bool, tile_count);
    auto& tiles = jsonResult["tiles"];
    for(const auto& tile : tiles) {
      int tile_id = tile["id"].get<int>();
      for(const auto& tile_property : tile["properties"]) {
        if(tile_property["name"] == "walkable") {
          tileset->walkableBuffer[tile_id] = tile_property["value"].get<bool>();
        }
      }
    }
  }
}
```

`LoadAllTilesets()` loops over `TileDataEntry` array and calls `LoadTileset` for each of them. it also passes `Tileset* tilesetBuffer` that we will add to an pass in from `GameData` in `gameState.h`
`LoadTileset` then tries and find the file found at `entry->path` and uses the `nlohmann` json parser to turn it from text into something we can use in code. We store the full parsed json data in `auto jsonResult` .
`tile_count` is fetched by taking our full json data and fetching the data held in `tilecount` . This is a number inside our .tsj file. Note. We are creating our tilesets from our .tsj file(s). Not from the actual level file which is our .tmj . The .tsj file is just our tiles and their custom properties exported from Tiled.
we allocate our `walkableBuffer` and make it as large as the amount of tiles in our tileset.
We then fetch the json array holding all our tiles by using `["tiles"]` . You can open our .tsj file in Sublime Text to learn what each array is called. A JSON file is so nice because it is human-readable.
We then use the same `const auto&` syntax to fetch each of the entries from the json array. We use `auto` everwhere when working with this parser so we can just ignore the underlying types.
Inside the for-loop we fetch the id of the tile then we loop over all properties stored on that tile. Currently we only store `walkable` but we can imagine having more attributes for a tile. When the property is `walkable` then we know we can read the value stored in that json array element to figure out if the tile is walkable or not. We assign the `walkableBuffer` array at that index to the parsed value . And as we know (because we've opened and inspected or .tsj file) the name was `walkable` the type was `bool` and the value was `true/false` . So using `.Get<bool>()` we on the value element we can get back `true` or `false` .
I will happily suggest spending extra time learning how to parse a JSON. But as this step only happens during the Initialization of our program means that we know immediatly if things have worked or not. And thankfully a LLM is a great help when figuring out how to parse a JSON like this. If you provide it the JSON file and what data you want to retrieve it will help you figure out how to get there.
So after `LoadTileset()` has run we have allocated a boolean array for the `walkable` parameter synced with the local id of each tile. We also assign the relevant `tileset*` pointer from `tilesetBuffer` to point to this data.
Lets look at `gameState.h` and add what we need to it

```cpp
// gameState.h
struct GameData {
  // other variables hidden for clarity
  Tileset* tilesetBuffer;
};
```

That's it.
We'll start using the width/height of a tile in tileset space, meaning the actual size in pixels (not the scaled up size). We'll modify `common.h` to hold both a `_RAW` size and a `_SCALED` size.

```cpp
// common.h
const int TILE_SIZE_PX_RAW = 16;
const int TILE_SIZE_PX_SCALED = TILE_SIZE_PX_RAW * UPSCALE_FACTOR;
```

I used `space+r` to rename `CELL_SIZE_PX` to `TILE_SIZE_PX_SCALED` . If I change the name using this renaming command it will automatically update throughout my codebase. I then add `TILE_SIZE_PX_RAW` and use it to calculate `_SCALED` .
In `entity.h` we're updating the `enum class ID` to `enum class ENTITY_ID` . if we use `space+r` then it will similarly update throughout the codebase. We're also removing `GROUND` and `WALL` and reordering our enum to match the layout of entities from our `entities.tmx` file. (we'll create a more robust solution for this matching later)

```cpp
// entity.h
enum class ENTITY_ID : uint8_t {
  MEDUSA = 0,
  DEMON = 1,
  ROCK = 2,
  SIREN = 3,
  GOLEM = 4,
};
```

We don't yet have the tiles in `entities.tmx` for `SIREN` and `GOLEM` . But we'll add these later so I'll keep them around.
With `ID::NONE/GROUND/WALL` removed we will get more than a few errors throughout our codebase as we previously had switch-cases for them. We're adding a `bool active` to our `Entity` struct to be used instead of our `ID == NONE` checks that we did earlier

```cpp
// entity.h
struct Entity {
  ENTITY_ID id;
  bool active;
  Direction facing;
  int strength;
  int x;
  int y;
  int x_prev;
  int y_prev;
  float progress_01;
  Behaviour behaviour;
};
```

in `entity.cpp` we used to assert that `ID` was not `NONE` . We will change this to look at `active` instad

```cpp
// entity.cpp
void InitializeBaseBehaviour(Entity* entity) {
  // assert(entity->id != ID::NONE); // old
  assert(entity->active);
  switch (entity->id) {
    // ...
  }
}
```

Now we need to make sure we actually load all (we just have one) tilesets from `game.cpp`

```cpp
// game.cpp
void Initialize(GameData* data, SDL_Window* window, SDL_Renderer* renderer) {
  DEV::Initialize(window, renderer);
  AssetManagement::LoadAllSprites(data->spriteBuffer, renderer);
  data->imGui_context = ImGui::GetCurrentContext();
  AssetManagement::LoadAllTilesets(data->tilesetBuffer, data->arena_images);
  SDL_Texture* blackfade = GetSprite(SPRITE_ID::black_1x1, data->spriteBuffer)->texture;
  SDL_SetTextureBlendMode(blackfade, SDL_BLENDMODE_BLEND);
  InitializeGame(&data->scenes.gameplay, data->arena_levels, data->tilesetBuffer);
  ChangeScene(data, SCENE_TYPES::TITLESCREEN);
}
```

We also update our `InitializeGame()` to take our `tilesetBuffer` as a parameter

```cpp
// game.cpp
void InitializeGame(Gameplay* gameplay, Arena* arena_levels, Tileset* tilesetBuffer) {
  assert(gameplay->initialized == false);
  gameplay->currentLevelIndex = 0;
  CreateLevel(arena_levels, &gameplay->levels[0], &tilesetBuffer[(int)TILESETS::Dungeon], "assets/levels/testing.tmj");
  gameplay->initialized = true;
}
```

As you can see, our `CreateLevel()` now takes a `Tileset*` pointer as a parameter. This is the tileset the level is using to draw its contents.
`CreateLevel()` and `CreateEntities()` in `level.cpp` has changed a lot to work with our new tilemap logic.

```cpp
// levels.cpp
void CreateLevel(Arena* arena, LevelData* level, Tileset* tileset, const char* level_name) {
  fstream stream(level_name);
  auto result = json::parse(stream);
  bool found = false;
  vector<uint16_t> levelData;
  for (const auto& layer : result["layers"]) {
    if (layer["name"] == "level") {
      levelData = layer["data"].get<vector<uint16_t>>();
      found = true;
      break;
    }
  }
  assert(found);
  int first_non_zero_id = 0;
  for (int id : levelData) {
    if(id != 0) {
      first_non_zero_id = id;
      break;
    }
  }
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

We still parse a json file loaded from the specified path. But once we have that we need to find the actual layer in Tiled that we have used to draw our levels. In Tiled we currently have 3 layers

- `blueprint`
- `level`
- `entities`

To make sure we fetch our data from the correct place we loop over each layer until we find one named "level". When we do we flip the `found` bool to true and collect all the cell ids stored in the "data" array.
Once again, these names are all just lifted from the JSON file. We open it in Sublime Text to easily inspect its contents.
we then loop over all ids until we find one that is not `0` . We will use this non-zero ID to get the `firstgid` offset of our tileset within our .tmj file.

Then we fetch the width and height without any changes compared to before the refactor.
The last change is that we assign `local_id` to the `level->cells[i]` by removing the `firstgid` `id_offset` from all numbers larger than 0. We only do this for non-zero values as `0` is not an actual tile ID but the info that there is nothing here. If we are not careful we could have created really odd behaviour with all zeroes suddenly becoming non-zero values.
You can see that we assign a pointer reference to `levelData` for `level->tileset` we have to update our `LevelData` struct to hold this pointer.

```cpp
// levels.h
struct LevelData {
  int w;
  int h;
  uint16_t* cells;
  const char* level_path;
  Entity* entityBuffer;
  int entityCount;
  const Tileset* tileset;
};

uint16_t GetCellID(LevelData* level, int x, int y);
```

We make sure the `tileset` is `const` as we are never going to want to adjust it when we pass it in alongside `LevelData` we have also updated `cells` to use `uint16_t` this is because `uint8_t` caps out at 255. And we could accidentally overshoot this value in a larger project with way more tiles.
We also update our `GetCellID` in both .h and .cpp to match this new return value.
Lets look at `CreateEntities()` next

```cpp
// levels.cpp
void CreateEntities(LevelData* lvl_data, Arena* arena) {
  Reset(arena);
  lvl_data->entityCount = 0;
  lvl_data->entityBuffer = (Entity*)Memory::Allocate(arena, sizeof(Entity) * 256);
  fstream stream(lvl_data->level_path);
  auto result = json::parse(stream);
  vector<uint16_t> entities;
  bool found = false;
  for (const auto& layer : result["layers"]) {
    if (layer["name"] == "entities") {
      entities = layer["data"].get<vector<uint16_t>>();
      found = true;
      break;
    }
  }
  if(!found) {
    return;
  }
  for (int i = 0; i < lvl_data->w * lvl_data->h; i++) {
    if(entities[i] == 0) {
      continue;
    }
    uint16_t entity_id = GetLocalTileID(entities[i], result);
    int x = i % lvl_data->w;
    int y = i / lvl_data->w;
    AddEntity((ENTITY_ID)entity_id, x, y, lvl_data);
  }
}
```

We get the JSON file from the `level_path` that was stored during `CreateLevel()` then we search for the layer named "entities" instead of "level" as we did during the previous function. Once we have found that layer we can fetch all the entities from the data block called "data". Similarly we flip the `found` bool to true.
But not similar to our `CreateLevel()` we don't assert that `found` was true, instead we return early. I've opted for this as I want to be able to test that a tileset is working properly even if I have not added an entities layer inside Tiled
Once we have our entities we go over each one and call `AddEntity()` as usual. But we make sure to fetch the tileset id using our helper function `GetLocalTileID()` . We could easily fetch the offset once and apply it to all the ids as we did in `CreateLevel()` . You're free to make this change if you want. But even rerunning this code for each entity has proven a non issue so far during testing.
Next its time to make `spriteLibrary.h/.cpp` have all the necessary info about a tileset in order to let us use it to render our level.
For now we're extending `Sprite` and `SpriteDataEntry` to hold variables related to the sprite being a tileset

```cpp
// spriteLibrary.h
struct Sprite {
  SDL_Texture* texture;
  int width;
  int height;
  int pivot_x;
  int pivot_y;
  int tileset_cell_count_x;
  int tileset_cell_count_y;
};

const int NOT_SET = -1;

struct SpriteDataEntry {
  SPRITE_ID id;
  const char* path;
  int pivot_x = NOT_SET;
  int pivot_y = NOT_SET;
  int tileset_cell_count_x = NOT_SET;
  int tileset_cell_count_y = NOT_SET;
};
```

We might opt for having a different struct all together for our Tilesets later. But for now I'm ok with expanding our `Sprite` struct.
We have also added our tilemap to our `SPRITE_ID` enum

```cpp
// spriteLibrary.h
enum class SPRITE_ID {
  Fallback,
  // Ground, <- removed
  // Ground_alt, <- removed
  // Wall, <- removed
  Rock,
  Demon,
  Medusa_Idle_Side,
  Medusa_Idle_Front,
  Medusa_Idle_Back,
  Golem,
  Siren,
  Dropshadow,
  titlescreen_background,
  black_1x1,
  dungeon_tileset
};
```

Then we update our `spriteLibrary.cpp`

```cpp
// spriteLibrary.cpp
static const SpriteDataEntry all_sprite_data[] = {
  {SPRITE_ID::Fallback, FALLBACK_PATH, 0, 0},
  {SPRITE_ID::Demon, "assets/sprites/player.png"},
  {SPRITE_ID::Rock, "assets/sprites/rock.png", 10, 20},
  {SPRITE_ID::Medusa_Idle_Side, "assets/sprites/medusa_idle_side.png", 12, 24},
  {SPRITE_ID::Medusa_Idle_Front, "assets/sprites/medusa_idle_front.png", 12, 24},
  {SPRITE_ID::Medusa_Idle_Back, "assets/sprites/medusa_idle_back.png", 12, 24},
  {SPRITE_ID::Dropshadow, "assets/sprites/dropshadow.png", 8, 8},
  {SPRITE_ID::black_1x1, "assets/sprites/1x1_black.png", 0, 0},
  {SPRITE_ID::titlescreen_background, "assets/sprites/titlescreen.png", 0, 0},
  {SPRITE_ID::dungeon_tileset, "assets/sprites/hell_of_a_time_dungeon_tileset.png", 0, 0, 9, 9}
};
```

because only `dungeon_tileset` is a tileset its the only one that has explicit values set for our new `tileset` variables. I counted the amount of rows and columns of the tileset by hand for this step. Later we might automate this as our .tsj file does know the amount of rows/columns it stores.

```cpp
// spriteLibrary.cpp
void LoadSprite(Sprite* spriteBuffer, SpriteDataEntry entry, SDL_Renderer* renderer) {
  SDL_Surface* surface = IMG_Load(entry.path);
  if(surface == nullptr) {
    surface = IMG_Load(FALLBACK_PATH);
  }
  assert(surface != nullptr);
  SDL_Texture* texture = SDL_CreateTextureFromSurface(renderer, surface);
  Sprite* sprite = &spriteBuffer[(int)entry.id];
  sprite->texture = texture;
  sprite->height = texture->h;
  sprite->width = texture->w;
  if(entry.pivot_x == NOT_SET || entry.pivot_y == NOT_SET) {
    sprite->pivot_x = sprite->width / 2;
    sprite->pivot_y = sprite->height / 2;
  }
  else {
    sprite->pivot_x = entry.pivot_x;
    sprite->pivot_y = entry.pivot_y;
  }
  sprite->tileset_cell_count_x = entry.tileset_cell_count_x;
  sprite->tileset_cell_count_y = entry.tileset_cell_count_y;
  SDL_DestroySurface(surface);
}
```

Then we update our `LoadSprite()` function to assign the tileset variables from the `SpriteDataEntry` to our `Sprite` .
in `TryMove()` in `game.cpp` and `RaycastFirstEntity()` in `Levels.cpp` we previously checked if we we've reached a wall. Now that we no longer have an `ID::GROUND/WALL` to check against we need to instead use our `walkable` bool that we collected when we created our level.
In `Levels.h/.cpp` we'll add a small helper function

```cpp
// levels.h
bool IsWalkable(int x, int y, LevelData* level);
```

```cpp
// levels.cpp
bool IsWalkable(int x, int y, LevelData* level) {
  uint16_t id = GetCellID(level, x, y);
  return level->tileset->walkableBuffer[id];
}
```

because our `walkableBuffer` is in alignment with our local cell ids we can find the walkable status by just checking the array at the same index.
In `main.cpp` we need to allocate our `tilesetBuffer` .

And because we're adding it to `arena_images` we need to give it more memory as we previously had the memory footprint set to exactly the size needed to hold our `Sprite*` array.

```cpp
// main.cpp
int SPRITE_COUNT = 256;
size_t IMAGE_ARENA_SIZE = MEGABYTES(1);
gameData->arena_images = Memory::CreateSubArena(arena_main, IMAGE_ARENA_SIZE);
gameData->spriteBuffer = ALLOC_ARRAY(gameData->arena_images, Sprite, SPRITE_COUNT);
gameData->tilesetBuffer = ALLOC_ARRAY(gameData->arena_images, Tileset, (int)TILESETS::COUNT);
```

We will need a new Render function to render our level using our tileset instead of individual `Sprite`s . We are going to use two `FRect` . One to tell SDL which box on the screen to draw our texture to. and the other what box/area within our tileset to fill the first Rect with. We're also removing `RenderSprite_Grid()` as after this refactor step no code is using it.

```cpp
// rendering.h
void RenderTile_World(Sprite* tileset, int cell_id, LevelData* lvl, SDL_Renderer* renderer, const Camera* camera, float x, float y, float scale, float alpha);
```

we've added `cell_id` and removed `flipped` as compared to `RenderSprite_World()`
We'll be calling this function from `levelRenderer.cpp`

```cpp
void RenderLevel(GameData* gameData, SDL_Renderer* renderer) {
  Gameplay* gameplay = &gameData->scenes.gameplay;
  LevelData* level = &gameplay->levels[gameplay->currentLevelIndex];
  Sprite* tileset;
  switch(level->tileset->type) {
    case TILESETS::Dungeon:
      tileset = GetSprite(SPRITE_ID::dungeon_tileset, gameData->spriteBuffer);
      break;
    case TILESETS::NONE:
    case TILESETS::COUNT:
      assert(false);
      break;
  }
  for(int x = 0; x < level->w; x++) {
    for (int y = 0 ; y < level->h; y++) {
      uint16_t id = GetCellID(level, x, y);
      RenderTile_World(tileset, id, level, renderer, &gameData->camera, x, y, 1, 1);
    }
  }
}
```

Currently we `assert(false)` if the `TILESETS` enum value was wrong. We really should have this part of the code live inside our tileset struct or a helper function. But just to get things up and running we're doing the linking between the tileset enum and the necessary tileset sprite here.
We get the full tileset sprite then we pass it along with the id so we can calculate which tile to render from the grid that is the tileset.
Lets finally look at the render function inside `rendering.cpp` to see how we render the tiles of our level

```cpp
void RenderTile_World(Sprite* tileset_atlas_sprite, int cell_id, LevelData* lvl, SDL_Renderer* renderer, const Camera* camera, float x, float y, float scale, float alpha) {
  SDL_FRect tilesetRect;
  tilesetRect.w = TILE_SIZE_PX_RAW;
  tilesetRect.h = TILE_SIZE_PX_RAW;
  tilesetRect.x = (cell_id % tileset_atlas_sprite->tileset_cell_count_x) * TILE_SIZE_PX_RAW;
  tilesetRect.y = (cell_id / tileset_atlas_sprite->tileset_cell_count_x) * TILE_SIZE_PX_RAW;
  camera::GridToWorld(&x, &y, lvl);
  SDL_FRect rect;
  rect.x = x;
  rect.y = y;
  float final_scale = UPSCALE_FACTOR * scale;
  rect.h = TILE_SIZE_PX_RAW * final_scale;
  rect.w = TILE_SIZE_PX_RAW * final_scale;
  rect.x -= tileset_atlas_sprite->pivot_x * final_scale;
  rect.y -= tileset_atlas_sprite->pivot_y * final_scale;
  rect.x -= camera->camera_x;
  rect.y -= camera->camera_y;
  SDL_SetTextureScaleMode(tileset_atlas_sprite->texture, SDL_SCALEMODE_PIXELART);
  SDL_SetTextureAlphaModFloat(tileset_atlas_sprite->texture, alpha);
  SDL_RenderTexture(renderer, tileset_atlas_sprite->texture, &tilesetRect, &rect);
}
```

We create our `tilesetRect` then we set the width and height of the rect to the actual dimensions of a tile before any scaling is applied. We then find the x and y coordinate of the tile by doing our 1D to 2D transformation using modulo `%` and divided by `/` . We then multiply the coordinate by the size of a tile to slide our rect into position. This 16x16 area within our tilemap that we've calculated will be the pixels drawn into our `rect` created just below.
We convert our x and y coordinates to world space then we set up our destination rect as usual. Accounting for the pivot, camera, scale and global `UPSCALE_FACTOR` .
Finally we make sure to pass along `&tilesetRect` to our `SDL_RenderTexture` where we previously used `NULL` . Because `NULL` meant use the entire texture.
now, after this pretty intense chapter we are rewarded with some actually decent graphics to look at. And it makes such a difference! Now adding more tilesets and making changes to them in Tiled will be easy!