> **Linux:** This chapter is adapted for Linux.

# Rendering images

So far we've just rendered an SDL_FRect to the screen. But we of course want to have our own .PNG or .BMP files and use those. To accomplish this we must do the following things:

1. Put an image into our assets folder
2. Expand our cmakelists.txt to copy over our assets folder
3. Prepare a portion of memory to store our images
4. Use SDL3_image from its corresponding SDL3_image.h file downloaded earlier
   > In case you don't have both these files, install SDL3_image using your package manager:
   > - **apt:** `sudo apt install libsdl3-image-dev`
   > - **dnf:** `sudo dnf install SDL3_image-devel`
   > - **pacman:** `sudo pacman -S sdl3_image`
   >
   > Or download from https://github.com/libsdl-org/SDL_image/releases
5. Swap our SDL_FRect to a texture

Opening any drawing software we can create a 32x32px square and fill it with whatever shapes and colors we please. I've created a red square with an 'X' running through it. I've saved it as "fallback.png" as this will be the sprite that gets loaded whenever I attempt to load a sprite that doesn't exist.

Inside my assets folder I've created a subdirectory `sprites` and added my `fallback.png` to it.

## Updating our cmakelists.txt

At the top of our cmakelists.txt we will be adding the following code:

```cmake
set(DIR_ASSETS_ORIGIN ${CMAKE_SOURCE_DIR}/assets)
set(DIR_ASSETS_DESTINATION $<TARGET_FILE_DIR:${PROJECT_NAME}>/assets)
```

Then at the very bottom of our cmakelists.txt we add:

```cmake
add_custom_command(TARGET ${PROJECT_NAME} POST_BUILD
  COMMAND ${CMAKE_COMMAND} -E copy_directory_if_different ${DIR_ASSETS_ORIGIN} ${DIR_ASSETS_DESTINATION}
  VERBATIM
)
```

With this updated cmakelists.txt we can start working with asset files.

The next step is taking our big monolith memory arena and placing another arena inside of it, segmenting a section of memory to be the exclusive area to hold pointers to our sprites.

> **NOTE:** sprites aka textures live on the GPU inside our VRAM compared to our game data that lives on the CPU. We will need to convert each .PNG file into `SDL_GPUTexture` storing it in VRAM and accessing it by pointer reference inside our memory arena.

At the top of our `main()` we will be adding a new memory arena by allocating it directly inside our top-level memory arena:

```cpp
void* game_memory = AllocateGameMemory();
if(game_memory == nullptr){
  return 1;
}
Memory::Arena* arena_main = new Memory::Arena();
Memory::Initialize(arena_main, game_memory, GAME_MEMORY_ALLOWANCE);
GameData* gameData = (GameData*)Memory::Allocate(arena_main, sizeof(GameData));
Memory::Arena* arena_image = (Memory::Arena*)Memory::Allocate(arena_main, sizeof(Memory::Arena));
void* image_memory_start = Memory::Allocate(arena_main, GAME_MEMORY_IMAGES);
Memory::Initialize(arena_image, image_memory_start, GAME_MEMORY_IMAGES);
```

We will set up a new struct `Image` to hold the relevant variables. We will be storing this in a new `image.h` file:

```cpp
#pragma once
#include <SDL3/SDL.h>
#include "arena.h"
struct Image{
  SDL_Texture* texture;
  int width;
  int height;
};
namespace AssetManagement
{
  Image* LoadSprite(Memory::Arena* arena, SDL_Renderer* renderer, const char* path);
}
```

> **Linux:** On Linux, include `<SDL3/SDL.h>` which provides all SDL headers.

Next we will need to add the actual code to `LoadImage`:

```cpp
#include <cassert>
#include <string>
#include <SDL3/SDL.h>
#include <SDL3_image/SDL_image.h>
#include "image.h"
#include "arena.h"
using namespace std;
const char* DIRECTORY = "assets/sprites/";
const char* FALLBACK = "assets/sprites/fallback.png";
Image* AssetManagement::LoadSprite(Memory::Arena* arena, SDL_Renderer* renderer, const char* name){
  string path = DIRECTORY;
  path = path.append(name);
  SDL_Surface* surface = IMG_Load(path.c_str());
  if(surface == nullptr){
    surface = IMG_Load(FALLBACK);
  }
  assert(surface != nullptr);
  SDL_Texture* texture = SDL_CreateTextureFromSurface(renderer, surface);
  Image* img = (Image*)Memory::Allocate(arena, sizeof(Image));
  img->texture = texture;
  img->height = texture->h;
  img->width = texture->w;
  SDL_DestroySurface(surface);
  return img;
}
```

We will modify our cmakelists.txt to create a list of shared scripts:

```cmake
# Flag executable specific files
set(EXE_EXCLUSIVE ${CMAKE_SOURCE_DIR}/src/main.cpp
)
# Flag files that both our executable and shared library need to know about
set(SHARED_SOURCES
  ${CMAKE_SOURCE_DIR}/src/image.cpp
  ${CMAKE_SOURCE_DIR}/src/arena.cpp
)
```

We can then create a new static library:

```cmake
set(SHARED_LIB_NAME ${PROJECT_NAME}_common)
# SHARED STATIC LIBRARY
add_library(${SHARED_LIB_NAME} STATIC ${SHARED_SOURCES})
target_include_directories(${SHARED_LIB_NAME} PRIVATE include)
```

Inside our `GameData` struct we could store a series of `Image*`:

```cpp
Image* ship;
Image* asteroid_big;
Image* asteroid_small;
Image* background;
```

Lets add our fallback texture to our `GameData` and expand our `Draw()`:

```cpp
#pragma once
#include <SDL3/SDL.h>
#include "image.h"
struct GameData {
  SDL_FRect rect;
  float move_speed;
  Image* fallback;
};
```

Now lets use our memory arena and our new `LoadSprite()` function:

```cpp
gameData->fallback = AssetManagement::LoadSprite(arena_image, renderer, "fallback.png");
```

We should also update the size of our `arena_image`:

```cpp
size_t IMAGE_ARENA_SIZE = sizeof(Image) * 1024;
Memory::Arena* arena_image = (Memory::Arena*)Memory::Allocate(arena_main, sizeof(Memory::Arena));
void* image_memory_start = Memory::Allocate(arena_main, IMAGE_ARENA_SIZE);
Memory::Initialize(arena_image, image_memory_start, IMAGE_ARENA_SIZE);
```

In our `Draw()` function:

```cpp
void Draw(GameData* data, SDL_Renderer* renderer){
  SDL_SetRenderDrawColor(renderer, 0, 70, 8, 255);
  SDL_RenderClear(renderer);
  SDL_RenderTexture(renderer, data->fallback->texture, NULL, &data->rect);
  SDL_RenderPresent(renderer);
}
```

Lets create a simplified rendering function in `rendering.h`:

```cpp
#pragma once
#include <SDL3/SDL.h>
void RenderSprite(Image* sprite, SDL_Renderer* renderer, int xPos, int yPos, float scale = 1);
```

And in `rendering.cpp`:

```cpp
#include "rendering.h"
#include <SDL3/SDL.h>
void RenderSprite(Image* sprite, SDL_Renderer* renderer, int xPos, int yPos, float scale){
  SDL_FRect rect;
  rect.x = xPos;
  rect.y = yPos;
  rect.h = sprite->height * scale;
  rect.w = sprite->width * scale;
  SDL_RenderTexture(renderer, sprite->texture, NULL, &rect);
}
```

Inside `game.cpp` our `Draw()` function now reads:

```cpp
void Draw(GameData* data, SDL_Renderer* renderer){
  SDL_SetRenderDrawColor(renderer, 0, 70, 8, 255);
  SDL_RenderClear(renderer);
  RenderSprite(data->fallback, renderer, 50, 50);
  SDL_RenderPresent(renderer);
}
```

We will be implementing an enforced framerate. For a 60FPS game each tick gets 1/60 seconds to process.

```cpp
void CalculateRemainingFrameTime_MS(double* milliseconds){
  Uint64 frame_end_time_ns = SDL_GetTicksNS();
  double frame_time_spent_ns = frame_end_time_ns - PREV;
  double frame_time_spent_ms = frame_time_spent_ns / 1e6;
  *milliseconds = FRAME_TIME_MS - frame_time_spent_ms;
}
```

Our delay logic:

```cpp
double time_to_sleep_ms;
CalculateRemainingFrameTime_MS(&time_to_sleep_ms);
if(time_to_sleep_ms > 0){
  if(time_to_sleep_ms > 1){
    SDL_Delay(time_to_sleep_ms - 1);
  }
  while (time_to_sleep_ms > 0) {
    CalculateRemainingFrameTime_MS(&time_to_sleep_ms);
  }
}
else{
  printf("missed frame \n");
}
```

`SDL_Delay()` works cross-platform and is the recommended function to use.

Now our application enforces a stable framerate!

A lot of what we do in our `main.cpp` is boilerplate code that will be reusable in just about every project.
