# 35 Animation III

Lets add some idle animations to Medusa!
in `chapter 36 asset.zip` you'll find the three new spritesheets we'll be working with. Lets set them up in our `spriteLibrary.h/.cpp`

```cpp
// spriteLibrary.h
enum class SPRITE_ID {
  Fallback,
  Rock,
  Demon,
  Medusa_Rotate,
  Medusa_Idle_Left,
  Medusa_Idle_Front,
  Medusa_Idle_Back,
  Golem,
  Siren,
  Dropshadow,
  titlescreen_background,
  black_1x1,
  dungeon_tileset,
  selection_marker,
  Goal
};
```

```cpp
// spriteLibrary.cpp
static const SpriteDataEntry all_sprite_data[] = {
  {SPRITE_ID::Fallback, FALLBACK_PATH, 8, 8},
  {SPRITE_ID::Demon, "assets/sprites/player.png"},
  {SPRITE_ID::Rock, "assets/sprites/rock.png", 10, 18},
  {SPRITE_ID::Medusa_Rotate, "assets/sprites/medusa_rotate.png", 12, 24, 8, 1},
  {SPRITE_ID::Medusa_Idle_Left, "assets/sprites/medusa_idle_left.png", 12, 24, 4, 1},
  {SPRITE_ID::Medusa_Idle_Front, "assets/sprites/medusa_idle_front.png", 12, 24, 4, 1},
  {SPRITE_ID::Medusa_Idle_Back, "assets/sprites/medusa_idle_back.png", 12, 24, 4, 1},
  {SPRITE_ID::Dropshadow, "assets/sprites/dropshadow.png", 8, 8},
  {SPRITE_ID::black_1x1, "assets/sprites/1x1_black.png", 0, 0},
  {SPRITE_ID::titlescreen_background, "assets/sprites/titlescreen.png"},
  {SPRITE_ID::selection_marker, "assets/sprites/selection_marker.png", 9, 9},
  {SPRITE_ID::dungeon_tileset, "assets/sprites/hell_of_a_time_dungeon_tileset.png", 0, 0, 9, 9},
  {SPRITE_ID::Goal, "assets/sprites/goal.png", 8, 8, 8, 1}
};
```

Each of these animations are four frames long.
Then lets extend our `SpriteRenderInfo` struct to hold a value for if the sprite should be flipped along the x-axis .

```cpp
// spriteLibrary.h
struct SpriteRenderInfo {
  Sprite* sprite;
  int frame;
  bool flipped_x;

  SpriteRenderInfo() {
    this->sprite = nullptr;
    this->frame = 0;
    this->flipped_x = false;
  }

  SpriteRenderInfo(int frame, Sprite* sprite) {
    this->frame = frame;
    this->sprite = sprite;
    this->flipped_x = false;
  }

  SpriteRenderInfo(int frame, Sprite* sprite, bool flipped_x) {
    this->frame = frame;
    this->sprite = sprite;
    this->flipped_x = flipped_x;
  }
};

SpriteRenderInfo(Sprite* sprite) {
  this->sprite = sprite;
  this->frame = 0;
  this->flipped_x = false;
}
```

We'll also set this to `false` for each of the constructors except a new fourth constructor that accepts a `bool` as a third parameter. This will allow us to skip adding this parameter and still pass `Sprite` as if it was a full `SpriteRenderInfo` struct with default values.
Because we want our animations to play all the time, even when the player has made no inputs we have to have some way of giving our `GetSprite_FromEntityState` function knowledge about elapsed time. We're going to make a consession here, if the game can't be played at a stable framerate then this function will cause our animations to not run smoothly. but(!) not running at a stable framerate is not an option for a shipped title - so any safeguard here would be a silly bandaid for the actual solution which is to improve our performance until it can at least hit a stable 30fps. Right now on my machine we always hit our target of 240fps.
We'll be holding onto a `uint64_t` in `GameData` . This will increase by 1 each frame/tick. And because a `uint64_t` can hold fantastically huge numbers it will actually take hundreds of years before we get to the upper bounds of this variable at 240fps.

```cpp
// gameState.h
struct GameData {
  SCENE_TYPES scene_current;
  SCENE_TYPES scene_previous;
  Scenes scenes;
  Transition transition;
  EditorData editor_data;
  Input input;
  Sprite* spriteBuffer;
  Tileset* tilesetBuffer;
  AudioSystem audio;
  uint64_t* ticks_total;
  // ...
};
```

Then we need to allocate it from `main.cpp`

```cpp
// main.cpp
gameData->ticks_total = ALLOC(arena_main, uint64_t);
```

Then in `game.cpp` we will give it a base value of 0 at Intialization then increase it by 1 each time we call `Update()`

```cpp
// game.cpp
void Initialize(GameData* data, SDL_Window* window, SDL_Renderer* renderer) {
  *data->ticks_total = 0;
  DEV::Initialize(window, renderer);
  InitializeAudioSystem(&data->audio, data->arena_main);
  AssetManagement::LoadAllSFX(&data->audio);
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

we have to make sure we assign 0 to the value being pointed to and not the pointer itself. As we're allowed to set the pointer to 0 causing it to become `nullptr` instead. So take note of the `*` before the name.

```cpp
// game.cpp
void Update(GameData* data, float dt) {
  *data->ticks_total += 1;
  // other code hidden for brevity
}
```

We have to remember the dereference `*` here as well or we'll make it `nullptr` .
Now we need to add this `tick_total` to our `GetSprite_FromEntityState()`

```cpp
// spriteLibrary.h
SpriteRenderInfo GetSprite_FromEntityState(Entity* entity, Sprite* spritebuffer, const uint64_t* ticks_total);
```

and then use it in that function to calculate the relevant frame. Note, this function is still very messy.

```cpp
// spriteLibrary.cpp
// in GetSprite_FromEntityState()
switch (entity->id) {
  case ENTITY_ID::MEDUSA: {
    Sprite* sprite = nullptr;
    int frame = 0;
    switch (entity->facing_current) {
      case Direction::RIGHT:
        sprite = GetSprite(SPRITE_ID::Medusa_Idle_Left, spritebuffer);
        frame = (int)((*ticks_total * 8) / FPS % 4);
        return {frame, sprite, true};
      case Direction::LEFT:
        sprite = GetSprite(SPRITE_ID::Medusa_Idle_Left, spritebuffer);
        frame = (int)((*ticks_total * 8) / FPS % 4);
        return {frame, sprite};
      case Direction::DOWN:
        sprite = GetSprite(SPRITE_ID::Medusa_Idle_Back, spritebuffer);
        frame = (int)((*ticks_total * 8) / FPS % 4);
        return {frame, sprite};
      case Direction::UP:
        sprite = GetSprite(SPRITE_ID::Medusa_Idle_Front, spritebuffer);
        frame = (int)((*ticks_total * 8) / FPS % 4);
        return {frame, sprite};
    }
  }
  case ENTITY_ID::DEMON:
    return GetSprite(SPRITE_ID::Demon, spritebuffer);
  case ENTITY_ID::ROCK:
    return GetSprite(SPRITE_ID::Rock, spritebuffer);
  default:
    return GetSprite(SPRITE_ID::Fallback, spritebuffer);
}
```

At the end of the function, after we have not entered the if-statement guarded by us having to have been in `Actions::ROTATING` we instead fetch the relevant idle animation, which for all entities other than Medusa is just a still frame.
For medusa we check the relevant `Direction` then fetch the corresponding spritesheet. We then calculate the correct frame by the following equation: `(total_ticks * framerate) / game_update_speed_in_frames % frames_in_animation` .
This will for as long as we maintain a stable framerate select frame 0,1,2 or 3 as it evaluates `ticks_total` . In the case of `Direction::RIGHT` we give it `true` as a third parameter before returning. This is our new `flipped_x` parameter. Meaning that when the character faces to the right we will render the left facing spritesheet flipped along the x-axis.
Now we should make our `RenderSprite_World()` function care about our `flipped_x` value

```cpp
// rendering.cpp
// at the bottom of the function
SDL_FlipMode flip = (flipped || spriteRenderInfo.flipped_x) ? SDL_FlipMode::SDL_FLIP_HORIZONTAL : SDL_FlipMode::SDL_FLIP_NONE;
SDL_RenderTextureRotated(renderer, sprite->texture, &tilesetRect, &rect, 0, 0, flip);
```

by doing `flipped || spriteRenderInfo.flipped_x` we will make `flip` evaluate to true if either value was set to `true` .
With this our Medusa idles as the game runs!
But we can do better. The magic number for framerate in our frame algorithm can be put into our `Sprite` struct and we can use the fact that we have our helper function `GetSpriteCount` to get the amount of frames.

```cpp
// spriteLibrary.h
struct Sprite {
  SDL_Texture* texture;
  int width;
  int height;
  int pivot_x;
  int pivot_y;
  int sprite_count_x;
  int sprite_count_y;
  int framerate;
};

struct SpriteDataEntry {
  SPRITE_ID id;
  const char* path;
  int pivot_x = NOT_SET;
  int pivot_y = NOT_SET;
  int tileset_cell_count_x = NOT_SET;
  int tileset_cell_count_y = NOT_SET;
  int framerate = NOT_SET;
};
```

Then we need to assign it to `Sprite`

```cpp
// spriteLibrary.cpp
void LoadSprite(Sprite* spriteBuffer, SpriteDataEntry entry, SDL_Renderer* renderer) {
  // other code hidden for brevity
  sprite->framerate = entry.framerate;
}
```

And finally we need to assign it to our idle animations `SpriteDataEntry`

```cpp
// spriteLibrary.cpp
{
  .id = SPRITE_ID::Medusa_Idle_Left,
  .path = "assets/sprites/medusa_idle_left.png",
  .pivot_x = 12,
  .pivot_y = 24,
  .tileset_cell_count_x = 4,
  .tileset_cell_count_y = 1,
  .framerate = 8
},
{SPRITE_ID::Medusa_Idle_Front, "assets/sprites/medusa_idle_front.png", 12, 24, 4, 1, 8},
{SPRITE_ID::Medusa_Idle_Back, "assets/sprites/medusa_idle_back.png", 12, 24, 4, 1, 8},
```

I've opted to show a different way of styling our `{}`. If we add a `.` and the name of the variable we can designate them by name, making it easier to understand which variable we're actually adding to. This might become more necessary as our `Sprite` struct grows or maybe we should see this as a bit smelly and look for another way of simplifying/clarifying this.
Lets go back to our frame calculation algorithm

```cpp
// spriteLibrary.cpp
switch (entity->id) {
  case ENTITY_ID::MEDUSA: {
    Sprite* sprite = nullptr;
    int frame = 0;
    switch (entity->facing_current) {
      case Direction::RIGHT:
        sprite = GetSprite(SPRITE_ID::Medusa_Idle_Left, spritebuffer);
        frame = (int)((*ticks_total * sprite->framerate) / FPS % GetSpriteCount(sprite));
        return {frame, sprite, true};
      case Direction::LEFT:
        sprite = GetSprite(SPRITE_ID::Medusa_Idle_Left, spritebuffer);
        frame = (int)((*ticks_total * sprite->framerate) / FPS % GetSpriteCount(sprite));
        return {frame, sprite};
      case Direction::DOWN:
        sprite = GetSprite(SPRITE_ID::Medusa_Idle_Back, spritebuffer);
        frame = (int)((*ticks_total * sprite->framerate) / FPS % GetSpriteCount(sprite));
        return {frame, sprite};
      case Direction::UP:
        sprite = GetSprite(SPRITE_ID::Medusa_Idle_Front, spritebuffer);
        frame = (int)((*ticks_total * sprite->framerate) / FPS % GetSpriteCount(sprite));
        return {frame, sprite};
    }
  }
}
```

now we have gotten rid of all our magic numbers and if we build and run everything still works just as before!
One tiny thing before we end this chapter. Our camera does not exactly place our level in the center of the screen. We need 1 more step

```cpp
// camera.cpp
void camera::GridToWorld(float* x, float* y, const LevelData* lvl) {
  *x *= TILE_SIZE_PX_SCALED;
  *x += TILE_SIZE_PX_SCALED;
  *x += SCREEN_WIDTH / 2.0;
  *x -= lvl->w * TILE_SIZE_PX_SCALED / 2.0;
  *y *= TILE_SIZE_PX_SCALED;
  *y += TILE_SIZE_PX_SCALED;
  *y += SCREEN_HEIGHT / 2.0;
  *y -= lvl->h * TILE_SIZE_PX_SCALED / 2.0;
}
```

that I believe should do the trick!
