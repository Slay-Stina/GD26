# 39 Buttons Part II

we could always use a fixed-size bespoke button texture. But when the text that we overlay ontop of the button has different sizes we don't want our text to go outside of the bounds of the button texture. We could always make the text smaller. But a much more established way is to introduce nine-slicing . This means that we create our buttons as a 3x3 atlas . We then render each of the four corners at the appropriate positions then stretch the sprites between the corners until they fill the entire space.
This is not really "difficult" it just requires us to be a bit extra alert when programming the position code, there is a bunch of small equations to get the actual sizes and positions.
When creating this chapter I had to do quite a bit of testing to get things right. Just knowing how to write this without any hiccups is not how code like this usually goes.
We'll be adding a new variable to our `Button` struct

```cpp
// button.h
struct Button {
  ButtonType type;
  SDL_FRect rect;
  Sprite* sprite;
  bool is_active;
  bool is_dynamic;
};
```

Previously our `Button` struct held a `SDL_Texture*` for its texture, but we've refactored this to be a `Sprite*` instead.
we use `is_dynamic` to control if we want to render our button using the normal method or a new render function we'll add to `rendering.h`

```cpp
// rendering.h
void RenderButton_Dynamic(Button* button, bool is_selected, SDL_Renderer* renderer);
```

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
    if(button->is_dynamic) {
      RenderButton_Dynamic(button, i == mainmenu->activeButtonIndex, renderer);
    }
    else {
      RenderButton(button, i == mainmenu->activeButtonIndex, renderer);
    }
  }
  mainmenu->activeButtonCount = 0;
}
```

We're also fixing a bug in our earlier code. When we press Start Game we stop calling `UpdateMenu` (on purpose). But the `activeButtonCount` and `activeButtons` are calculated in `Update`. This means that our `activeButtons` array when we stop calling `Update`s points at garbage memory. By setting `activeButtonCount` to 0 after we are done with a `Draw` call we can at least know that if we never call `Update` again then we will at least not have any value here to cause a loop through the garbage array. This is not a robust fix, just a temporary measure so we can focus on what we're working on.
In `SetupButton()` we're assigning `is_dynamic` to our Start Game button.

```cpp
// mainmenu.cpp
void InitializeMenu(MainMenu* mainmenu, Sprite* spriteBuffer, Memory::Arena* arena_main) {
  assert(mainmenu->initialized == false);
  mainmenu->button_count = 2;
  mainmenu->buttons = ALLOC_ARRAY(arena_main, Button, mainmenu->button_count);
  SetupButton(&mainmenu->buttons[0], ButtonType::START_GAME, spriteBuffer, {SCREEN_WIDTH / 2.0, SCREEN_HEIGHT / 2.0, 400, 170}, Alignment::Centered);
  mainmenu->buttons[0].is_dynamic = true;
  SetupButton(&mainmenu->buttons[1], ButtonType::QUIT, spriteBuffer, {SCREEN_WIDTH / 2.0, (SCREEN_HEIGHT / 2.0) + 100, 200, 80}, Alignment::Centered);
  mainmenu->background_horizon = GetSprite(SPRITE_ID::Menu_Horizon, spriteBuffer);
  mainmenu->background_cloud_back = GetSprite(SPRITE_ID::Menu_Cloud_Back, spriteBuffer);
  mainmenu->background_cloud_front = GetSprite(SPRITE_ID::Menu_Cloud_Front, spriteBuffer);
  mainmenu->background_middle = GetSprite(SPRITE_ID::Menu_Middle, spriteBuffer);
  mainmenu->background_front = GetSprite(SPRITE_ID::Menu_Front, spriteBuffer);
  mainmenu->activeButtonIndex = 0;
  mainmenu->initialized = true;
}
```

With the change from `SDL_Texture*` to `Sprite*` we need to update our callsites that used the `SDL_Texture` . This was only two functions `SetupButton()` in `button.cpp` and `RenderButton` in `rendering.cpp` .

```cpp
void SetupButton(Button* button, ButtonType type, Sprite* spriteBuffer, SDL_FRect rect, Alignment mode) {
  assert(type != ButtonType::NONE);
  button->type = type;
  button->rect = rect;
  if(mode == Alignment::Centered) {
    button->rect.x -= button->rect.w / 2;
    button->rect.y -= button->rect.h / 2;
  }
  button->is_active = true;
  switch(button->type) {
    case ButtonType::START_GAME:
      button->sprite = GetSprite(SPRITE_ID::Button_Basic, spriteBuffer);
      break;
    case ButtonType::QUIT:
      button->sprite = GetSprite(SPRITE_ID::Fallback, spriteBuffer);
      break;
    default:
      button->sprite = GetSprite(SPRITE_ID::Fallback, spriteBuffer);
      break;
  }
}
```

We also make sure that our `START_GAME` button uses our `Button_Basic` `SPRITE_ID` .

```cpp
// rendering.cpp
void RenderButton(Button* button, bool is_selected, SDL_Renderer* renderer) {
  SDL_Texture* texture = button->sprite->texture;
  SDL_SetTextureScaleMode(texture, SDL_SCALEMODE_PIXELART);
  uint8_t colorOverlay = is_selected ? 255 : 230;
  SDL_SetTextureBlendMode(texture, SDL_BLENDMODE_BLEND);
  SDL_SetTextureColorMod(texture, colorOverlay, colorOverlay, colorOverlay);
  SDL_RenderTexture(renderer, button->sprite->texture, NULL, &button->rect);
}
```

We are adding a new sprite to our assets folder

```cpp
// spriteLibrary.h
enum class SPRITE_ID {
  // other SPRITE_IDs hidden for clarity
  Button_Basic
};
```

this is our 66x66 pixel square that we'll cut up to create our dynamic buttons

```cpp
// spriteLibrary.cpp
const char* FALLBACK_PATH = "assets/sprites/fallback.png";

static const SpriteDataEntry all_sprite_data[] = {
  // other SpriteDataEntry hidden for clarity
  {SPRITE_ID::Button_Basic, "assets/sprites/basic_button.png", 0, 0, 3, 3},
};
```

Lets look at the pretty massive `RenderButton_Dynamic()`

```cpp
// rendering.cpp
// RenderButton_Dynamic Part 1
void RenderButton_Dynamic(Button* button, bool is_selected, SDL_Renderer* renderer) {
  assert(button->sprite->sprite_count_x == 3);
  assert(button->sprite->sprite_count_y == 3);
  uint8_t colorOverlay = is_selected ? 255 : 230;
  SDL_Texture* texture = button->sprite->texture;
  SDL_FRect rect = button->rect;
  // ...
}
```

we make sure that we're working with a nine-slice sprite using two asserts . We then fetch the `texture*` and `rect` from the `button->sprite` to make the callsites shorter. We also prepare our `colorOverlay` to darken the button if it is not the selected one.

```cpp
// rendering.cpp
// RenderButton_Dynamic Part 2
float part_w = texture->w / 3.0;
float part_h = texture->h / 3.0;
float vertical_center_height = rect.h - (part_h * 2);
float horizontal_center_width = rect.w - (part_w * 2);
float right_x = rect.x + rect.w - part_w;
float bottom_y = rect.y + rect.h - part_h;
float center_y = rect.y + part_h;
float center_x = rect.x + part_w;
```

as we know our texture is a 3x3 atlas we can take the total width and height of the texture and divide it by 3. Then we get the size of one of the atlas pieces.
We then remove the size of two of these parts from the width and height of the buttons own size. This will give us the length of the stretching segments that will be used to fill the space between the corners of the button.
We'll be creating two pairs of `SDL_FRects` one list of `dst` aka destinations. These are the rects we'll render our texture into on the screen. and `src` aka source. These are the rects in our atlas texture . We'll be rendering 9 textures in this one function to reconstruct the entire button.
the `right_x`, `bottom_y` etc variables are used to calculate the x and y position of the different parts of the button.

```cpp
// rendering.cpp
// RenderButton_Dynamic Part 3
SDL_FRect topLeftdst     = {rect.x,     rect.y,        part_w,                    part_h};
SDL_FRect topRightdst    = {right_x,    rect.y,        part_w,                    part_h};
SDL_FRect topCenterdst   = {center_x,   rect.y,        horizontal_center_width,   part_h};
SDL_FRect bottomLeftdst  = {rect.x,     bottom_y,      part_w,                    part_h};
SDL_FRect bottomRightdst = {right_x,    bottom_y,      part_w,                    part_h};
SDL_FRect bottomCenterdst = {center_x,  bottom_y,      horizontal_center_width,   part_h};
SDL_FRect centerLeftdst  = {rect.x,     center_y,      part_w,                    vertical_center_height};
SDL_FRect centerRightdst = {right_x,    center_y,      part_w,                    vertical_center_height};
SDL_FRect centerdst      = {center_x,   center_y,      horizontal_center_width,   vertical_center_height};
```

I've aligned each parameter to make it easier to get an overview. Each of these 9 rects refer to a portion of the button that we'll be reconstructing on the screen. For example the `bottomRightdst` has its x position set to `right_x` , its y to `bottom_y` then its width and height is equal to `part_w/h` . We can see how only the stretching parts deviate from the `part_w/h` when it comes to setting the width and height . Try and intuit where the piece will be in the 3x3 grid and use the parameters to figure out how it would look.

```cpp
// rendering.cpp
// RenderButton_Dynamic Part 4
SDL_FRect topLeftsrc     = {0,          0,          part_w, part_h};
SDL_FRect topRightsrc    = {part_w * 2, 0,          part_w, part_h};
SDL_FRect topCentersrc   = {part_w * 1, 0,          part_w, part_h};
SDL_FRect bottomLeftsrc  = {0,          part_h * 2, part_w, part_h};
SDL_FRect bottomRightsrc = {part_w * 2, part_h * 2, part_w, part_h};
SDL_FRect bottomCentersrc = {part_w * 1, part_h * 2, part_w, part_h};
SDL_FRect centerLeftsrc  = {0,          part_h * 1, part_w, part_h};
SDL_FRect centerRightsrc = {part_w * 2, part_h * 1, part_w, part_h};
SDL_FRect centersrc      = {part_w * 1, part_h * 1, part_w, part_h};
```

Because our `src` rectangles are not in the world itself it's easier to just select the appropriate square in the 3x3 grid by shifting along the x and y using `part_w/h` .

```cpp
// rendering.cpp
// RenderButton_Dynamic Part 5
SDL_SetTextureScaleMode(texture, SDL_SCALEMODE_PIXELART);
SDL_SetTextureBlendMode(texture, SDL_BLENDMODE_BLEND);
SDL_SetTextureColorMod(texture, colorOverlay, colorOverlay, colorOverlay);
```

we really shouldn't be calling these each time but rather cache these settings when we create the `Sprite` in `spriteLibrary.cpp` . But we can worry about that later. These settings will help with adding the grey color and not getting blurry art when we scale the pixel art between the corners.

```cpp
// rendering.cpp
// RenderButton_Dynamic Part 6
SDL_RenderTexture(renderer, texture, &topLeftsrc,     &topLeftdst);
SDL_RenderTexture(renderer, texture, &topCentersrc,   &topCenterdst);
SDL_RenderTexture(renderer, texture, &bottomCentersrc, &bottomCenterdst);
SDL_RenderTexture(renderer, texture, &centerLeftsrc,  &centerLeftdst);
SDL_RenderTexture(renderer, texture, &centerRightsrc, &centerRightdst);
SDL_RenderTexture(renderer, texture, &bottomLeftsrc,  &bottomLeftdst);
SDL_RenderTexture(renderer, texture, &topRightsrc,    &topRightdst);
SDL_RenderTexture(renderer, texture, &bottomRightsrc, &bottomRightdst);
SDL_RenderTexture(renderer, texture, &centersrc,      &centerdst);
```

Finally we take the `src` and `dst` pairs and render the atlas to the screen in 9 steps. Each one responsible for one of the 3x3 squares. You can comment out these `RenderTexture()` calls to see what part of the button the draw.
once you've unzipped `chapter 40 assets.zip` you can run the game and look at our non-blurry non-bad-scaled start button!
The next step will be adding text ontop of the button

```cpp
// button.h
struct Button {
  ButtonType type;
  SDL_FRect rect;
  Sprite* sprite;
  bool is_active;
  bool is_dynamic;
  FontAtlas* font;
  const char* text;
};
```

first lets put `STOP_CHAR` into `common.h` from `rendering.cpp` along with a small helper function.

```cpp
// common.h
static const char STOP_CHAR = '\0';

inline bool IsStringEmpty(const char* str) {
  return str == nullptr || str[0] == STOP_CHAR;
}
```

we'll use this to check if we've actually added any text to the button before we try and render this text.
At the bottom of `RenderButton()` and `RenderButton_Dynamic()` we'll add a call to `RenderText()`

```cpp
// rendering.cpp
if(!IsStringEmpty(button->text)) {
  float glyph_height = button->font->glyphs['H'].atlasPosition.h / 2.0;
  RenderText(button->font, button->text, renderer, nullptr, rect.x + (rect.w / 2.0), rect.y + (rect.h / 2.0) - glyph_height, Alignment::Centered);
}
```

we do some short equations to find the center of the button. Then we shift the text upwards by half the height of a "random" uppercase letter.
Lets refactor our `SetupButton()` to accept `FontAtlas*` and `const char*` as optional parameters

```cpp
// button.h
void SetupButton(Button* button,
  ButtonType type,
  Sprite* spriteBuffer,
  SDL_FRect rect,
  Alignment mode,
  FontAtlas* font = nullptr,
  const char* text = nullptr);
```

the need to put the parameters on different lines to avoid the line becoming very very long is a sign that the function accepts to many variables. But that is not in itself a sign that we need to change it. But we should keep an eye on it

```cpp
// button.cpp
void SetupButton(Button* button, ButtonType type, Sprite* spriteBuffer, SDL_FRect rect, Alignment mode, FontAtlas* font, const char* text) {
  assert(type != ButtonType::NONE);
  button->type = type;
  button->rect = rect;
  if(mode == Alignment::Centered) {
    button->rect.x -= button->rect.w / 2;
    button->rect.y -= button->rect.h / 2;
  }
  button->is_active = true;
  switch(button->type) {
    case ButtonType::START_GAME:
      button->sprite = GetSprite(SPRITE_ID::Button_Basic, spriteBuffer);
      break;
    case ButtonType::QUIT:
      button->sprite = GetSprite(SPRITE_ID::Fallback, spriteBuffer);
      break;
    default:
      button->sprite = GetSprite(SPRITE_ID::Fallback, spriteBuffer);
      break;
  }
  bool hasText = !IsStringEmpty(text);
  if(font == nullptr) {
    assert(!hasText);
  }
  if(hasText) {
    assert(font != nullptr);
  }
  button->font = font;
  button->text = text;
}
```

We do two asserts to ensure that both the font and the text was set if either was assigned. Then we assign `button->font/text` to the supplied parameters.
To have our font available at the callsite in `InitializeMenu()` we need to add it as a parameter

```cpp
// mainmenu.h
void InitializeMenu(MainMenu* mainmenu, Sprite* spriteBuffer, FontAtlas* font, Memory::Arena* arena_main);
```

```cpp
// mainmenu.cpp
void InitializeMenu(MainMenu* mainmenu, Sprite* spriteBuffer, FontAtlas* font, Memory::Arena* arena_main) {
  assert(mainmenu->initialized == false);
  mainmenu->button_count = 2;
  mainmenu->buttons = ALLOC_ARRAY(arena_main, Button, mainmenu->button_count);
  SetupButton(&mainmenu->buttons[0], ButtonType::START_GAME, spriteBuffer, {SCREEN_WIDTH / 2.0, SCREEN_HEIGHT / 2.0, 300, 110}, Alignment::Centered, font, "Start Game");
  mainmenu->buttons[0].is_dynamic = true;
  SetupButton(&mainmenu->buttons[1], ButtonType::QUIT, spriteBuffer, {SCREEN_WIDTH / 2.0, (SCREEN_HEIGHT / 2.0) + 200, 200, 80}, Alignment::Centered, font, "Quit");
  mainmenu->background_horizon = GetSprite(SPRITE_ID::Menu_Horizon, spriteBuffer);
  mainmenu->background_cloud_back = GetSprite(SPRITE_ID::Menu_Cloud_Back, spriteBuffer);
  mainmenu->background_cloud_front = GetSprite(SPRITE_ID::Menu_Cloud_Front, spriteBuffer);
  mainmenu->background_middle = GetSprite(SPRITE_ID::Menu_Middle, spriteBuffer);
  mainmenu->background_front = GetSprite(SPRITE_ID::Menu_Front, spriteBuffer);
  mainmenu->activeButtonIndex = 0;
  mainmenu->initialized = true;
}
```

and add it when calling `InitializeMenu()`

```cpp
// game.cpp
void Initialize(GameData* data, SDL_Window* window, SDL_Renderer* renderer) {
  *data->ticks_total = 0;
  DEV::Initialize(window, renderer);
  InitializeAudioSystem(&data->audio, data->arena_main);
  AssetManagement::LoadAllSFX(&data->audio);
  AssetManagement::LoadAllSprites(data->spriteBuffer, renderer);
  data->imGui_context = ImGui::GetCurrentContext();
  AssetManagement::LoadFont(renderer, "assets/fonts/ByteBounce.ttf", &data->font, 48);
  AssetManagement::LoadAllTilesets(data->tilesetBuffer, data->arena_images);
  SDL_Texture* blackfade = GetSprite(SPRITE_ID::black_1x1, data->spriteBuffer)->texture;
  SDL_SetTextureBlendMode(blackfade, SDL_BLENDMODE_BLEND);
  InitializeGame(&data->scenes.gameplay, data->arena_levels, data->tilesetBuffer);
  InitializeMenu(&data->scenes.mainMenu, data->spriteBuffer, &data->font, data->arena_main);
  PlaySong(SONG_ID::THEME);
  ChangeScene(data, SCENE_TYPES::MAINMENU);
}
```

Now we can give a button text to render. That's cool!
