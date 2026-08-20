# 38 Text

We can of course already render text if each time we want to do so we just create a bespoke sprite and use that as a text-proxy. But this is not a good way of doing it an neither is it industry standard. What we'll be doing is rendering text one character at a time by taking a font and converting it to a `SDL_Texture` .
The process will be us creating what is called a Texture Atlas a texture atlas is similar to a spritesheet because it has multiple individual things all layed out in a larger grid.
To convert a font into a texture we need to work with some external library (or spend a lot of effort coding our own). The easiest solution for us it to download `SDL_TTF` a library that helps us work with TTF files. a TTF file is a True Text Font . This is the same file format used by all your text-displaying software.
the `SDL_TTF` github is at: https://github.com/libsdl-org/SDL_ttf after navigating to Releases we're downloading `SDL3_ttf-devel-3.2.2-VC.zip`

> [!NOTE]
> On Linux, you can install SDL_ttf via your package manager. For Arch Linux: `sudo pacman -S sdl3_ttf`. For Debian/Ubuntu: `sudo apt install libsdl3-ttf-dev`. The `.so` files and headers will be installed to system paths, so you may not need to manually copy them. You'll need to link against `SDL3_ttf::SDL3_ttf` in CMake.

after having unziped our file we will find the `SDL_textengine.h` and `SDL_ttf.h` in `include` and add it to our own. I've opted to put these two .h files into their own subdirectory inside `include` that I've named `SDL_TTF` .
we also need `libSDL3_ttf.so` and `libSDL3_ttf.a` (on Linux). These are going into their own subdirectory inside our `lib` folder. I've named their subdirectory `SDL_TTF` just as I did for our `include` folder.
Thankfully this will work out of the gate requiring no adjustments to our `cmakelists.txt` .

For this chapter I've downloaded a font called ByteBounce from: https://www.1001fonts.com/bytebounce-font.html . I've put this `ByteBounce.ttf` file into a new subdirectory `assets/fonts` . We'll be using this path to load it later.
We want to store our text in a fashion to save us from having to re-create a texture each frame that we want to show text to the screen - that would be terribly slow.
We need to initialize SDL_TTF if we miss this step then even if we code everything correct afterwards then nothing will show up on our screen.

```cpp
// main.cpp
void SDL_Setup() {
  SDL_Init(SDL_INIT_EVENTS);
  SDL_SetLogPriorities(SDL_LOG_PRIORITY_VERBOSE);
  TTF_Init();
}

window = SDL_CreateWindow("hell of a time", SCREEN_WIDTH, SCREEN_HEIGHT, 0);
renderer = SDL_CreateRenderer(window, NULL);
```

in a new `fontLibrary.h` we'll add structs to hold individual letters that we call glyphs when working with fonts as well as a struct to hold our font atlas

```cpp
// fontLibrary.h
#pragma once
#include <SDL3/SDL.h>

struct Glyph {
  SDL_FRect atlasPosition;
};

struct FontAtlas {
  SDL_Texture* atlasTexture;
  static const int GLYPH_COUNT = 128;
  Glyph glyphs[GLYPH_COUNT];
};

namespace AssetManagement {
  void LoadFont(SDL_Renderer* renderer, const char* font_path, FontAtlas* fontAtlas, float ptsize);
}
```

`Glyph` just has a `SDL_FRect` inside of it now, meaning that we could ignore it and just work the the `SDL_FRect` struct directly. But I find the `Glyph` struct easy to understand at a glance and we might b adding more variables to this struct later. If you wish you can ignore creating the `Glyph` struct.
`FontAtlas` has a pointer to a `SDL_Texture` we'll be creating this texture at runtime. We're also working with an array of a known size. So we can specify the size as a `static const int` meaning that the value can't change and we'll be able to access this number by accessing the struct type itself.
Finally we create an array of `Glyph`s with the specified size. the 128 size will be enough to fetch all common numbers, letters and symbols on our keyboard. There is more to font handling and especially when it comes to localization. But for our needs and the font we've selected this will be plenty.
We've added a new function to our `AssetManagement` namespace . This will grab a font and construct the contents of a `FontAtlas` based on it.
With only one function in our .h file we have only one function to write in a `fontLibrary.cpp` . We could have inlined the function but we have not done so for other `xxxLibrary.h/.cpp` pairs so I'm opting for consistency.

```cpp
// fontLibrary.cpp
#include "fontLibrary.h"
#include <SDL3/SDL.h>
#include "SDL_TTF/SDL_ttf.h"
#include <cassert>

namespace AssetManagement {
  void LoadFont(SDL_Renderer* renderer, const char* font_path, FontAtlas* fontAtlas, float ptsize) {
    TTF_Font* font = TTF_OpenFont(font_path, ptsize);
    assert(font != nullptr);
    SDL_Color white = {255, 255, 255, 255};
    int atlas_size = 1024;
    SDL_Surface* atlas_surface = SDL_CreateSurface(atlas_size, atlas_size, SDL_PIXELFORMAT_RGBA32);
    int draw_point_x = 0;
    int draw_point_y = 0;
    int tallest_glyph_in_row = 0;
    int FIRST_RELEVANT_GLYPH = 32;
    for (int i = FIRST_RELEVANT_GLYPH; i < FontAtlas::GLYPH_COUNT; i++) {
      SDL_Surface* glyph_surface = TTF_RenderGlyph_Blended(font, i, white);
      if(glyph_surface == nullptr) {
        continue;
      }
      if(draw_point_x + glyph_surface->w > atlas_size) {
        draw_point_x = 0;
        draw_point_y += tallest_glyph_in_row;
        tallest_glyph_in_row = 0;
      }
      if(tallest_glyph_in_row < glyph_surface->h) {
        tallest_glyph_in_row = glyph_surface->h;
      }
      SDL_Rect glyph_position = {draw_point_x, draw_point_y, glyph_surface->w, glyph_surface->h};
      SDL_BlitSurface(glyph_surface, NULL, atlas_surface, &glyph_position);
      fontAtlas->glyphs[i].atlasPosition = {(float)glyph_position.x, (float)glyph_position.y, (float)glyph_position.w, (float)glyph_position.h};
      draw_point_x += glyph_surface->w;
      SDL_DestroySurface(glyph_surface);
    }
    fontAtlas->atlasTexture = SDL_CreateTextureFromSurface(renderer, atlas_surface);
    SDL_DestroySurface(atlas_surface);
    TTF_CloseFont(font);
  }
}
```

Because SDL_TTF and the base SDL create some textures in memory outside of our memory arena we need to remember to call `Destroy()` and `Close()` functions. If we forget to do this we will not be able to recover this piece of memory for the entire runtime of the program even though we no longer need the memory. If we continue to not mark memory as available in a program that does not use memory arena structure we can eventually run out of memory completely and crash the program. This type of crash can happen at any time and can be very hard to track down.
We start by loading our font using `TTF_OpenFont()` this will allow us to fetch the font data. We then set up a new `SDL_Surface` . A `SDL_Surface` lives on the CPU and a `SDL_Texture` lives in VRAM . This makes `SDL_Texture` much for flexible and faster. But `SDL_Surface` is how we intially represent a texture before doing a convertion. the float `ptsize` control how large the font will draw the individual glyphs.
So we create a new `SDL_Surface` and give it a size of 1024x1024. This surface will be the prototexture that we'll stamp each glyph onto. The goal is to stamp each glyph onto the surface in a grid, just like if it was a spritesheet.
`draw_point_x/y` will be the point on our surface that we will stamp the next glyph onto. As we continue stamping glyphs we'll move these two variables to ensure that each glyph has its own space on the spritesheet/atlas. `tallest_glyph_on_row` holds the height of the tallest glyph we've found since moving down to a new row. We need to shift down by (at least) this height in order to guarantee that the next row of glyph don't overlap with the once from the row above.
In a standard ascii table the glyphs from 0-31 are reserved for what is called control codes these are not meant to be rendered but instead used to help control hardware and data. We have no use for these can they can only cause headaches. So we start our for-loop at the 32'nd index instead. I put this in a named integer just to make the code self-explanatory .
We create the glyph as an `SDL_Surface` using `TTF_RenderGlyph_Blended` then sometimes a glyph can be missing from a font. In case it is `nullptr` we just continue to the next glyph.

```cpp
if(draw_point_x + glyph_surface->w > atlas_size) {
  draw_point_x = 0;
  draw_point_y += tallest_glyph_in_row;
  tallest_glyph_in_row = 0;
}
```

This if statement check if the stamp position along with the width of the glyph would go down beyond the right edge of our atlas . We it would we reset x back to the furthest left point aka 0 . We then shift the `draw_point_y` down by the height of the tallest glyph we've found so far. We then reset `tallest_glyph_in_row` as we're at the beginning of a new row.

```cpp
if(tallest_glyph_in_row < glyph_surface->h) {
  tallest_glyph_in_row = glyph_surface->h;
}
```

we check the current glyph against the tallest one we've found so far. And if the new glyph is taller we update our variable to reflect this.

```cpp
SDL_Rect glyph_position = {draw_point_x, draw_point_y, glyph_surface->w, glyph_surface->h};
SDL_BlitSurface(glyph_surface, NULL, atlas_surface, &glyph_position);
```

We then construct a `SDL_Rect` that will be the square we stamp the glyph onto. It has both the position and the size of the rect itself.
We then call `BlitSurface` , this takes the pixels from one `SDL_Surface` and adds them to another `SDL_Surface` at a specified position. the `NULL` value makes the program take the entire `glyph_surface` instead of a portion of it.

```cpp
fontAtlas->glyphs[i].atlasPosition = {(float)glyph_position.x, (float)glyph_position.y, (float)glyph_position.w, (float)glyph_position.h};
```

Our `fontAtlas` struct has our array of glyphs. We take the current index and ensure that it remembers this `glyph_position` . But you can see how we take each of the x,y,w,h and cast them to float this is because `atlasPosition` is a `SDL_FRect` instead of a `SDL_Rect` . We do this because the `SDL_Rendering` function we will use later demands that we work with `SDL_FRect` . Note how we're using the shorthand of `{}` to create the struct.

```cpp
draw_point_x += glyph_surface->w;
SDL_DestroySurface(glyph_surface);
```

As the last step of our for-loop we shift the `draw_point_x` the full width of the glyph. This will prepare the next loop of the for-loop to evaluate the next stamp position from just to the right of the last glyph.
`SDL_DestroySurface()` needs to be called now that we're done with this loop. We're never going to want to use this surface again, we've already stamped in onto our atlas .
We continue stamping glyphs until we've reached `GLYPH_COUNT` .

```cpp
fontAtlas->atlasTexture = SDL_CreateTextureFromSurface(renderer, atlas_surface);
SDL_DestroySurface(atlas_surface);
TTF_CloseFont(font);
```

After that we take the now finished `SDL_Surface` `atlas_surface` and using `CreateTextureFromSurface` we store a pointer to a VRAM-living texture in `atlasTexture` .

After that we destroy the now redundant `atlas_surface` and call `CloseFont()` to free all the memory on the heap that we used to run this function.
We can add an enum to help us manage more fonts in the future. But for this chapter we'll only have the one. We'll need to store this in our `GameData`

```cpp
// gameState.h
struct GameData {
  // other variables hidden for clarity
  FontAtlas font;
};
```

From `Initialize()` in `game.cpp` we'll call our `loadFont` function

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
  // ...
}
```

We make sure to set a large enough `ptsize` to actually see the glyphs clearly. 48 felt ok to me.
Rendering text will be a bit different. We'll add a bespoke function for it in `rendering.h/.cpp`

```cpp
// rendering.h
struct Button;
struct FontAtlas;

enum class Alignment {
  Right,
  Centered
};

void RenderText(FontAtlas* atlas, const char* text, SDL_Renderer* renderer, Camera* camera, const float x, const float y, Alignment alignment);
```

`Alignment` is the same enum that we used to call `ButtonMode` . I've removed `ButtonMode` from `button.h` and placed this new `Alignment` enum in `rendering.h` . I've also forward-declared `Button` and `FontAtlas` to avoid circular dependencies.
The refactoring of `ButtonMode` to `Alignment` and the move from `button.h` to `rendering.h` means that we need to update our callsites that previously used `ButtonMode::` . This happens in `mainmenu.cpp` and `game.cpp` and `button.cpp` . Pressing `space+D` will list all of the current errors from `clangd` . You should be able to find each callsite by going through this list.

```cpp
// rendering.cpp
static const char STOP_CHAR = '\0';

void RenderText(FontAtlas* atlas, const char* text, SDL_Renderer* renderer, Camera* camera, const float x, const float y, Alignment mode) {
  assert(atlas->atlasTexture != nullptr);
  float draw_position_x = x;
  float draw_position_y = y;
  if(camera != nullptr) {
    draw_position_x -= camera->camera_x;
    draw_position_y -= camera->camera_y;
  }
  if(mode == Alignment::Centered) {
    float totalWidth = 0;
    for (int i = 0; text[i] != STOP_CHAR; i++) {
      totalWidth += atlas->glyphs[text[i]].atlasPosition.w;
    }
    draw_position_x -= totalWidth / 2.0;
  }
  for (int i = 0; text[i] != STOP_CHAR; i++) {
    Glyph glyph = atlas->glyphs[text[i]];
    SDL_FRect renderRectangle = {draw_position_x, draw_position_y, glyph.atlasPosition.w, glyph.atlasPosition.h};
    SDL_RenderTexture(renderer, atlas->atlasTexture, &glyph.atlasPosition, &renderRectangle);
    draw_position_x += glyph.atlasPosition.w;
  }
}
```

A `char*` pointer array will always end with a `\0` this is a special character that tells us that we have reached the end of the array. I've opted to place this in a variable to make the code easier to read.
This function loops over all the individual letters in `text` and select the appropriate rectangle in our atlas to render before shifting the position by its width and rendering the next letter.
if our `Alignment` is set to `Centered` then we first loop over all glyphs and collect their combined width. We can then shift `draw_position_x` by half that to have the text centered. We are also allow our `Camera*` parameter being passed as `nullptr` . In that case we skip adjusting the `draw_position_x/y` by the camera offset.
our for loop responsible for rendering the text has `text[i] != STOP_CHAR` as its condition. This means that `i` will increase and the loops will continue until a `STOP_CHAR` has been reached. `char` also maps to `int` without any loss of information. This allows us to pass `text[i]` into our `glyphs` array to fetch the relevant `Glyph` struct - nifty!
Once we have the relevant glyph we use our `draw_position_x/y` along with the `glyph.atlasPosition.w/.h` to construct the position and size of the rectangle we will be drawing to the screen.
So we use the `glyph.atlasPosition` to find where on the `atlasTexture` our `Glyph` lives. Then we use `renderRectangle` to say where in the game we want to draw it.
Once we have drawn a `Glyph` we shift `draw_position_x` by its width .
This code could be further expanded to account for the text becoming to long and how to handle that. The most common case is shrinking it to fit or allowing the text to wrap down to a new line.
Now we can test our text rendering and write whatever we think is funny. I've put a temporary call to `RenderText` in `DrawScene()`

```cpp
// game.cpp
// in the switch case inside DrawScene()
case SCENE_TYPES::MAINMENU:
  DrawMenu(&data->scenes.mainMenu, renderer, data->spriteBuffer, &data->input);
  RenderText(&data->font, "hello sailor", renderer, &data->camera, SCREEN_WIDTH / 2.0, SCREEN_HEIGHT / 2.0, Alignment::Centered);
  break;
```

This is the basics of working with fonts and rendering text!