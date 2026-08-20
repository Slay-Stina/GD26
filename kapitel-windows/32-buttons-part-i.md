# 32 Buttons Part I

We're going to create the skeleton of a main menu in this chapter. We'll need buttons, a way to render them and the logic to allow us to press them.
Our `GameData` struct and the variables inside `gameState.h` have started to grow. And our refactoring step of putting scene specific variables in their own struct did help I want to go further. So at this stage, as we're adding the variables for our main menu, we'll be putting them inside its own .h/.cpp file. So to start with lets set up `mainmenu.h/.cpp`

```cpp
// mainmenu.h
#pragma once

struct GameData;
struct SDL_Renderer;
struct Button;
struct Sprite;

namespace Memory {
  struct Arena;
}
```

We are including `mainmenu.h` in `gameState.h` and if we were to include `gameState.h` inside `mainmenu.h` we would get a chain of circular dependencies. To fix this we're forward declaring all the structs we'll be using in this file.

```cpp
// mainmenu.h
struct MainMenu {
  Button* buttons;
  int button_count;
  int activeButtonIndex;
  Button** activeButtons;
  int activeButtonCount;
  bool initialized;
};

void InitializeMenu(MainMenu* mainmenu, Sprite* spriteBuffer, Memory::Arena* arena_main);
void UpdateMenu(GameData* data);
void DrawMenu(MainMenu* mainmenu, SDL_Renderer* renderer, Sprite* spriteBuffer);
```

`buttons` is an array with the size of `button_count` .

`activeButtonIndex` will be used to control how our buttons are rendered and to make sure we press the correct button.

our `activeButtons**` pointer pointer array will be a subset of `buttons*` that are all `is_active` set to `true` . `initialized` will be set during `IntializeMenu` and an `assert` will make sure we don't reinitialize our menu.
Right now our main menu is very similar to our titlescreen, with the addition of a new type of struct `Button` .

We'll look at `button.h` right after this.

Our main menu has `Initialize`, `Update` and `Draw` we'll be calling these when appropriate from `game.cpp`

```cpp
// button.h
#pragma once
#include "SDL3/SDL_rect.h"
#include "SDL3/SDL_render.h"

struct Sprite;
struct GameData;
```

We do some normal inclusion of SDL3 headers and forward declare `Sprite` and `GameData` . To be honest I don't think we necesserily need to forward declare in this case.

```cpp
// button.h
enum class ButtonMode {
  Centered,
  Raw
};

enum class ButtonType {
  NONE,
  START_GAME,
  QUIT
};

struct Button {
  ButtonType type;
  SDL_FRect rect;
  SDL_Texture* texture;
  bool is_active;
};
```

We create our `Button` struct, it has a type, a rectangle to be drawn in and a pointer to a Texture to render to the screen. `is_active` will allow us to add buttons to a scene without having them display until we're ready.

```cpp
// button.h
void PressButton(Button* button, GameData* data);
int GetActiveButtonCount(Button* buttons, int count);
bool IsHoveredOver(Button* button, float x, float y);
void SetupButton(Button* button, ButtonType type, Sprite* spriteBuffer, SDL_FRect rect, ButtonMode mode);
```

We'll call `SetupButton` when we create a button. `IsHoveredOver` checks the current mouse position and does some basic collision bounds detection. `GetActiveButtonCount` is a small helper function to loop over all buttons and return the total count of buttons with `is_active` set to `true` .
We'll be checking our collision from `collision.h` . Our bounds collision detection is the most basic type of collision we can detect

```cpp
// collision.h
#pragma once
#include "SDL3/SDL_rect.h"

bool CheckCollisionInsideBounds(SDL_FRect bounds, float x, float y);
```

Lets look at the implementation of this function

```cpp
// collision.cpp
#include "collision.h"

bool CheckCollisionInsideBounds(SDL_FRect bounds, float x, float y) {
  if(x > bounds.x + bounds.w) return false;
  if(x < bounds.x) return false;
  if(y > bounds.y + bounds.h) return false;
  if(y < bounds.y) return false;
  return true;
}
```

by checking the x and y positions and returning false if any of them are outside of the rectangles area we've implemented collision detection between a point in space and a rectangle! To check collision between two rectangles instead we use something called AABB collision detection but we wont be needing that for this game. Just wanted you to have heard about it.

```cpp
// button.cpp
#include "button.h"
#include "collision.h"
#include "game.h"
#include "gameState.h"
#include "spriteLibrary.h"

bool IsHoveredOver(Button* button, float x, float y) {
  if(button == nullptr) return false;
  if(button->is_active == false) return false;
  assert(button->type != ButtonType::NONE);
  return CheckCollisionInsideBounds(button->rect, x, y);
}
```

we make sure the button is correctly set up or at least not supposed to be triggered (null pointer or not active). Then we call our new collision detection function and return the bool result.

```cpp
// button.cpp
void SetupButton(Button* button, ButtonType type, Sprite* spriteBuffer, SDL_FRect rect, ButtonMode mode) {
  assert(type != ButtonType::NONE);
  button->type = type;
  button->rect = rect;
  if(mode == ButtonMode::Centered) {
    button->rect.x -= button->rect.w / 2;
    button->rect.y -= button->rect.h / 2;
  }
  button->is_active = true;
  switch(button->type) {
    case ButtonType::START_GAME:
      button->texture = GetSprite(SPRITE_ID::Fallback, spriteBuffer)->texture;
      break;
    case ButtonType::QUIT:
      button->texture = GetSprite(SPRITE_ID::Fallback, spriteBuffer)->texture;
      break;
    default:
      button->texture = GetSprite(SPRITE_ID::Fallback, spriteBuffer)->texture;
      break;
  }
}
```

here we set all the internals of a new `Button` struct. currently we assign fallback as the texture for all of the buttons. in a later chapter we'll update these with some actual graphics. We are also doing a bit more defensive programming with `assert` in this chapter than usual. But it's good pratice to do so.

```cpp
// button.cpp
void PressButton(Button *button, GameData *data) {
  if(button == nullptr) {
    return;
  }
  assert(button->is_active);
  switch(button->type) {
    case ButtonType::NONE:
      assert(false);
    case ButtonType::START_GAME:
      ChangeScene(data, SCENE_TYPES::GAME);
      break;
    case ButtonType::QUIT:
      data->running = false;
      break;
  }
}
```

It's a bit brutish to pass along the entire `GameData` pointer, but with it we can have our buttons really affect the entire program. (for good and for bad).

```cpp
// button.cpp
int GetActiveButtonCount(Button *buttons, int count) {
  if(count == 0) {
    return 0;
  }
  int amount = 0;
  for (int i = 0; i < count; i++) {
    if(buttons[i].is_active) {
      amount++;
    }
  }
  return amount;
}
```

lastly we have a function that loops over all buttons and increments `amount` whenever we find a button that is active. Finally it returns this value. Right now only our main menu have buttons inside the game. But once we add more we'll continue to create these helper functions to assist us with our boilerplate. But right now we could easily take our one callsite and just add the code inside this function to it. It is really only a good idea to create a function when the code inside it is called from more than 1 place. Every function and file adds obsuscation to our code by not having the logic live next to each other.
We'll be using a bespoke rendering function for our buttons as they live in UI-space and not in game space. The camera should for example not affect a button at all.

```cpp
// rendering.h
void RenderButton(Button* button, bool is_selected, SDL_Renderer* renderer);
```

and then the implementation

```cpp
// rendering.cpp
void RenderButton(Button* button, bool is_selected, SDL_Renderer* renderer) {
  SDL_Texture* texture = button->texture;
  SDL_SetTextureScaleMode(texture, SDL_SCALEMODE_PIXELART);
  uint8_t colorOverlay = is_selected ? 255 : 230;
  SDL_SetTextureBlendMode(texture, SDL_BLENDMODE_BLEND);
  SDL_SetTextureColorMod(texture, colorOverlay, colorOverlay, colorOverlay);
  SDL_RenderTexture(renderer, button->texture, NULL, &button->rect);
}
```

we fetch the texture, set it to have no bilinear filtering. We then work with something we don't do usually. We are gonig to be adding color ontop of our button. We are either adding (255,255,255) on top or (230,230,230) this is the RGB values that are mapped to an `uint8_t` meaning that we have a precision of 0-255. giving us 255x255x255 = 16581375 unique colors in our color space. 255 in all channels means pure white - aka no added color at all. 230,230,230 adds a very light grey ontop, making the texture a little darker. We use this to make all buttons that are not the selected button a bit darker. We assing this overlayed color usign `SDL_SetTextureColorMod` .

lastly we render the texture to the backbuffer.
We're simplifying access to our `running` boolean by passing it into our `GameData` struct

```cpp
// gameState.h
struct GameData {
  // other variables hidden for clarity
  bool running;
};
```

And in `main.cpp` we remove our local `running` and substitute `gameData->running`

```cpp
// main.cpp
gameData->running = true;
float dt;
float dt_scaler = 1;
gameData->dt = &dt;
gameData->dt_scaler = &dt_scaler;
while(gameData->running) {
  // ...
}
```

Now lets take our logic and call it from `game.cpp`

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
  InitializeMenu(&data->scenes.mainMenu, data->spriteBuffer, data->arena_main);
  ChangeScene(data, SCENE_TYPES::MAINMENU);
}
```

in `Update()` we call into mainmenu

```cpp
// game.cpp
case SCENE_TYPES::MAINMENU:
  UpdateMenu(data);
  break;
```

And in `Draw()`

```cpp
// game.cpp
case SCENE_TYPES::MAINMENU:
  DrawMenu(&data->scenes.mainMenu, renderer, data->spriteBuffer);
  break;
```

With everything set up we can fill out the different functions in `mainmenu.cpp`

```cpp
// mainmenu.cpp
#include "mainmenu.h"
#include "SDL3/SDL_scancode.h"
#include "arena.h"
#include "button.h"
#include "common.h"
#include "gameState.h"
#include "input.h"
#include "rendering.h"
#include "spriteLibrary.h"
#include <cassert>

void InitializeMenu(MainMenu* mainmenu, Sprite* spriteBuffer, Memory::Arena* arena_main) {
  assert(mainmenu->initialized == false);
  mainmenu->button_count = 2;
  mainmenu->buttons = ALLOC_ARRAY(arena_main, Button, mainmenu->button_count);
  SetupButton(&mainmenu->buttons[0], ButtonType::START_GAME, spriteBuffer, {SCREEN_WIDTH / 2.0, SCREEN_HEIGHT / 2.0, 200, 80}, ButtonMode::Centered);
  SetupButton(&mainmenu->buttons[1], ButtonType::QUIT, spriteBuffer, {SCREEN_WIDTH / 2.0, (SCREEN_HEIGHT / 2.0) + 100, 200, 80}, ButtonMode::Centered);
  mainmenu->initialized = true;
}
```

During initialization we make sure we haven't already done our initialization. Then we currently hard-code our `button_count` meaning that if we want to setup a new button we need to remember to increase this number. There are a multitude of ways of refactoring out this error-prone step. We might do that later. Otherwise it's a simple exersice for you, the reader.
We allocate our buttons to our `arena_main` as we have no need of ever purging them from memory. We then call `SetupButton` passing along the relevant variables. The biggest parameter is our `SDL_FRect` that we pass with the condensed `{}` syntax.
I've added `ButtonMode` here to allow us to put our button centered on the position of our rect, or if set to `::Raw` it will get rendered from the top left corner.
lastly we set `initialized` to `true` .

```cpp
// mainmenu.cpp
void DrawMenu(MainMenu* mainmenu, SDL_Renderer* renderer, Sprite* spriteBuffer) {
  Sprite* background = GetSprite(SPRITE_ID::titlescreen_background, spriteBuffer);
  float scale = (SCREEN_HEIGHT / ((float)background->height * UPSCALE_FACTOR));
  RenderSprite_World(GetSprite(SPRITE_ID::titlescreen_background, spriteBuffer), renderer, NULL, SCREEN_WIDTH / 2.0, SCREEN_HEIGHT / 2.0, scale);
  for (int i = 0; i < mainmenu->activeButtonCount; i++) {
    Button* button = mainmenu->activeButtons[i];
    RenderButton(mainmenu->activeButtons[i], i == mainmenu->activeButtonIndex, renderer);
  }
}
```

first we're rendering our titlescreen background. We've made some changes to it in `spriteLibrary.cpp`

```cpp
// spriteLibrary.cpp
{SPRITE_ID::titlescreen_background, "assets/sprites/titlescreen.png"},
```

we've opted to omit the pivot as we want our program to automatically set this to the dead center of the texture.
We pass `NULL` as our parameter for our `const Camera* camera` when calling `RenderSprite_World()` this will currently break but we can easily allow for this behaviour by making a small adjustment to the function

```cpp
// rendering.cpp
void RenderSprite_World(SpriteRenderInfo spriteRenderInfo, SDL_Renderer* renderer, const Camera* camera, float x, float y, float scale, float alpha, bool flipped) {
  // code hidden for brevity
  if(GetSpriteCount(sprite) > 1) {
    // hidden for brevity
  }
  else {
    // hidden for brevity
  }
  // old
  SDL_FRect rect;
  rect.x = x;
  rect.y = y;
  float final_scale = UPSCALE_FACTOR * scale;
  rect.w = tilesetRect.w * final_scale;
  rect.h = tilesetRect.h * final_scale;
  rect.x -= sprite->pivot_x * final_scale;
  rect.y -= sprite->pivot_y * final_scale;
  // new if-statement with old code inside
  if(camera != NULL) {
    rect.x -= camera->camera_x;
    rect.y -= camera->camera_y;
  }
  // code hidden for brevity
}
```

We just make sure that if camera was `NULL` we don't try and use it to adjust our rect's position. Now we can safely pass `NULL` instead of our camera.
Back in `DrawMenu()` We create a scale variable that we'll use to scale the texture up and down depending on the size of the game window `SCREEN_HEIGHT` . This means that the larger the difference is between the game window and the height of the texture the more scale increases. This ensures that the titlescreen background fills the entire window.
When we call `RenderSprite_World()` we also pass along `SCREEN_WIDTH/HEIGHT / 2.0` to place the rendering origin in the center of the screen. Doing this with the pivot set to the center of the texture is what we need to do to center the entire thing.

lastly we loop over each of the `activeButtons` and call `RenderButton` by passing it along. `i == mainmenu->activeButtonIndex` will be true only for the active button and false for all others. Letting us add the light grey overlay color on all non selected buttons.

```cpp
// mainmenu.cpp
// updateMenu part 1
void UpdateMenu(GameData* data) {
  MainMenu* mainmenu = &data->scenes.mainMenu;
  Input* input = &data->input;
  mainmenu->activeButtonCount = GetActiveButtonCount(mainmenu->buttons, mainmenu->button_count);
  if(mainmenu->activeButtonCount == 0) {
    return;
  }
  mainmenu->activeButtons = ALLOC_ARRAY(data->arena_scratch, Button*, mainmenu->activeButtonCount);
  int index = 0;
  for (int i = 0; i < mainmenu->button_count; i++) {
    Button* button = &mainmenu->buttons[i];
    if(button->is_active) {
      mainmenu->activeButtons[index] = button;
      index += 1;
    }
  }
  int* buttonIndex = &mainmenu->activeButtonIndex;
  bool anyHoveredOver = false;
  bool mouseMoving = input->mouse_magnitude > 0.1;
  if(mouseMoving) {
    for (int i = 0; i < mainmenu->activeButtonCount; i++) {
      Button* button = mainmenu->activeButtons[i];
      if(IsHoveredOver(button, input->mouse_x, input->mouse_y)) {
        anyHoveredOver = true;
        *buttonIndex = i;
        break;
      }
    }
  }
  // ...
}
```

passing the entire `GameData*` is currenly necessary as we need to pass that same fat struct into `PressButton` . Fixing this would mean actually commiting to the variables that `PressButton` needs and then walking up this call-chain fixing so we only send those variables as parameters. For now we'll pass everything
we collect `Input*` and `MainMenu*` pointers to reduce the length of each line of code that references them. We then fetch how many active buttons we have and allocate our pointer pointer array of `activeButtons` to our `scratch arena` . This arena gets reset each frame (on purpose).
We then loop over all buttons and assign the active ones in order to our `activeButtons` array.
Next we're checking if any button is hovered over and if the mouse is currently in motion. To do this we need to expand our `Input` struct and add some new boilerplate to `main.cpp` if the mouse is in fact moving we loop over all buttons and if any of them are hovered over we set `anyHoveredOver` to `true` and set the `activeButtonIndex` to the value of `i` . This is how we get the relevant button for mouse inputs.

```cpp
// input.h
struct Input {
  const bool* keys_current;
  const bool* keys_previous;
  float* keys_held_time;
  SDL_MouseButtonFlags mouse_current;
  SDL_MouseButtonFlags mouse_previous;
  float* mouse_held_time;
  float mouse_x;
  float mouse_y;
  float mouse_x_delta;
  float mouse_y_delta;
  double mouse_magnitude;
};
```

```cpp
// main.cpp
gameData->input.keys_current = SDL_GetKeyboardState(nullptr);
float* delta_x = &gameData->input.mouse_x_delta;
float* delta_y = &gameData->input.mouse_y_delta;
*delta_x = gameData->input.mouse_x;
*delta_y = gameData->input.mouse_y;
gameData->input.mouse_current = SDL_GetMouseState(&gameData->input.mouse_x, &gameData->input.mouse_y);
*delta_x = gameData->input.mouse_x - *delta_x;
*delta_y = gameData->input.mouse_y - *delta_y;
float dx = *delta_x;
float dy = *delta_y;
gameData->input.mouse_magnitude = std::sqrt(dx * dx + dy * dy);
dll.update(gameData, dt);
```

we fetch pointers to `mouse_x/y_delta` . A delta value is the value as a comparison between the current frame and the previous one. We already use this for `deltatime` . So we assign `delta_x/y` to the value from `mouse_x/y` . But we do this before we call `GetMouseState` for this frame, meaning that the values stored are the last frames values. We then update our `delta_x/y` to be the current position of the mouse minus last frames position. This gives us the length the mouse travelled in x and y since the last frame. After that we call `std::sqrt` that gives us a vector in space, the length of this is the total distance travelled in both the x and y direction, this is the pythagorean theorem.
I've saved two `printf` calls that you can uncomment if you want to have a look at what happens when you move the mouse around.

```cpp
// mainmenu.cpp
// UpdateMenu() part 2
bool up = KeyPressed(input, SDL_SCANCODE_UP);
bool down = KeyPressed(input, SDL_SCANCODE_DOWN);
if(up || down) {
  int direction = up ? 1 : -1;
  *buttonIndex += direction + mainmenu->activeButtonCount;
  *buttonIndex = *buttonIndex % mainmenu->activeButtonCount;
}
Button* selected = mainmenu->activeButtons[*buttonIndex];
if(selected != nullptr) {
  if(KeyPressed(input, SDL_SCANCODE_RETURN)) {
    PressButton(mainmenu->activeButtons[*buttonIndex], data);
    return;
  }
}
if(IsHoveredOver(selected, input->mouse_x, input->mouse_y)) {
  if(MousePressed(input, MouseButtons::LEFT)) {
    PressButton(selected, data);
    return;
  }
}
```

Now that we can select a button using the mouse we can check if we are pressing the up or down key. If so we modify the `buttonIndex` pointer that points to the same place in memory as our `mainmenu->activeButtonIndex` . Instead of having two if-statements we use the `?` operator to select if we're going backwards or forwards. But with the Modulo `%` allowing for negative numbers we actually have to shift the index by the full count of `activeButtonCount` this does nothing to change the value as this full value addition will get stripped by the modulo operator - but it will ensure that even when we remove using `direction = -1` we never reach a negative number.
we then point to the button that is selected from `buttonIndex` and store it in `selected` .
If we have a selected we check if we are pressing enter aka `return` or if we are hovering over the button and pressing left mouse button. In either case we call `PressButton` .
With this we've added buttons and the collision and pressing of said buttons!