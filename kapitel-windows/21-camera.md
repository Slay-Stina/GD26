# 21 Camera

In this chapter we'll implement a naive camera as well as refactor rendering code to simplify asking questions about positions as well as simplifying the render functions inside `levelRenderer.h/cpp` .
A camera in a 2D game is, at its simplest, a position in space. We'll be taking that position and shifting everything we render by that amount multiplied by -1 . This means that as the camera shifts right, everything drawn shifts left.
We're setting this up to later simplify mouse controls when we start working on additional dev tools. But the best part is that once we're done we can create new functions responsible for rendering with a fraction of the code we currently have in `RenderEntities()` and `RenderLevel()` . So once we're done with this chapter, if we've set everything up right the game will look exactly the same - and that's a good thing.
We're setting up a new `camera.h/cpp`

```cpp
// camera.h
#pragma once
#include "levels.h"

struct Camera {
  float camera_x;
  float camera_y;
};

namespace camera {
  void GridToWorld(float* x, float* y, const LevelData* lvl);
  void WorldToGrid(float x_world, float y_world, int* x, int* y, const LevelData* lvl);
  bool GetIsPointInsideGrid(float x, float y, const LevelData* lvl);
};
```

for now, our `Camera` struct has only two floats, responsible for storing the position of the camera. Later we'll expand this list of variables as we create additional camera features.
we're also creating three useful helper functions. `GridToWorld()` will let us specify a position on the game board and get back the actual position in the game window. `WorldToGrid()` does the opposite and takes any point in space and finds what cell this would belong to on the game board. Note that this can give us positions that are outside of the level bounds. To easily reason about what is inside and outside of the grid we will also be using the `GetIsPointInsideGrid()`
The implementation in our `camera.cpp` looks like:

```cpp
// camera.cpp
#include "camera.h"
#include "common.h"

bool camera::GetIsPointInsideGrid(float x, float y, const LevelData* lvl) {
  int x_grid;
  int y_grid;
  WorldToGrid(x, y, &x_grid, &y_grid, lvl);
  return x_grid >= 0 && y_grid >= 0 && x_grid < lvl->w && y_grid < lvl->h;
}

void camera::GridToWorld(float* x, float* y, const LevelData* lvl) {
  *x *= CELL_SIZE_PX;
  *x += SCREEN_WIDTH / 2.0;
  *x -= lvl->w * CELL_SIZE_PX / 2.0;
  *y *= CELL_SIZE_PX;
  *y += SCREEN_HEIGHT / 2.0;
  *y -= lvl->h * CELL_SIZE_PX / 2.0;
}

void camera::WorldToGrid(float x_world, float y_world, int* x, int* y, const LevelData* lvl) {
  *x = x_world;
  *y = y_world;
  *x += lvl->w * CELL_SIZE_PX / 2.0;
  *x -= SCREEN_WIDTH / 2.0;
  *x /= CELL_SIZE_PX;
  *y += lvl->h * CELL_SIZE_PX / 2.0;
  *y -= SCREEN_HEIGHT / 2.0;
  *y /= CELL_SIZE_PX;
}
```

Our `WorldToGrid()` function accepts two floats that specify the point in space to check, then two pointers to integers, these integer pointers are being modified by the function. So for our `GetIsPointInsideGrid()` we create two new integers then pass along pointer references to them using `&` . The return looks a bit long and scary, but we're just making sure that the x/y_grid is larger than zero and smaller than the width w and height h of the level we passed in. We can chain multiple and operators aka `&&` to make our expression only evaluate to true if all conditions were true.
`GridToWorld()` actually does the same arithmetic as our `RenderEntities()` and `RenderLevel()` did in `levelRenderer.cpp` but with the change that we're operating on the two float pointers we passed in. And to do that we need to make changes to the values they point to using the dereference operator aka putting a `*` before the variable name. Inside `GridToWorld()` we do the following steps for both x and y

2. we take the size of a cell in pixels and multiply it with the cell coordinate. Shifting our coordinate into pixels
3. add half of the width of the screen to make the 0,0 position be at the center of the screen instead of at the top left corner
4. we take the width of the level aka the total amount of cells then convert that number to a length in pixels and remove half of it from x/y . This shifts the position so that the center of the level is at the center of the screen.

for `WorldToGrid()` we do the same operations but in reverse to get the same result back.
we pass `LevelData*` as a `const` pointer to indicate that we're not supposed to make any changes to it inside these functions, just use its variables.
We're making some small changes to our `rendering.h/cpp` to include our `Camera` struct in our rendering step

```cpp
// rendering.h
#pragma once
#include "SDL3/SDL_render.h"
#include "camera.h"
#include "image.h"
#include "levels.h"

void RenderSprite_World(Image* sprite, SDL_Renderer* renderer, const Camera* camera, float x, float y, float scale = 1);
void RenderSprite_Grid(Image* sprite, LevelData* lvl, SDL_Renderer* renderer, const Camera* camera, float x, float y, float scale = 1);
```

We have taken what was previously a single `RenderSprite()` function and made a distinction between `_World` and `_Grid` . This is to give us a simplified way of rendering using an entities grid position instead of always having to manually do the convertion between grid and world.

```cpp
// rendering.cpp
#include "rendering.h"
#include "SDL3/SDL_render.h"
#include "common.h"

void RenderSprite_World(Image* sprite, SDL_Renderer* renderer, const Camera* camera, float x, float y, float scale) {
  SDL_FRect rect;
  rect.x = x;
  rect.y = y;
  rect.h = sprite->height * UPSCALE_FACTOR * scale;
  rect.w = sprite->width * UPSCALE_FACTOR * scale;
  rect.x -= camera->camera_x;
  rect.y -= camera->camera_y;
  SDL_RenderTexture(renderer, sprite->texture, NULL, &rect);
}

void RenderSprite_Grid(Image* sprite, LevelData* lvl, SDL_Renderer* renderer, const Camera* camera, float x, float y, float scale) {
  camera::GridToWorld(&x, &y, lvl);
  RenderSprite_World(sprite, renderer, camera, x, y, scale);
}
```

We can see how `RenderSprite_World()` is the same as our old `RenderSprite()` except we adjust the final `rect.x/y` by the camera's position. `RenderSprite_Grid()` just uses our newly created `GridToWorld()` function before calling `RenderSprite_World` making this just a small helper function really.
finishing up we're adding a `Camera` struct variable to our `GameData` inside `gameState.h` . We need to pass this along to our `levelRenderer.h/cpp` functions, but

```cpp
// gameState.h
struct GameData {
  // other variables hidden for clarity
  Camera camera;
};
```

next we add `#include "camera.h"` in `levelRenderer.cpp` and simplify our Render functions a lot.

```cpp
// levelRenderer.cpp
#include "levelRenderer.h"
#include "common.h"
#include "rendering.h"
#include <cmath>

void RenderLevel(GameData* gameData, SDL_Renderer* renderer) {
  LevelData lvl = gameData->levels[gameData->currentLevelIndex];
  for(int x = 0; x < lvl.w; x++) {
    for (int y = 0 ; y < lvl.h; y++) {
      uint8_t cellType = lvl.GetCellID(x, y);
      Image* sprite;
      switch(cellType) {
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
      RenderSprite_Grid(sprite, &lvl, renderer, &gameData->camera, x, y);
    }
  }
}
```

Now instead of having the grid to pixel calculations in `RenderLevel()` we just fetch the relevant sprite and call `RenderSprite_Grid()` .

```cpp
// levelRenderer.cpp
void RenderEntities(GameData* data, SDL_Renderer* renderer) {
  LevelData lvl = data->levels[data->currentLevelIndex];
  loop(i, lvl.entityCount) {
    Image* img;
    Entity entity = lvl.entityBuffer[i];
    switch(entity.id) {
      case ID::PLAYER:
        img = data->player;
        break;
      case ID::BOX:
        img = data->box;
        break;
      default:
        img = data->fallback;
        break;
    }
    float x_animated = std::lerp(entity.x_prev, entity.x, entity.progress_01);
    float y_animated = std::lerp(entity.y_prev, entity.y, entity.progress_01);
    RenderSprite_Grid(img, &lvl, renderer, &data->camera, x_animated, y_animated);
  }
}
```

We still perform our lerp logic inside `RenderEntities()` to get the point between `x/y_prev` and `x/y` . But then it's very similar.
That's it. We have done a bit of cleanup and layed the foundation for a camera and simplified our render logic!