# 37 Parallax

Parallax is the effect of things further away not moving out of your field of view as fast as objects that are closer to you. Hold out a finger and slide your head from left to right. Your finger will move more relative to your head than the wall behind it.
We use Parallax to produce depth in 2D scenes. This is featured prominently in 2D sidescrollers. We'll be replicating the effect found in a game called Arco for our main menu.

in the `chapter 38 assets.zip` you'll find five images that when stacked ontop of each other form our final main menu. Please add these to our `assets/sprites` folder.
We'll be referencing these in our `mainmenu.h/.cpp` as well as our `spriteLibrary.h/.cpp`

```cpp
// spriteLibrary.h
enum class SPRITE_ID {
  // other enums hidden for brevity
  Menu_Horizon,
  Menu_Cloud_Back,
  Menu_Cloud_Front,
  Menu_Middle,
  Menu_Front
};
```

```cpp
// spriteLibrary.cpp
static const SpriteDataEntry all_sprite_data[] = {
  // other SpriteDataEntries hidden for clarity
  {SPRITE_ID::Menu_Horizon, "assets/sprites/mainmenu_background.png"},
  {SPRITE_ID::Menu_Cloud_Back, "assets/sprites/mainmenu_cloud_back.png"},
  {SPRITE_ID::Menu_Cloud_Front, "assets/sprites/mainmenu_cloud_front.png"},
  {SPRITE_ID::Menu_Middle, "assets/sprites/mainmenu_middle.png"},
  {SPRITE_ID::Menu_Front, "assets/sprites/mainmenu_front.png"},
};
```

```cpp
// mainmenu.h
struct MainMenu {
  Button* buttons;
  int button_count;
  int activeButtonIndex;
  Button** activeButtons;
  int activeButtonCount;
  bool initialized;
  Sprite* background_horizon;
  Sprite* background_cloud_back;
  Sprite* background_cloud_front;
  Sprite* background_middle;
  Sprite* background_front;
};

void DrawMenu(MainMenu* mainmenu, SDL_Renderer* renderer, Sprite* spriteBuffer, Input* input);
```

We add references to these five partial sprites in `MainMenu` and we will also be needing the mouse position to control the parallax effect in the main menu. To do that we've added `Input* input` to our `DrawMenu()` function.
this means that we have to update the callsite in `game.cpp`

```cpp
// game.cpp
// in DrawScene()
case SCENE_TYPES::MAINMENU:
  DrawMenu(&data->scenes.mainMenu, renderer, data->spriteBuffer, &data->input);
  break;
```

With this we can add the necessary logic to `mainmenu.cpp`

```cpp
// mainmenu.cpp
void InitializeMenu(MainMenu* mainmenu, Sprite* spriteBuffer, Memory::Arena* arena_main) {
  assert(mainmenu->initialized == false);
  mainmenu->button_count = 2;
  mainmenu->buttons = ALLOC_ARRAY(arena_main, Button, mainmenu->button_count);
  SetupButton(&mainmenu->buttons[0], ButtonType::START_GAME, spriteBuffer, {SCREEN_WIDTH / 2.0, SCREEN_HEIGHT / 2.0, 200, 80}, ButtonMode::Centered);
  SetupButton(&mainmenu->buttons[1], ButtonType::QUIT, spriteBuffer, {SCREEN_WIDTH / 2.0, (SCREEN_HEIGHT / 2.0) + 100, 200, 80}, ButtonMode::Centered);
  // new
  mainmenu->background_horizon = GetSprite(SPRITE_ID::Menu_Horizon, spriteBuffer);
  mainmenu->background_cloud_back = GetSprite(SPRITE_ID::Menu_Cloud_Back, spriteBuffer);
  mainmenu->background_cloud_front = GetSprite(SPRITE_ID::Menu_Cloud_Front, spriteBuffer);
  mainmenu->background_middle = GetSprite(SPRITE_ID::Menu_Middle, spriteBuffer);
  mainmenu->background_front = GetSprite(SPRITE_ID::Menu_Front, spriteBuffer);
  mainmenu->initialized = true;
}
```

we fetch the relevant sprites.

```cpp
// mainmenu.cpp
void DrawMenu(MainMenu* mainmenu, SDL_Renderer* renderer, Sprite* spriteBuffer, Input* input) {
  float scale = (SCREEN_HEIGHT / ((float)mainmenu->background_horizon->height * UPSCALE_FACTOR));
  scale *= 1.2;
  float mouse_x = input->mouse_x;
  float mouse_y = input->mouse_y;
  float center_x = SCREEN_WIDTH / 2.0;
  float center_y = SCREEN_HEIGHT / 2.0;
  float offset_x = center_x - mouse_x;
  float offset_y = center_y - mouse_y;
  RenderSprite_World(GetSprite(SPRITE_ID::Menu_Horizon, spriteBuffer), renderer, NULL, center_x, center_y, scale);
  RenderSprite_World(GetSprite(SPRITE_ID::Menu_Cloud_Back, spriteBuffer), renderer, NULL, center_x + (offset_x / 11), center_y + (offset_y / 11), scale);
  RenderSprite_World(GetSprite(SPRITE_ID::Menu_Cloud_Front, spriteBuffer), renderer, NULL, center_x + (offset_x / 9), center_y + (offset_y / 9), scale);
  RenderSprite_World(GetSprite(SPRITE_ID::Menu_Middle, spriteBuffer), renderer, NULL, center_x + (offset_x / 7), center_y + (offset_y / 7), scale);
  RenderSprite_World(GetSprite(SPRITE_ID::Menu_Front, spriteBuffer), renderer, NULL, center_x + (offset_x / 5), center_y + (offset_y / 5), scale);
  for (int i = 0; i < mainmenu->activeButtonCount; i++) {
    Button* button = mainmenu->activeButtons[i];
    RenderButton(mainmenu->activeButtons[i], i == mainmenu->activeButtonIndex, renderer);
  }
}
```

We then refactor our `DrawMenu()` . We now fetch the x/y of our mouse then calculate how far the mouse is from the dead center of the game window. With this offset we can render all of our menu art pieces at the center of the screen plus a portion of this offset. The sprites that we render first get offset by a smaller value than the objects closer to us. You can try and reverse this order for a different effect all together.
We also multiply scale by 1.2 as we need to be zoomed in a little to avoid our sprites cutting of at the edge of the 1920x1080 window. You can try and remove the scale multiplication and watch as we reach the edge of our sprites.
This is the basics of parallax. We adjust the position of something in relationship to some movement of a camera based on how far away from the camera it is. Currently we just divide the offset by a value that will represent the depth of the object. The larger the fraction the further away the layer is. And no fraction at all means that it's at the horizon-distance.
That's it!