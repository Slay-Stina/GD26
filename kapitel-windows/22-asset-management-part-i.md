# 22 Asset Management Part I

We're going to refactor out our `image.h/cpp` and create at least a slightly more robust way of loading sprites. The fact that we currently have our `Image*` pointers lying flat inside our `GameData` then loaded one-by-one in our `Initialize()` function inside `game.cpp` makes it very obvious that we should refactor as this solution is very transparent BUT more cumbersome than necessary.
Another issue is that we have to pass each `Image*` manually when we want to pass them to a function or pass the entire `GameData` struct.
We're making two changes to start with, we're removing everything inside `image.h/cpp` and instead creating `spriteLibarary.h/cpp` . We're also taking our `Image` struct and changing its name to `Sprite` .
> [!NOTE]
> We're renaming `Image` to `Sprite` using the built in rename command in Helix this is performed by having the caret over the word and pressing `space+r` . Then in the command line at the bottom of the screen we just erase/type the name we want to change it to. Finally pressing `enter` confirms the change. This will update the word across our entire codebase. (which is nice).

We'll soon be moving from our generic Sokoban game, with a player and a box, and adding some actual game design to this base shell. One of the first changes we're making is adding more contextually relevant IDs inside `entity.h`

```cpp
// entity.h
enum class ID : uint8_t {
  NONE = 0,
  GROUND = 1,
  WALL = 2,
  DEMON = 3,
  ROCK = 4,
  MEDUSA = 5,
  GHOST = 6,
  GOLEM = 7,
};
```

We've added Medusa Ghost and Golem as well as I've renamed Player to Demon and Box to Rock . With these changes we currently can't build our project because we have compilation errors. But after our refactor is finished we will be able to.

```cpp
// spriteLibrary.h
#pragma once
#include "SDL3/SDL_render.h"
#include "entity.h"

struct Sprite {
  SDL_Texture* texture;
  int width;
  int height;
};
```

so first we're copying over our `Image` struct and renaming it `Sprite` . in a later chapter we'll expand on this to help us with animation.

```cpp
// spriteLibrary.h
enum class SPRITE_ID {
  Fallback,
  Ground,
  Wall,
  Rock,
  Demon,
  Medusa,
  Golem,
  Ghost
};
```

we're also creating a new enum that will look very (very) similar to our Entity ID enum that we wrote earlier. Currently we're mapping our entities to a sprite 1-to-1 but later we'll be adding more `SPRITE_ID`s like `Demon_walking` or `Golem_pushing` . But for now we'll keep it like this.

```cpp
// spriteLibrary.h
struct SpriteDataEntry {
  SPRITE_ID id;
  const char* path;
};

Sprite* GetSpriteFromID(ID id, Sprite* spriteBuffer);

namespace AssetManagement {
  void LoadSprite(Sprite* spriteBuffer, SpriteDataEntry entry, SDL_Renderer* renderer);
  void LoadAllSprites(Sprite* spriteBuffer, SDL_Renderer* renderer);
}
```

We're creaing a `SpriteDataEntry` struct that pairs a `SPRITE_ID` with a corresponding path to where in the assets folder it is supposed to be found.
We're also creating a helper function `GetSpriteFromID()` that allows us to pass an ID and get back a `Sprite*` . The `spritebuffer` will be added to our `GameData` as a way of fetching every sprite we've loaded.

We're protecting our `LoadSprite` and new `LoadAllSprites` with the `AssetManagement` namespace. This is just to highlight that these functions have a specific place within our program. We're also changing our `LoadSprite` function to work with our allocated `spriteBuffer` instead of being passed our `Memory::Arena` directly
With everything added to `spriteLibrary.h` we can remove everything from `image.h` and remove `image.cpp` for our `SHARED_SOURCES` inside our `CmakeLists.txt` .

```cmake
set(SHARED_SOURCES
  # ${CMAKE_SOURCE_DIR}/src/image.cpp <--- remove this
  ${CMAKE_SOURCE_DIR}/src/arena.cpp
  ${CMAKE_SOURCE_DIR}/src/input.cpp
)
```

We'll be loading all of our sprites from inside our .DLL with saving their pointers in a new `Sprite* spriteBuffer`

```cpp
// gameState.h
struct GameData {
  // Image* fallback
  // Image* wall
  // Image* ground
  // etc
  Sprite* spriteBuffer;
  // other code hidden for clarity
};
```

We've deleted all of our direct `Image*` references from `GameData` and replaced them with a single `Sprite*` array, Now we need to go to our `main.cpp` and allocate our sprite buffer

```cpp
// main.cpp
int SPRITE_COUNT = 256;
size_t IMAGE_ARENA_SIZE = sizeof(Sprite) * SPRITE_COUNT;
gameData->arena_images = Memory::CreateSubArena(arena_main, IMAGE_ARENA_SIZE);
gameData->spriteBuffer = (Sprite*)Memory::Allocate(gameData->arena_images, sizeof(Sprite) * SPRITE_COUNT);
```

So in our little ocean of arena allocations we're allocating a subarena (just like before) then allocating our `spriteBuffer` to it. Currently each `Sprite` is 16 bytes meaning that our `arena_images` is exactly 4096 bytes (or ~4kb) in size.
We're also removing the line in `main.cpp` where we loaded fallback manually. This gets handled automatically from now on. (and if not removed will not let us compile our program as `image.cpp` is no longer part of our Executables known files)
Now lets look at our `spriteLibrary.cpp`

```cpp
// spritelibrary.cpp
#include "spriteLibrary.h"
#include "SDL3_Image/SDL_image.h"
#include <cassert>

const char* FALLBACK_PATH = "assets/sprites/fallback.png";

static const SpriteDataEntry all_sprite_data[] = {
  {SPRITE_ID::Fallback, FALLBACK_PATH},
  {SPRITE_ID::Wall, "assets/sprites/wall.png"},
  {SPRITE_ID::Demon, "assets/sprites/player.png"},
  {SPRITE_ID::Rock, "assets/sprites/box.png"},
  {SPRITE_ID::Ground, "assets/sprites/ground.png"},
  {SPRITE_ID::Medusa, "assets/sprites/medusa.png"},
};
```

Here we have first a path to where our fallback sprite is located. We store this to help us with our `LoadSprite` in case we have an issue with a sprite not exiting.
We then create a `static` array of our `SpriteDataEntry` . Using the c++ way we're allowed to outline a struct with just a pair of curcly bracer `{}` we can add the contents of this array right here inside the script. the `static` keyword makes the array scoped only to our `spriteLibrary.cpp` meaning that no other file can ever access it and even if another file had a variable with the exact same name it would not create a compile conflict. This variable is constructed for us before `main()` even runs and will live for the duration of the program. adding `const` makes this piece of memory read-only as nothing is allowed to change it at runtime.
We might revisit this setup later and try and automate it a bit more.
With this setup we can declare the contents of our array up front

```cpp
// example
static const VariableType name[] = {
  {},
  {},
  {}
};
```

in this example code we have an array of 3 elements (empty for the sake of simplicity). The array can't be modified by code and the array will at compile-time get the correct size 3 automatically as the compiler finds three items in it. Each item is separated by a `,` comma and any extra newline is just to help us humans lay it out in a more easily understood way.
our `SpriteDataEntry` struct has two variables. first a `SPRITE_ID` then a `const char* path` . This has to be taken into account when using just the `{}` setup. As the variables are added in the order they show up inside the struct.

```cpp
// spritelibrary.cpp
Sprite* GetSpriteFromID(ID id, Sprite* spriteBuffer) {
  switch (id) {
    case ID::NONE:
      return nullptr;
    case ID::GROUND:
      return &spriteBuffer[(int)SPRITE_ID::Ground];
    case ID::WALL:
      return &spriteBuffer[(int)SPRITE_ID::Wall];
    case ID::DEMON:
      return &spriteBuffer[(int)SPRITE_ID::Demon];
    case ID::ROCK:
      return &spriteBuffer[(int)SPRITE_ID::Rock];
    case ID::MEDUSA:
      return &spriteBuffer[(int)SPRITE_ID::Medusa];
    case ID::GHOST:
      return &spriteBuffer[(int)SPRITE_ID::Ghost];
    case ID::GOLEM:
      return &spriteBuffer[(int)SPRITE_ID::Golem];
    default:
      return &spriteBuffer[(int)SPRITE_ID::Fallback];
  }
}
```

The next part is to add the logic for our `GetSpriteFromID` function. It accepts an ID and passes along our `spriteBuffer` . Then it maps each of the IDs to its corresponding `SPRITE_ID` , it also returns our fallback sprite in the `default` case. This happens when we pass an ID to the function that we have not created a case for yet. We need to cast our `SPRITE_ID` to `int` as arrays expect ints for their index. Then we take a pointer to that sprite inside the `spriteBuffer` using the `&` operator.
Right now, as mentioned previously, this just maps each entity to a sprite 1-to-1. But this will be refactored later when we add animations.

```cpp
// spriteLibrary.cpp
namespace AssetManagement {
  void LoadAllSprites(Sprite* spriteBuffer, SDL_Renderer *renderer) {
    for (SpriteDataEntry entry : all_sprite_data) {
      LoadSprite(spriteBuffer, entry, renderer);
    }
  }

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
    SDL_DestroySurface(surface);
  }
}
```

next, inside the same namespace we have in our `spriteLibrary.h` we add the logic for our `LoadAllSprites()` and `LoadSprite()` functions.
`LoadAllSprites` accepts the `spriteBuffer` and will pass it along each time it calls `LoadSprite()` from its for-loop
It then uses a nifty simplified way of writing a for-loop that first asks us about the type we want to loop over `SpriteEntryData` in this case. It asks us to give it a name `entry` then asks us what array we're working with. we pass it our `all_sprite_data` array that we wrote. This is much simpler to write as we're asking the compiler to infer the size of the array. Otherwise we would have to either write the array size manually or use a bit of code to calculate it.
For each of the `SpriteDataEntry` elements in the `all_sprite_data` array we call `LoadSprite()` and we pass along the `entry` so that our `LoadSprite()` function can fetch the correct path and ID . To update the `Sprite` that the `SpriteBuffer` points to we need to fetch the pointer to the value, we do this by using the pointer to operator `&` . Now when we make changes to `texture, height and width` these will persist at the location pointed to by `spriteBuffer` at that index.
Other than the change to what parameters are passed in and how `path` is fetched from our `entry` the `LoadSprite()` is similar to its original version.
Now it's time to fully deprecate `Image.h/cpp` I have opted for just making these files empty to help with continuity in the lecture series. But I recommend deleting these files all together.
At this point you will have errors related to `#include "image.h"` that was found in more than a few of our files. We can safely delete those lines and if a file can't find `Sprite` its because we have not included `#include "spritelibrary.h"`
we'll go to our `game.cpp` and erase all lines where we try and set our old `Image*` manually. Swapping them out for our new logic

```cpp
// game.cpp
void Initialize(GameData* data, SDL_Window* window, SDL_Renderer* renderer) {
  // Image* fallback = LoadSprite(... ... ...) <---- all manual sprite loading removed
  // Image* box = ... ... ...
  // <----- all manual sprite loading removed
  AssetManagement::LoadAllSprites(data->spriteBuffer, renderer);
  // other code hidden for clarity
}
```

Finally we can go to our `levelRenderer.cpp` and simplify both `RenderLevel()` and `RenderEntities()`

```cpp
// levelRenderer.cpp
void RenderLevel(GameData* gameData, SDL_Renderer* renderer) {
  LevelData lvl = gameData->levels[gameData->currentLevelIndex];
  for(int x = 0; x < lvl.w; x++) {
    for (int y = 0 ; y < lvl.h; y++) {
      uint8_t cellType = lvl.GetCellID(x, y);
      Sprite* sprite = GetSpriteFromID((ID)cellType, gameData->spriteBuffer);
      RenderSprite_Grid(sprite, &lvl, renderer, &gameData->camera, x, y);
    }
  }
}

void RenderEntities(GameData* data, SDL_Renderer* renderer) {
  LevelData lvl = data->levels[data->currentLevelIndex];
  loop(i, lvl.entityCount) {
    Entity entity = lvl.entityBuffer[i];
    Sprite* sprite = GetSpriteFromID(entity.id, data->spriteBuffer);
    float x_animated = std::lerp(entity.x_prev, entity.x, entity.progress_01);
    float y_animated = std::lerp(entity.y_prev, entity.y, entity.progress_01);
    RenderSprite_Grid(sprite, &lvl, renderer, &data->camera, x_animated, y_animated);
  }
}
```

now instead of each function needing a switch case to fetch the correct `Image*` (now `Sprite*` ) we can use our helper function `GetSpriteFromID` instead.
With these changes we have refactored our asset loading system and how we fetch sprites.