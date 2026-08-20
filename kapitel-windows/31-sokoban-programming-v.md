# 31 Sokoban Programming V

### 31.1 Control deltatime

We're going to continue working on the core of our game as we introduce some sprite animations, gameplay logic and refactoring to support our changes.
First, lets add a feature to allow us to speed up and slow down the entire game.

```cpp
// gameState.h
const float* dt;
float* dt_scaler;
```

We will be multiplying `dt` with `dt_scaler` as we pass `dt` to our .DLL this will allow us to control the game speed. We'll use this to

a) slow down the game to more easily check animations and other effects
b) speed up the game to reach desired gamestates faster.

```cpp
// main.cpp
bool running = true;
float dt;
float dt_scaler = 1;
gameData->dt = &dt;
gameData->dt_scaler = &dt_scaler;
```

Then we modify `dt` in our boilerplate layer

```cpp
void CalculateDeltaTime(float* dt, float scaler) {
  NOW = SDL_GetTicksNS();
  *dt = NOW - PREV;
  *dt = SDL_NS_TO_SECONDS(*dt);
  *dt *= scaler;
  PREV = NOW;
}
```

Now lets add a slider to our `dev_gui.cpp`

```cpp
// dev_gui.cpp
void DrawFPS(GameData* data) {
  EditorData* editor = &data->editor_data;
  // here we need to multiply by `dt_scaler` again or else we get the wrong numbers back
  editor->fps_buffer[editor->fps_buffer_index++] = 1.0 / *data->dt * *data->dt_scaler;
  editor->fps_buffer_index %= editor->fps_buffer_count;
  ImGui::PlotHistogram("fps", editor->fps_buffer, editor->fps_buffer_count, 0, nullptr, 0, FPS, ImVec2(-1, 35));
}

void DEV::Draw(GameData* data, SDL_Renderer* renderer) {
  ImGui::Begin("Dev Tools");
  ImGui::Text("memory arena usage amount");
  Draw_Imgui_Arena_Usage(data->arena_main, "all memory");
  Draw_Imgui_Arena_Usage(data->arena_images, "images");
  Draw_Imgui_Arena_Usage(data->arena_levels, "levels");
  Draw_Imgui_Arena_Usage(data->arena_commands, "commands");
  Draw_Imgui_Arena_Usage(data->arena_entities, "entities");
  Draw_Imgui_Arena_Usage(data->arena_input, "input");
  Draw_Imgui_Arena_Usage(data->arena_scratch, "scratch");
  DrawFPS(data);
  ImGui::SliderFloat("deltaTimeScaler", data->dt_scaler, 0.1, 3);
  ImGui::End();
}
```

With that we can change our game's speed with a simple slider

### 31.2 Select an active entity

Currently all of our entities that respond to inputs are allowed to move at the same time during a key press. This is good for some type of game, but not the one we're making. We want to press X to swap which entity is the one we're moving.
We're going to make use of our `arena_scratch` to set up a pointer-pointer array holding all relevant targets. We'll be recreating this array each frame rather than storing it alongside `entityBuffer` we're doing this so that we never run the risk of having the two arrays drift out of sync by us forgetting to update one when we update the other.

```cpp
struct Gameplay {
  CommandBuffer* commandBuffer;
  LevelData* levels;
  int levelCount;
  int currentLevelIndex;
  float undo_timer;
  Position* input_buffer;
  int input_buffer_capacity;
  int input_buffer_write_count;
  int input_buffer_read_count;
  bool initialized;
  int activePlayerIndex;
  Entity** activePlayerBuffer;
};
```

in `game.cpp` we'll find how many eligable entities we have then allocate that amount to our scratch arena.

```cpp
// game.cpp
void UpdateGame(Gameplay* gameplay, Input* input, Arena* arena_scratch, const float dt) {
  int player_count = 0;
  for (int i = 0; i < level->entityCount; i++) {
    if(entityBuffer[i].active == false) {
      continue;
    }
    if(HasBehaviour(&level->entityBuffer[i], (Behaviour)(IS_PLAYER))) {
      player_count++;
    }
  }
  int index = 0;
  gameplay->activePlayerBuffer = ALLOC_ARRAY(arena_scratch, Entity*, player_count);
  for (int i = 0; i < level->entityCount; i++) {
    if(entityBuffer[i].active == false) {
      continue;
    }
    if(HasBehaviour(&level->entityBuffer[i], (Behaviour)(IS_PLAYER))) {
      gameplay->activePlayerBuffer[index++] = &level->entityBuffer[i];
    }
  }
}
```

So this is an array that sits in sequence in memory (as all arrays do) but they point to pointers to entities that are not in an ordered sequence inside `entityBuffer`. Now each frame this array is recreated. Notice how we've added `Arena* arena_scratch` as a parameter to `UpdateGame` . So in `Update()` we must pass this along as well

```cpp
// game.cpp
// inside a switch-case inside Update()
case SCENE_TYPES::GAME:
  UpdateGame(gameplay, &data->input, data->arena_scratch, dt);
  break;
```

To hammer home the point, recreating this from "scratch" each frame means that there is no way that the data inside it could "go stale". meaning that its referencing old data. It's always automatically kept up to date.
Our goal now is to limit which character acts based on `activePlayerBuffer[activePlayerIndex]` . We are going to do a few things now.

1. create a command that shifts the `activePlayerIndex` forward
2. Push this command
3. Limit our top level `TryMove()` call to only work on this specific entity.

```cpp
// command.h
struct SwapActiveEntityCommand : Command {
  int index_current;
  int index_previous;
  int* value_to_change;

  SwapActiveEntityCommand(int* activeEntityIndex, int limit) {
    index_previous = *activeEntityIndex;
    index_current = *activeEntityIndex + 1;
    value_to_change = activeEntityIndex;
    index_current %= limit;
    type = CMD_TYPE::SWAP_ACTIVE;
  }
};
```

The `limit` is used to ensure that once we reach the end of our entity count we wrap back to 0 instead of going outside of the bounds of the array. This is also why we store `index_previous` as a separate value. It simplifies fetching the old value when we call undo . Though we could do some check to look at if we've reached 0 and wrap to limit during undo/execute I find this less appealing. Storing it inside our command is easy and simplifies the places where we use the command. the `value_to_change` is a bit more generic than necessary. But its a pointer that will point to the memory address of our `activeEntityIndex` so that we can modify it from `Execute()/Undo()`
Now we need to add the `SWAP_ACTIVE` enum to our list as well as creating a constructor for `AnyCommand` that accepts a `SwapActiveEntityCommand` . Though we have outlined this process multiple times in the course material. If you struggle with this step, return to earlier chapters on creating new commands and repeat those steps.

```cpp
// game.cpp
if(KeyPressed(input, SDL_SCANCODE_X) && player_count > 0) {
  SwapActiveEntityCommand swap(&gameplay->activePlayerIndex, player_count);
  Push(gameplay->commandBuffer, swap, GetCurrentLevel(gameplay));
  gameplay->commandBuffer->timestamp += 1;
}
```

so if we press X we create and push our swap command, then we progress `commandBuffer->timestamp` as we want to make sure that this command gets undone/redone in isolation and not part of other commands that comes after.
In `command.cpp` we add our `Execute()` case and `Undo` case. The setup is very similar to our other commands

```cpp
// command.cpp
// Execute()
case CMD_TYPE::SWAP_ACTIVE: {
  SwapActiveEntityCommand* swap = &cmd.swap_active;
  *swap->value_to_change = swap->index_current;
  break;
}
// Undo()
case CMD_TYPE::SWAP_ACTIVE: {
  SwapActiveEntityCommand* swap = &cmd.swap_active;
  *swap->value_to_change = swap->index_previous;
  break;
}
```

Currently changing this pointer's value does absolutely nothing, but behind the scenes we can cycle between our player entities, which currently are all entities besides rocks...
Lets visualize our selected entity. To do this we'll actually go down a pretty deep rabbithole of refactoring. In the Chapter 32 assets.zip you'll find a new sprite `selection_marker.png` we'll put this ontop of our selected entity.
You can also see that we have deleted the `Medusa_Idle_side/front/back` and replaced them with a spritesheet called `medusa_rotate.png` . This is a sequence of sprites we'll use to create our first frame-by-frame animation. This will require some setup and to (eventually) make things easier we'll be refactoring our old naive rendering code.

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
};
```

we remove `tileset` from the name as we will use this for both tilesets and spritesheet animations.

```cpp
// spriteLibrary.h
struct SpriteRenderInfo {
  Sprite* sprite;
  int frame;

  SpriteRenderInfo() {
    this->sprite = nullptr;
    this->frame = 0;
  }

  SpriteRenderInfo(int frame, Sprite* sprite) {
    this->frame = frame;
    this->sprite = sprite;
  }
};

SpriteRenderInfo(Sprite* sprite) {
  this->sprite = sprite;
  this->frame = 0;
}
```

Ok, this struct is a little strange. Mostly it just references a `Sprite` by pointer. But it has a `frame` as well. We'll use the `frame` to get the appropriate sprite using our clever 1D to 2D algorithm later.
We also have 3(!) constructors. One is the default constructor that accepts no parameters. when we start creating our own constructors we can no longer create one of these structs without passing some parameters along the compiler no longer creates a default constructor for us during compilation. By recreating this default constructor we get the ability to do so back.
The second constructor passes both the parameters and assigns them, this creates a fully formed `SpriteRenderInfo` . But with a constructor that accepts only a `Sprite*` we have actually created a way of passing a `Sprite*` as a parameter as a subtitute for a full `SpriteRenderInfo` . This is really neat as this reduces the amount of code duplication and extra boilerplate we have to write. this is called an implicit conversion constructor

```cpp
// spriteLibrary.cpp
static const SpriteDataEntry all_sprite_data[] = {
  // fallback pivot placed in the center due to later changes to rendering
  {SPRITE_ID::Fallback, FALLBACK_PATH, 8, 8},
  {SPRITE_ID::Demon, "assets/sprites/player.png"},
  {SPRITE_ID::Rock, "assets/sprites/rock.png", 10, 20},
  // replaces three old medusa elements
  {SPRITE_ID::Medusa_Rotate, "assets/sprites/medusa_rotate.png", 12, 24, 8, 1},
  {SPRITE_ID::Dropshadow, "assets/sprites/dropshadow.png", 8, 8},
  {SPRITE_ID::black_1x1, "assets/sprites/1x1_black.png", 0, 0},
  {SPRITE_ID::titlescreen_background, "assets/sprites/titlescreen.png"},
  {SPRITE_ID::selection_marker, "assets/sprites/selection_marker.png", 9, 9},
  {SPRITE_ID::dungeon_tileset, "assets/sprites/hell_of_a_time_dungeon_tileset.png", 0, 0, 9, 9}
};
```

The Medusa rotate spritesheet has 8 frames layed out in a line, that's why we pass 8, 1 as the two last parameters. This is just as with `dungeon_tileset` . You should also cleanup `SPRITE_ID` and remove the old Medusa entries.

```cpp
// spriteLibrary.h
Sprite* GetSprite(SPRITE_ID sprite_id, Sprite* spriteBuffer);
SpriteRenderInfo GetSprite_FromEntityState(Entity* entity, Sprite* spritebuffer);
```

We're updating `GetSprite_FromEntityState` to return `SpriteRenderInfo` instead, this workhorse function will be responsible for picking the right frame of our animations based on the states of our entities. It will grow pretty huge pretty soon and we will for sure need to think about how we can manage its size. For this chapter we're going to be super messy and just ensure that the Medusa character works as she should.
Before we dive into `rendering.h/.cpp` we need to fix an issue we had that caused a bug during `Redo` . We need to place all `PostMove/PreRotate/PostRotate()` function calls inside an if-statement to make them only fire if we are not redoing our command. We're also going to be a bit more defensive with our `fromRedo` parameter as we can accidentally pass another variable by mistake and many variables can "decay" into bools. Meaning that they get converted to true if they are for example not a `nullptr` .

```cpp
// command.cpp
enum class FromRedo { No, Yes };

void Execute(AnyCommand cmd, LevelData* level, CommandBuffer* commandBuffer, FromRedo fromRedo = FromRedo::No) {
  switch(cmd.command.type) {
    case CMD_TYPE::NONE:
      break;
    case CMD_TYPE::MOVE: {
      MoveCommand mv = cmd.move;
      mv.entity->x_prev = mv.entity->x;
      mv.entity->y_prev = mv.entity->y;
      mv.entity->x += mv.xDir;
      mv.entity->y += mv.yDir;
      if(fromRedo == FromRedo::Yes) {
        mv.entity->progress_01 = 1;
      }
      mv.entity->action = Actions::MOVING;
      if(fromRedo == FromRedo::No) {
        PostMove(mv.entity, level, commandBuffer);
      }
      break;
    }
    case CMD_TYPE::ROTATE: {
      RotateCommand* rotate = &cmd.rotate;
      if(!HasBehaviour(rotate->entity, CAN_ROTATE)) {
        break;
      }
      if(fromRedo == FromRedo::Yes) {
        rotate->entity->progress_01 = 1;
      }
      rotate->entity->action = Actions::ROTATING;
      if(fromRedo == FromRedo::No) {
        PreRotation(rotate->entity, level, commandBuffer, rotate->from, rotate->to);
      }
      rotate->entity->facing_previous = rotate->from;
      rotate->entity->facing_current = rotate->to;
      if(fromRedo == FromRedo::No) {
        PostRotation(rotate->entity, level, commandBuffer, rotate->from, rotate->to);
      }
      break;
    }
    // other cases hidden for brevity
  }
}
```

our `FromRedo` enum now forces us to pass it explicitly fixing the issue where a bool could decay. Now each `PostMove` and `Rotation` call is held inside a `if FromRedo::No` and that our `progress_01 = 1` only happens on a `FromRedo::Yes` .
We are also storing the `facing_previous` direction inside our `Entity` now. We'll use it to help with animations later. This means that the old `facing` has been renamed to `facing_current` .

```cpp
// Entity.h
struct Entity {
  Actions action;
  ENTITY_ID id;
  bool active;
  Direction facing_current;
  Direction facing_previous;
  int strength;
  int x;
  int y;
  int x_prev;
  int y_prev;
  float progress_01;
  Behaviour behaviour;
};
```

You can also see that we assign `entity->action` to `MOVING/ROTATING` depending on the command.

```cpp
// entity.h
enum class Actions {
  NONE = 0,
  MOVING = 1,
  ROTATING = 2
};

struct Entity {
  Actions action;
  ENTITY_ID id;
  bool active;
  Direction facing_current;
  Direction facing_previous;
  int strength;
  int x;
  int y;
  int x_prev;
  int y_prev;
  float progress_01;
  Behaviour behaviour;
};
```

each Entity has an `action` enum variable we can assign and query against in other code. On its own this does nothing. But it tracks the status of the entity.
We also set `entity->action` to `NONE` during `AddEntity()` inside `levels.cpp` .

```cpp
// levels.cpp
void AddEntity(ENTITY_ID entity_id, int x, int y, LevelData *level) {
  Entity* entity = GetEntity(level, x, y);
  if(entity == nullptr) {
    entity = GetNextAvailableEntity(level);
  }
  entity->active = true;
  entity->x = x;
  entity->y = y;
  entity->x_prev = x;
  entity->y_prev = y;
  entity->id = entity_id;
  entity->action = Actions::NONE;
  InitializeBaseBehaviour(entity);
}
```

our `rendering.h/.cpp` is getting a facelift. We're selecting better function names and removing a few functions as we can consolidate our calls down to three functions in total.

```cpp
// rendering.h
void RenderTile(Sprite* tileset, int cell_id, LevelData* level,
  SDL_Renderer* renderer, const Camera* camera,
  float x, float y, float scale, float alpha);
void RenderSprite_World(SpriteRenderInfo tileset, SDL_Renderer* renderer,
  const Camera* camera, float x, float y,
  float scale = 1, float alpha = 1, bool flipped = false);
void RenderSprite_OnTile(SpriteRenderInfo spriteInfo, LevelData* level,
  SDL_Renderer* renderer, const Camera* camera, float x,
  float y, float scale = 1, float alpha = 1, bool flipped = false);
```

these are our three rendering functions. Eventually all rendering calls go to `RenderSprite_World` . You can also see how we use `SpriteRenderInfo` instead of `Sprite*` . As we allow a `Sprite*` to degrade into a `SpriteRenderInfo` with our third constructor made earlier we have opted for maximum clarity in the case of `RenderTile` . Forcing us to specify the `cell_id` each time it's called.
`RenderTile` this accepts a `Sprite` with more than one tile inside it and a 1D `cell_id` that is then remapped to the correct spot.
`RenderSprite_OnTile` makes sure to offset the rendered entity correctly to make its origin correct.

```cpp
// rendering.cpp
void RenderTile(Sprite* tileset, int cell_id, /* other parameters hidden for brevity */) {
  camera::GridToWorld(&x, &y, level);
  RenderSprite_World({cell_id, tileset}, renderer, camera, x, y, scale, alpha, false);
}

void RenderSprite_OnTile(/* parameters hidden for brevity */) {
  camera::GridToWorld(&x, &y, level);
  x += TILE_SIZE_PX_SCALED / 2.0;
  y += TILE_SIZE_PX_SCALED / 2.0;
  RenderSprite_World(spriteInfo, renderer, camera, x, y, scale, alpha, flipped);
}
```

So both these functions call into `RenderSprite_World` but they modify x and y in different ways. We can also see how `RenderTile` constructs the `SpriteRenderInfo` using the shorthand `{}` and passes both `cell_id` and `tileset` into it.
ok, now we're going to look at the pretty large `RenderSprite_World()` function. I've added comments to break up the code into blocks

```cpp
// rendering.cpp
void RenderSprite_World(SpriteRenderInfo spriteRenderInfo, /* other parameters hidden for brevity */) {
  // fetch some variables to make using them take less characters
  int frame = spriteRenderInfo.frame;
  Sprite* sprite = spriteRenderInfo.sprite;
  // Check if we are working with a tileset/spritesheet by calling `GetSpriteCount()` a new function
  SDL_FRect tilesetRect;
  if(GetSpriteCount(sprite) > 1) {
    int width = sprite->width / sprite->sprite_count_x;
    int height = sprite->height / sprite->sprite_count_y;
    tilesetRect.w = width;
    tilesetRect.h = height;
    // 1D to 2D convertion of the frame to grid-space.
    tilesetRect.x = (frame % sprite->sprite_count_x) * width;
    tilesetRect.y = (frame / sprite->sprite_count_x) * height;
  }
  else {
    tilesetRect.w = sprite->width;
    tilesetRect.h = sprite->height;
    tilesetRect.x = 0;
    tilesetRect.y = 0;
  }
  // the usual offset based on size, pivot and camera
  SDL_FRect rect;
  rect.x = x;
  rect.y = y;
  float final_scale = UPSCALE_FACTOR * scale;
  rect.h = tilesetRect.w * final_scale;
  rect.w = tilesetRect.h * final_scale;
  rect.x -= sprite->pivot_x * final_scale;
  rect.y -= sprite->pivot_y * final_scale;
  rect.x -= camera->camera_x;
  rect.y -= camera->camera_y;
  SDL_SetTextureScaleMode(sprite->texture, SDL_SCALEMODE_PIXELART);
  SDL_SetTextureAlphaModFloat(sprite->texture, alpha);
  // made a variable for SDL_FlipMode to make the function call below shorter.
  SDL_FlipMode flip = flipped ? SDL_FlipMode::SDL_FLIP_HORIZONTAL : SDL_FlipMode::SDL_FLIP_NONE;
  SDL_RenderTextureRotated(renderer, sprite->texture, &tilesetRect, &rect, 0, 0, flip);
}
```

`GetSpriteCount()` is a new helper function in `spriteLibrary.h`

```cpp
// spriteLibrary.h
inline int GetSpriteCount(Sprite* sprite) {
  if(sprite->sprite_count_x == NOT_SET) return 1;
  if(sprite->sprite_count_y == NOT_SET) return 1;
  return sprite->sprite_count_x * sprite->sprite_count_y;
}
```

I've inlined the function but if you find this weird you can always declare it in the .h file and then write the code in `spriteLibrary.cpp` .
The `RenderSprite_World` function has to do quite a lot, but its mostly just math. All parts of this function have existed in previous functions, we have just collected them into one.

Now we have to update our `levelRenderer.cpp` so it correctly uses these functions

```cpp
// levelRenderer.cpp
void RenderLevel(GameData* gameData, SDL_Renderer* renderer) {
  Gameplay* gameplay = &gameData->scenes.gameplay;
  LevelData* level = &gameplay->levels[gameplay->currentLevelIndex];
  Sprite* sprite;
  switch(level->tileset->type) {
    case TILESETS::Dungeon:
      sprite = GetSprite(SPRITE_ID::dungeon_tileset, gameData->spriteBuffer);
      break;
    case TILESETS::NONE:
    case TILESETS::COUNT:
      assert(false);
      break;
  }
  for(int x = 0; x < level->w; x++) {
    for (int y = 0 ; y < level->h; y++) {
      uint16_t id = GetCellID(level, x, y);
      RenderTile(sprite, id, level, renderer, &gameData->camera, x, y, 1, 1);
    }
  }
}
```

from `RenderLevel()` we call `RenderTile()`

```cpp
// levelRenderer.cpp
void RenderEntities(GameData* data, SDL_Renderer* renderer) {
  LevelData* lvl = &data->scenes.gameplay.levels[data->scenes.gameplay.currentLevelIndex];
  Entity** SortedEntities = ALLOC_ARRAY(data->arena_scratch, Entity*, lvl->entityCount);
  for (int i = 0; i < lvl->entityCount; i++) {
    SortedEntities[i] = &lvl->entityBuffer[i];
  }
  std::sort(SortedEntities, SortedEntities + lvl->entityCount, IsEntityBelowOtherEntity);
  Gameplay* gameplay = &data->scenes.gameplay;
  Entity* activeEntity = gameplay->activePlayerBuffer[gameplay->activePlayerIndex];
  for (int i = 0; i < lvl->entityCount; i++) {
    Entity* entity = SortedEntities[i];
    if(entity->active == false) {
      continue;
    }
    SpriteRenderInfo sprite = GetSprite_FromEntityState(entity, data->spriteBuffer);
    if(HasBehaviour(entity, Behaviour::IS_PETRIFIED)) {
      sprite = GetSprite(SPRITE_ID::Rock, data->spriteBuffer);
    }
    float x_animated = std::lerp(entity->x_prev, entity->x, entity->progress_01);
    float y_animated = std::lerp(entity->y_prev, entity->y, entity->progress_01);
    float ground_y = y_animated;
    if(entity->action == Actions::MOVING && HasBehaviour(entity, Behaviour::JUMPS) && !HasBehaviour(entity, Behaviour::IS_PUSHING)) {
      y_animated -= 0.5 * sinf(entity->progress_01 * 3.14);
    }
    Sprite* dropshadow = &data->spriteBuffer[(int)SPRITE_ID::Dropshadow];
    RenderSprite_OnTile(dropshadow, lvl, renderer, &data->camera, x_animated, ground_y, 1, 0.4, false);
    if(entity == activeEntity) {
      SpriteRenderInfo selection_marker = GetSprite(SPRITE_ID::selection_marker, data->spriteBuffer);
      RenderSprite_OnTile(selection_marker, lvl, renderer, &data->camera, x_animated, ground_y);
    }
    RenderSprite_OnTile(sprite, lvl, renderer, &data->camera, x_animated, y_animated, 1, 1, false);
  }
}
```

This function is also growing longer. At the moment it's a non-issue, but with a few more edge-cases we would want to do something about it.
`GetSprite_FromEntityState()` now returns a `SpriteRenderInfo` . We also check if the entity we're rendering is the `ActiveEntity`. This is also
We have also added a `Actions` enum to our `Entity` struct. We'll be using this to help us simplify asking questions about what the `Entity` is doing. We'll look at how this is handled soon.
by storing `activeEntity` we can easily check if the currently rendered entity is the `activeEntity`. And if it is we go ahead and Render the `selection_marker` between the dropshadow and the entity itself. We use the `ground_y` so the selection stays on the ground. `ground_y` is just the old `dropshadow_y` that I have renamed.

for the dropshadow, selection marker and the entities themselves we use `RenderSprite_OnTile` and we sometimes pass just a `Sprite*` to the function (that converts to `SpriteRenderInfo` ) and other times we pass the fully qualified `SpriteRenderInfo` from `GetSprite_FromEntityState()`
In `Game.cpp` we're doing 2 things:

1. forcing the game to wait to move an entity until they have finished rotating.
2. only allow the `activeEntity` to Rotate and TryMove

```cpp
bool are_entities_acting = false;
LevelData *level = GetCurrentLevel(gameplay);
Entity* entityBuffer = level->entityBuffer;
for (int i = 0; i < level->entityCount; i++) {
  if(IsActing(&entityBuffer[i])) {
    are_entities_acting = true;
    break;
  }
}
for (int i = 0; i < level->entityCount; i++) {
  Entity* entity = &entityBuffer[i];
  if(!entity->active) continue;
  switch(entity->action) {
    case Actions::NONE:
      continue;
    case Actions::MOVING:
      entity->progress_01 += MOVE_SPEED * dt;
      break;
    case Actions::ROTATING:
      entity->progress_01 += 8 * dt;
      break;
  }
}
for (int i = 0; i < level->entityCount; i++) {
  Entity* entity = &entityBuffer[i];
  if(entity->progress_01 >= 1) {
    entity->x_prev = entity->x;
    entity->y_prev = entity->y;
    entity->facing_previous = entity->facing_current;
    entity->action = Actions::NONE;
    entity->progress_01 = 0;
    if(HasBehaviour(entity, Behaviour::IS_PUSHING)) {
      RemoveBehaviour(entity, Behaviour::IS_PUSHING);
    }
  }
}
```

we've updated `are_entities_moving` to a more general `_acting` . We've also added a new function `IsActing()` to `entity.h/.cpp` to make checking their acting status easier.

```cpp
// entity.h
bool IsActing(Entity* e);
```

```cpp
// entity.cpp
bool IsActing(Entity* e) {
  if(e->active == false) return false;
  return e->action != Actions::NONE;
}
```

We loop over each entity and checking the `Actions` enum they are using we control how fast `progress_01` should advance. We then loop over all of them again and makes sure that we have no stale references to old positions/rotations and reset `progress_01` and the relevant action to `NONE`. As this happens before any input has been parsed we are yet to update current to a new value. We have also moved the `IS_PUSHING` reset code from our for-loop that calls `TryMove()` .

```cpp
// game.cpp
// at the end of UpdateGame()
if(are_entities_acting) {
  return;
}
if(gameplay->input_buffer_read_count == gameplay->input_buffer_write_count) {
  return;
}
Entity* entity = GetActiveEntity(gameplay);
if(!HasBehaviour(entity, (Behaviour)(RESPOND_TO_INPUT | CAN_MOVE))) {
  return;
}
if(HasBehaviour(entity, Behaviour::IS_PETRIFIED)) {
  return;
}
int xDir = gameplay->input_buffer[gameplay->input_buffer_read_count % gameplay->input_buffer_capacity].x;
int yDir = gameplay->input_buffer[gameplay->input_buffer_read_count % gameplay->input_buffer_capacity].y;
Direction new_facing = DirectionFromXY(xDir, yDir);
if(new_facing != entity->facing_current) {
  RotateCommand rotate(entity, entity->facing_current, new_facing);
  Push(gameplay->commandBuffer, rotate, level);
  return;
}
if(!IsActing(entity)) {
  TryMove(entity, level, gameplay->commandBuffer, xDir, yDir, entity->strength);
  gameplay->commandBuffer->timestamp += 1;
  gameplay->input_buffer_read_count++;
}
```

The biggest win of actually setting up the gameplay code we want to use is that we could entirely remove the old for-loop that iterated over all entities. Now that we are only interested in the `activeEntity` we kan just fetch a pointer to it using our helper function `GetActiveEntity()` . Then we can check if it is supposed to rotate, and if yes we push a `RotateCommand` onto the stack. This will set `entity->action` to `Actions::Rotate` .
We have more than a few if-statements that we evaluate. And if the condition is met we return early. Meaning that no code below it runs. This is the same as nesting if-statements inside each other but instead we have flipped the question to instead of allowing us in we keep the execution out with our return therefore avoiding the Inception style statement within statements.
It also has the added benefit of making the very (very) much easier to read and the flow is easier to understand at a glance.
By gating `TryMove()` behind an `IsActing()` we only progress the `input_buffer` if all previous actions have been resolved. Fair warning. There is some code-smell with the way `Actions` are set up. Part of me wishes to get rid of the entire enum. and work to clarify and make the input reading more robust. This might happen in a later chapter.
Our `GetSprite_FromEntityState()` function will require a large overhaul as we add more spritesheets and sprites for entities. Right now we're just getting the minimal code made to showcase the rotation animation of our Medusa character. Also, the math to select what frames of our looping rotate animation to pick based on `progress_01` took a fair bit of trial and error. It's not the easiest to wrap your mind around but as the case is very narrow it's not a bad idea to talk through the code with an LLM.

```cpp
// spriteLibrary.cpp
// Part 1
SpriteRenderInfo GetSprite_FromEntityState(Entity* entity, Sprite* spritebuffer) {
  if(HasBehaviour(entity, Behaviour::IS_PETRIFIED)) {
    return GetSprite(SPRITE_ID::Rock, spritebuffer);
  }
  if(entity->id == ENTITY_ID::MEDUSA && entity->action == Actions::ROTATING) {
    Sprite* spritesheet = GetSprite(SPRITE_ID::Medusa_Rotate, spritebuffer);
    int start = 0;
    int end = 0;
    switch(entity->facing_previous) {
      case Direction::RIGHT:
        start = 6;
        break;
      case Direction::LEFT:
        start = 2;
        break;
      case Direction::UP:
        start = 4;
        break;
      case Direction::DOWN:
        start = 0;
        break;
    }
    switch(entity->facing_current) {
      case Direction::RIGHT:
        end = 6;
        break;
      case Direction::LEFT:
        end = 2;
        break;
      case Direction::UP:
        end = 4;
        break;
      case Direction::DOWN:
        end = 0;
        break;
    }
    // ...
  }
}
```

our first if statement just checks if we're petrified and returns the rock sprite. This will be modified to a cooler looking thing later. Then we do our brutalistic check that is exclusive to a rotating Medusa. We fetch the frame index for the final poses based on the layout of our `medusa_rotate.png` . We do this for both `_current` and `_previous` facing directions as we want to interpolate between them.

```cpp
// spriteLibrary.cpp
// part 2
int sprite_count = GetSpriteCount(spritesheet);
int forward = ((end - start) % sprite_count + sprite_count) % sprite_count;
int backward = sprite_count - forward;
end = (forward <= backward) ? (start + forward) : (start - backward);
int current_frame = (int)lerp(start, end, entity->progress_01) % sprite_count;
return {current_frame, spritesheet};
```

Next we get how many sprites the full spritesheet is then we do some fancy math to get the distance between the two selected frames going both in a forward direction and a backward direction. We then either set `end` to the start position plus the difference or minus the difference. The reason we have to do this is because we want our `end` to potentially go beyond our `sprite_count` as that is needed to loop over the right-side edge of the sheet back to frame 0. We then make a lerp that goes between our `start` and `end` based on `progress_01` and because our `end` can exist outside of our spritesheet bounds we need to modulo it against our `sprite_count` .

```cpp
// spriteLibrary.cpp
// part 3
switch (entity->id) {
  case ENTITY_ID::MEDUSA: {
    Sprite* sprite = GetSprite(SPRITE_ID::Medusa_Rotate, spritebuffer);
    switch (entity->facing_current) {
      case Direction::RIGHT:
        return {6, sprite};
        break;
      case Direction::LEFT:
        return {2, sprite};
        break;
      case Direction::DOWN:
        return {0, sprite};
        break;
      case Direction::UP:
        return {4, sprite};
        break;
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

Then if we weren't rotating we check if we were in fact Medusa but just not rotating. In that case we just return the frame that points in the right direction. If we were not Medusa we switch-case for two other Entities. And if we reach default we will display our fallback letting us know that we have cases missing for some new entity.
With all of this done we now have spritesheet animations for medusa rotating implemented as well as a delay that makes her rotate before jumping to her next position. But we also have the boilerplate in place to help us animate more entities later. we have also added gameplay logic to handle selecting which entity to use and we have simplified some code.
A lot of systems touched each other in this chapter! Good job getting through this one!