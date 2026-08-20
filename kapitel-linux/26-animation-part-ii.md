# 26 Animation Part II

It's time to start using our known gamestate to select appropriate sprites to render to the screen. In the beginning of this course we set our tile size, both in our .PNGs and in `common.h` to 32. At this stage when we're adding our tileset and version 1.0 of our entities I've opted for 16x16 tiles as the base unit.
This means that the first step is to adjust `common.h` as well as grab all the .pngs from the .ZIP file `SOKOBAN_CHAPTER_027_SPRITES.zip` and replace the contents of `assets/sprites` with these new .png files.

```cpp
// common.h
const int UPSCALE_FACTOR = 4;
const int CELL_SIZE_PX = 16 * UPSCALE_FACTOR;
```

we've increased `UPSCALE_FACTOR` to 4 to account for the smaller base tiles.
Our next issue is the fact that we're going to have entities that we

a) want to place in the middle of a tile
b) want to be larger than a 1x1 tile

Our entities will be standing in the middle of their tile and their heads and arms can reach outside of its own tile. If we decided to ensure that each entity lived exactly in its own tile then we would not have to worry about the next step - but that is very very uncommon artwise.
We need to give our `Sprite` and `SpriteDataEntry` both a `pivot_x` and `pivot_y` integer. These will be manually set by us. They will represent the pixel on our sprite that we want to have be put in the center of the tile.

```cpp
// spritelibrary.h
struct Sprite {
  SDL_Texture* texture;
  int width;
  int height;
  int pivot_x;
  int pivot_y;
};
```

Then our `SpriteDataEntry` will have the same variables

```cpp
// spritelibrary.h
const int NOT_SET = -1;

struct SpriteDataEntry {
  SPRITE_ID id;
  const char* path;
  int pivot_x = NOT_SET;
  int pivot_y = NOT_SET;
};
```

We create the `NOT_SET` constant as a way of flagging if the pivot variables were not set manually by us. This will allow us to catch these cases and programmatically set the pivot to the dead center of our sprite.
also inside `spritelibrary.h` we'll add `SPRITE_ID`s for our new sprites

```cpp
// spritelibrary.h
enum class SPRITE_ID {
  Fallback,
  Ground,
  Ground_alt,
  Wall,
  Rock,
  Demon,
  Medusa_Idle_Side,
  Medusa_Idle_Front,
  Medusa_Idle_Back,
  Golem,
  Siren,
  Dropshadow
};
```

In `spritelibrary.cpp` we can now add our manually set pivots to our static array of `SpriteDataEntry`

```cpp
// spritelibrary.cpp
static const SpriteDataEntry all_sprite_data[] = {
  {SPRITE_ID::Fallback, FALLBACK_PATH, 0, 0},
  {SPRITE_ID::Wall, "assets/sprites/wall.png", 0, 0},
  {SPRITE_ID::Demon, "assets/sprites/player.png"},
  {SPRITE_ID::Rock, "assets/sprites/rock.png", 10, 20},
  {SPRITE_ID::Ground, "assets/sprites/ground.png", 0, 0},
  {SPRITE_ID::Ground_alt, "assets/sprites/ground_alt.png", 0, 0},
  {SPRITE_ID::Medusa_Idle_Side, "assets/sprites/medusa_idle_side.png", 12, 24},
  {SPRITE_ID::Medusa_Idle_Front, "assets/sprites/medusa_idle_front.png", 12, 24},
  {SPRITE_ID::Medusa_Idle_Back, "assets/sprites/medusa_idle_back.png", 12, 24},
  {SPRITE_ID::Dropshadow, "assets/sprites/dropshadow.png", 8, 8}
};
```

Our `Wall` , `Ground` , `Fallback` and newly added `Ground_alt` are all manually set to `(0, 0)` . For all our our level tiles we'll make sure to have our pivot be in the top left corner. If we were to adjust our level tiles to have their pivots centered all of our entities would need to be adjusted by this same amount to not be offset. Fair disclosure, this issue stumped me for a pretty long while (ugh...).
As we can see, our `rock.png` is 20x20 px and the pivot has been placed at the very bottom-center. The same is true for the 24x24 px `medusa_idle_side/front/back` .
With custom pivots we can have sprites that are not 16x16 px and with some clever math always have them centered on their tile.
I have opted to have `SPRITE_ID::Demon` without a manual pivot to showcase how our `NOT_SET` sentinel will be used. The term sentinel means a special reserved value that should never be part of the actual scope of the variable. Used as a substitute for (in this case) a `bool not_set` variable inside the struct itself. Though that would also work and if this sentinel logic is confusing you could easily swap to a `bool` inside the struct instead.
In `LoadSprite()` in `spritelibrary.cpp` we fetch this new pivot and check against our sentinel

```cpp
// spritelibrary.cpp
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
  SDL_DestroySurface(surface);
}
```

we take the `pivot_x/y` from our `SpriteDataEntry` and sets the pivot of our `Sprite` . If we found that our `SpriteDataEntry` had its pivot set to our default sentinel value of `NOT_SET` aka `-1` then we place the pivot in the middle of the sprite.
We are no longer just fetching a sprite from as little data as the id of the entity. instead we'll be using its `behaviour` , `facing Direction` and `progress_01` to get the correct sprite to render.
In `spritelibrary.h/.cpp` we'll add a new function

```cpp
// spritelibrary.h
Sprite* GetSprite_FromEntityState(Entity* entity, Sprite* spritebuffer);
```

This will evaluate the variables inside `Entity` to select the appropriate sprite from the `spritebuffer`

```cpp
// spritelibrary.cpp
Sprite* GetSprite_FromEntityState(Entity* entity, Sprite* spritebuffer) {
  if(HasBehaviour(entity, Behaviour::IS_PETRIFIED)) {
    return &spritebuffer[(int)SPRITE_ID::Rock];
  }
  switch (entity->id) {
    case ID::MEDUSA:
      switch (entity->facing) {
        case Direction::RIGHT:
        case Direction::LEFT:
          return &spritebuffer[(int)SPRITE_ID::Medusa_Idle_Side];
          break;
        case Direction::DOWN:
          return &spritebuffer[(int)SPRITE_ID::Medusa_Idle_Back];
          break;
        case Direction::UP:
          return &spritebuffer[(int)SPRITE_ID::Medusa_Idle_Front];
          break;
      }
    default:
      return GetSpriteFromID(entity->id, spritebuffer);
      break;
  }
  return nullptr;
}
```

Right now we're heavily using the `GetSpriteFromID` as a fallback when we have not set up the specific logic for an entity. At this stage this function does two things.

1. checks if the entity `IS_PETRIFIED` then returns the `SPRITE_ID::Rock` in that case.
2. if it was Medusa we check its facing direction and pick the correct sprite. We'll be flipping the `_Side` sprite along the x-axis to avoid having to add a mirrored sprite to our `assets/sprites` folder each time.

In our `GetSpriteFromID` I've opted to return fallback inside the Medusa case to signal that something has gone terribly wrong

```cpp
// spritelibrary.cpp
Sprite* GetSpriteFromID(ID id, Sprite* spriteBuffer) {
  Sprite* sprite_to_return = nullptr;
  switch (id) {
    case ID::NONE:
      sprite_to_return = nullptr;
      break;
    case ID::GROUND:
      sprite_to_return = &spriteBuffer[(int)SPRITE_ID::Ground];
      break;
    case ID::WALL:
      sprite_to_return = &spriteBuffer[(int)SPRITE_ID::Wall];
      break;
    case ID::DEMON:
      sprite_to_return = &spriteBuffer[(int)SPRITE_ID::Demon];
      break;
    case ID::ROCK:
      sprite_to_return = &spriteBuffer[(int)SPRITE_ID::Rock];
      break;
    case ID::MEDUSA:
      sprite_to_return = nullptr;
      break;
    case ID::SIREN:
      sprite_to_return = &spriteBuffer[(int)SPRITE_ID::Siren];
      break;
    case ID::GOLEM:
      sprite_to_return = &spriteBuffer[(int)SPRITE_ID::Golem];
      break;
  }
  if(sprite_to_return == nullptr || sprite_to_return->texture == nullptr) {
    sprite_to_return = &spriteBuffer[(int)SPRITE_ID::Fallback];
  }
  return sprite_to_return;
}
```

During development we want our program to fail and fail loudly. We want to catch bugs as easily and as often as possible. That's why it's a good idea to be pretty liberal with asserts and to not have this function fail silently by having us return for example the most neutral Medusa Sprite. We're not trying to make it so the problem is as discrete as possible. we WANT it to completely blow up.
Next we need to start using our `GetSprite_FromEntityState()` as well as drawing our new dropshadow sprite beneath the entity. but before we do we're going to make it so that our entities jump in a small parabola when getting to an empty square. We'll be adding two new Behaviour to `entity.h` to control this

```cpp
// entity.h
enum Behaviour : uint32_t {
  NONE = 0,
  CAN_MOVE = 1 << 0,
  IS_PLAYER = 1 << 1,
  RESPOND_TO_INPUT = 1 << 2,
  IS_PETRIFIED = 1 << 3,
  CAN_ROTATE = 1 << 4,
  UNPUSHABLE = 1 << 5,
  JUMPS = 1 << 6,
  IS_PUSHING = 1 << 7
};
```

We will only allow entities that have `JUMPS` and is not currently pushing something to perform the jump.
in `entity.cpp` we'll make Medusa have the new `JUMPS` behaviour

```cpp
// entity.cpp
case ID::MEDUSA:
  SetBehaviour(entity, (Behaviour)(CAN_ROTATE | CAN_MOVE | IS_PLAYER | RESPOND_TO_INPUT));
  AddBehaviour(entity, Behaviour::JUMPS);
  entity->strength = 1;
  break;
```

Then in `TryMove()` and `Update()` inside `game.cpp` we'll be adding and removing `IS_PUSHING` .

```cpp
// game.cpp
bool TryMove(Entity* mover, LevelData* level, CommandBuffer* cmd_buffer, int xDir, int yDir, int strength) {
  // code above hidden for clarity
  if(HasBehaviour(stepInto_entity, CAN_MOVE) && !HasBehaviour(stepInto_entity, UNPUSHABLE)) {
    if(TryMove(stepInto_entity, level, cmd_buffer, xDir, yDir, --strength)) {
      MoveCommand mv(mover, xDir, yDir);
      AddBehaviour(mover, Behaviour::IS_PUSHING);
      Push(cmd_buffer, mv, level);
      return true;
    }
  }
}
```

so if we found an entity and managed to move it with the recursive `TryMove()` then we know that our Mover performed a push and we can now add the behaviour.
in `Update()` where we loop over all of our entities once `are_entities_moving` is false we can reset this behaviour flag to 0

```cpp
for (int i = 0; i < data->GetCurrentLevel()->entityCount; i++) {
  Entity* entity = &data->GetCurrentLevel()->entityBuffer[i];
  if(HasBehaviour(entity, Behaviour::IS_PUSHING)) {
    RemoveBehaviour(entity, Behaviour::IS_PUSHING);
  }
}

if(HasBehaviour(entity, (Behaviour)(RESPOND_TO_INPUT | CAN_MOVE))) {
  // code inside this if-statement hidden for clarity
}
```

Now `IS_PUSHING` is only true for moving entities during the visualisation when they were pushing something.
Lets look at `levelRenderer.cpp` and update our `RenderEntities()` function

```cpp
// levelRenderer.cpp
void RenderEntities(GameData* data, SDL_Renderer* renderer) {
  LevelData lvl = data->levels[data->currentLevelIndex];
  for (int i = 0; i < lvl.entityCount; i++) {
    Entity entity = lvl.entityBuffer[i];
    if(entity.id == ID::NONE) {
      continue;
    }
    Sprite* sprite = GetSprite_FromEntityState(&entity, data->spriteBuffer);
    if(HasBehaviour(&entity, Behaviour::IS_PETRIFIED)) {
      sprite = GetSpriteFromID(ID::ROCK, data->spriteBuffer);
    }
    float x_animated = std::lerp(entity.x_prev, entity.x, entity.progress_01);
    float y_animated = std::lerp(entity.y_prev, entity.y, entity.progress_01);
    float dropshadow_y = y_animated;
    if(HasBehaviour(&entity, Behaviour::JUMPS) && !HasBehaviour(&entity, Behaviour::IS_PUSHING)) {
      y_animated -= 0.5 * sinf(entity.progress_01 * 3.14);
    }
    Sprite* dropshadow = &data->spriteBuffer[(int)SPRITE_ID::Dropshadow];
    RenderEntity_OnTile(dropshadow, &lvl, renderer, &data->camera, x_animated, dropshadow_y, 1, 0.4, false);
    RenderEntity_OnTile(sprite, &lvl, renderer, &data->camera, x_animated, y_animated, 1, 1, entity.facing == Direction::RIGHT);
  }
}
```

It's not to complex, but I hope you see the use case for why we want to add features as we need them instead of trying to divine them as the function is first created. This allows us to only add what we need and keep code as simple as possible until a concrete need for change arrives.
Now we use `GetSprite_FromEntityState()` to retrieve the correct sprite.

We've also removed the old logic here that checked `IS_PETRIFIED` as that is being taken care of by the `GetSprite_FromEntityState()` function. We then store the y position for our dropshadow before we update `y_animated` to account for an entity being allowed to jump.

the following expression `y_animated -= 0.5 * sinf(entity.progress_01 * 3.14);` is part of linear algebra but I had to remind myself on how it was written. What we've done is mapped our `Progress_01` to a parabola that goes from 0 to 0.5 and back to 0 creating an arc.
By multiplying `progress_01` with `PI` we get a value that goes between 0 and `PI` . When we map 0 to `PI` in a sine wave function we start a 0 go up to 1 at `sinf(PI/2)` then back to 0 at `sinf(PI)` .
This changes our `progress_01` mapping from 0.0 - 0.5 - 1.0 into 0.0 - 1.0 - 0.0 then we multiply this value by 0.5 as this is the amplitude (or height) of the arc we want to use. 0.5 being half a tile in height aka 50% .
We then take away this jump height number from `y_animated` as negative y is upwards in SDL .
Linear algebra is a whole course in and of itself. If you can remember that a sine wave ocilates between -1 and 1 over time and that it does so in smooth arcs then with a little practice and some refreshing online you'll be able to retrieve this function (and many like it).
Then we call a new `RenderEntity_OnTile()` function that we'll look at right now inside `rendering.h/.cpp`

```cpp
// rendering.h
void RenderSprite_World(Sprite* sprite, SDL_Renderer* renderer, const Camera* camera, float x, float y, float scale = 1, float alpha = 1, bool flipped = false);
void RenderSprite_Grid(Sprite* sprite, LevelData* lvl, SDL_Renderer* renderer, const Camera* camera, float x, float y, float scale = 1, float alpha = 1, bool flipped = false);
void RenderEntity_OnTile(Sprite* sprite, LevelData* lvl, SDL_Renderer* renderer, const Camera* camera, float x, float y, float scale = 1, float alpha = 1, bool flipped = 1);
```

It has the same exact parameters as `RenderSprite_Grid()` . Also, note how all three of these functions now has a `bool flipped = false` parameter. Note that these are optional parameters so their values will be set to their defaults if not explicitly set.

> [!NOTE]
> with the addition of a new parameter we need to update the function both in our .h and our .cpp file.

Lets look at the changes to `rendering.cpp`

```cpp
// rendering.cpp
void RenderSprite_World(Sprite* sprite, SDL_Renderer* renderer, const Camera* camera, float x, float y, float scale, float alpha, bool flipped) {
  SDL_FRect rect;
  rect.x = x;
  rect.y = y;
  float final_scale = UPSCALE_FACTOR * scale;
  rect.h = sprite->height * final_scale;
  rect.w = sprite->width * final_scale;
  rect.x -= sprite->pivot_x * final_scale;
  rect.y -= sprite->pivot_y * final_scale;
  rect.x -= camera->camera_x;
  rect.y -= camera->camera_y;
  SDL_SetTextureScaleMode(sprite->texture, SDL_SCALEMODE_PIXELART);
  SDL_SetTextureAlphaModFloat(sprite->texture, alpha);
  SDL_RenderTextureRotated(renderer, sprite->texture, NULL, &rect, 0.0, NULL, flipped ? SDL_FlipMode::SDL_FLIP_HORIZONTAL : SDL_FlipMode::SDL_FLIP_NONE);
}

void RenderSprite_Grid(Sprite* sprite, LevelData* lvl, SDL_Renderer* renderer, const Camera* camera, float x, float y, float scale, float alpha, bool flipped) {
  camera::GridToWorld(&x, &y, lvl);
  RenderSprite_World(sprite, renderer, camera, x, y, scale, alpha, flipped);
}

void RenderEntity_OnTile(Sprite* sprite, LevelData* lvl, SDL_Renderer* renderer, const Camera* camera, float x, float y, float scale, float alpha, bool flipped) {
  camera::GridToWorld(&x, &y, lvl);
  x += CELL_SIZE_PX / 2.0;
  y += CELL_SIZE_PX / 2.0;
  RenderSprite_World(sprite, renderer, camera, x, y, scale, alpha, flipped);
}
```

In `RenderSprite_World()` we now call `SDL_SetTextureScaleMode` to make sure that our tiny tiny sprites are not rendered using billinear filtering . That is the most common type of filtering that blurs textures to avoid making sharp low-res textures in our game. But by setting our `SDL_SCALEMODE` to the provided `_PIXELART` we instead use point filtering meaning that no blurring happen. You can try and remove this line and see how bad the pixelart looks.
We also swap from `SDL_RenderTexture` to `SDL_RenderTextureRotated` as this version has a parameter for flipping the texture. We use a new operator to decide if we should use `SDL_FLIP_HORISONTAL` or `SDL_FLIP_NONE`

```cpp
// example
Boss BossToSpawn = difficulty >= DIFFICULTY::HARD ? Dragon : Sheep;
```

in this example we ask a question `difficulty >= DIFFICULTY::HARD ?` then we need to provide a true and a false output separated by a `:` . So if the difficulty in this example is at least `HARD` then the boss becomes a dragon. If it is not it becomes a sheep.
This is the same code as

```cpp
// example
Boss BossToSpawn;
if(difficulty >= DIFFICULTY::HARD) {
  BossToSpawn = Dragon;
}
else {
  BossToSpawn = Sheep;
}
```

Its just some syntactic sugar to help us reduce code lines. And once you know the code structure of the `?` operator its pretty easy to parse.
we have also collected `UPSCALE_FACTOR * scale` into a temporary variable as we'll be using it in 4 places now.

```cpp
rect.x -= sprite->pivot_x * final_scale;
rect.y -= sprite->pivot_y * final_scale;
```

This adjusts the position of the sprite based on the pivot set. This will put the pivot point at the top left corner of the tile (so not yet in the middle). What makes it adjust to be fully centered is the little piece of math that we do that differentiates `RenderSprite_Grid()` and `RenderEntity_OnTile()`

```cpp
void RenderEntity_OnTile(Sprite* sprite, LevelData* lvl, SDL_Renderer* renderer, const Camera* camera, float x, float y, float scale, float alpha, bool flipped) {
  camera::GridToWorld(&x, &y, lvl);
  x += CELL_SIZE_PX / 2.0;
  y += CELL_SIZE_PX / 2.0;
  RenderSprite_World(sprite, renderer, camera, x, y, scale, alpha, flipped);
}
```

by adjusting the x and y position when rendering an entity on a tile we shift the position by half the size of a tile. moving the rendering point from the upper left corner to the center of the tile.
Now we have made it so the default rendering point of an entity is the middle of a tile. Then we adjust the sprite to place its pivot point at this position. The result is an entity with its pivot right at the center of the tile.
The easiest way to check this is to start running the game then to comment out these lines and then call `./build.sh` or `make` to only rebuild the shared library then you can tab back to the game and see what happens when we add each adjustment.
An optional step that I've added is to draw the walkable grid two different colors like a checkerboard to help visualize the grid. I do this by adding to `RenderLevel()`

```cpp
// levelrenderer.cpp
// Sprite* sprite = GetSpriteFromID((ID)cellType, gameData->spriteBuffer); // <-- old
Sprite* sprite;
if(ID(cellType) == ID::GROUND) {
  sprite = &gameData->spriteBuffer[(x + y) % 2 == 0 ? (int)SPRITE_ID::Ground : (int)SPRITE_ID::Ground_alt];
}
else {
  sprite = GetSpriteFromID((ID)cellType, gameData->spriteBuffer);
}
```

`(x + y) % 2` will flip-flop between 1 and 0. So by using the handy `?` operator we can select `Ground` or `Ground_alt` alternating. Then if the ID was not ground we just go ahead and fetch the sprite as normal. This is a bit hacky and we'll most likely be refactoring it soon.
now we have crispy pixel art, that can be flipped along the x-axis and that leverages the pivot positions we've set to place the entity in the correct position.
