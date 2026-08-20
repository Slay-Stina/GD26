# 24 Level Editor

our use of the Tiled level creator has many positives, it's a visual way of laying out our levels and it provides us with a tool that an artist can learn to manage on their own. It exports to a handy .JSON file that is easy for us to consume through code.
but, there is some friction in our pipeline currently. I would like to avoid touching Tiled when I am testing mechanics and making the first implementation of new entities.
We're going to create a first version of a level editor. In this lecture it won't be able to store any of the changes we make to a level, but we can still place entities to test if they behave correctly. In a future chapter we will expand on the capabilities of our level editor.
We'll be using DearImGui to help us get some visuals up on screen. We want to have a bar containing all of our entities/tiles so that we can select them then place them anywhere on our game board. One could easily imagine more functionality, like being able to change the size of the game board etc, but for now we'll settle for having an easy way of testing entities.
adding an entity or a tile to a position on the board is not something we currently have an easy way of doing. We call `CreateLevel()` and `CreateEntities()` from `level.cpp` but those all use the .JSON file that we've exported from Tiled . That is fine for the normal case. But for our purposes we need to have a simpler way of handling this. So first we'll refactor parts of `level.h/cpp`

```cpp
// levels.h
Entity* GetNextAvailableEntitySlot(Entity* entityBuffer);
void AddEntity(ID entity_id, int x, int y, LevelData* level);
void removeEntity(int x, int y, LevelData* level);
```

we're adding three new functions to our `level.h` . giving us an easier way of adding/removing entities to the level.
Inside `level.cpp` we'll add the function bodies

```cpp
Entity* GetNextAvailableEntity(LevelData* level) {
  for (int i = 0; i < level->entityCount; i++) {
    if(level->entityBuffer[i].id == ID::NONE) {
      return &level->entityBuffer[i];
    }
  }
  return &level->entityBuffer[level->entityCount++];
}
```

We first check if any of the entities that we have at one point spawned has been set to `ID::NONE` again, meaning that they are no longer in use. If that is the case we return this gap-entity . If no gap entity was found we instead return the forward most index using `entityCount` then after we've done so we increment it by 1 so that it's ready to perform the same function next time.
Note how our for loop runs for `i < level->EntityCount` if we did `<=` we would always run up to the forward most slot and return that without incrementing `entityCount` .

```cpp
// levels.cpp
void AddEntity(ID entity_id, int x, int y, LevelData *level) {
  Entity* entity = level->GetEntity(x,y);
  if(entity == nullptr) {
    entity = GetNextAvailableEntity(level);
  }
  entity->x = x;
  entity->y = y;
  entity->x_prev = x;
  entity->y_prev = y;
  entity->id = entity_id;
  entity->InitializeBaseBehaviour();
}
```

We'll be setting up a way of changing the level tiles like ground or wall but I have opted not to give that logic its own function as it is one line of code and will only be used in one place currently.
our `AddEntity()` first checks if it can find an entity already located on the specified coordinate. Then only if it didnt does it go ahead and use our `GetNextAvailableEntity()` function to fetch the most appropriate index.
Then just like we previously did inside `CreateEntities` we assign the basic variables to our `Entity` .
And by calling `InitializeBaseBehaviour()` as well as updating the `entity_id` we've made sure that the entity is either completely added or the old entity totally overriden.
now we can simplify our `CreateEntities()` function to use this new `AddEntity()`

```cpp
// levels.cpp
void CreateEntities(LevelData* lvl_data, Arena* arena) {
  Reset(arena);
  lvl_data->entityCount = 0;
  fstream stream(lvl_data->level_path);
  auto result = nlohmann::json::parse(stream);
  auto entityData = result["layers"][ENTITIES_INDEX]["data"].get<vector<uint8_t>>();
  lvl_data->entityBuffer = (Entity*)Memory::Allocate(arena, sizeof(Entity) * 256);
  for (int i = 0; i < lvl_data->w * lvl_data->h; i++) {
    unsigned char entity_id = entityData[i];
    if(entity_id != 0) {
      int x = i % lvl_data->w;
      int y = i / lvl_data->w;
      AddEntity((ID)entity_id, x, y, lvl_data);
    }
  }
}
```

we're also adding the function body for our `RemoveEntity()` function

```cpp
// levels.cpp
void RemoveEntity(int x, int y, LevelData* level) {
  Entity* entity = level->GetEntity(x, y);
  if(entity == nullptr) {
    return;
  }
  *entity = {};
}
```

We try and fetch an entity pointer at the specified position. if none is found we can just return. Otherwise we take the value at the pointer reference and set it to its struct default using an empty pair of curly bracers `{}` . This will zero out all variables inside the struct.
We no longer collect `entityCount` then allocate that specific amount. Instead we've simplified our code to just add enough room for 256 entities. This is still a bit flimsy and we can think about refactoring this later.
We need a way of telling our game that we want to use our editor and a way of turning it off. We'll add a `bool` to our `GameData`

```cpp
// gameState.h
struct GameData {
  // other variables hidden for clarity
  bool edit_level;
};
```

then in `Update()` inside `game.cpp` we'll toggle it between true and false.

```cpp
// game.cpp
void Update(GameData* data, float dt) {
  if(KeyPressed(&data->input, SDL_SCANCODE_F2)) {
    data->edit_level = !data->edit_level;
  }
  // other code below hidden for clarity
}
```

We can use the not operator aka `!` to return the inverse of what the value actually was. So this line tells the `edit_level` bool to be set to whatever it was not. So false becomes true and then true becomes false again. This use of the not operator to create a toggle is very common.
The bool does nothing right now as the state of `edit_level` is never checked against.
Let's set up our `leveleditor.h/cpp`

```cpp
// leveleditor.h
#pragma once
#include "camera.h"
#include "input.h"
#include "spriteLibrary.h"

struct Editor {
  ID object_to_place_id;
};

namespace EDITOR {
  void DrawObjectPanel(Editor* editor, Sprite* spriteBuffer);
  void PlaceObject(const int x, const int y, Editor* editor, LevelData* level);
  void Update(Editor* editor, Input* input, LevelData* level);
  void DrawPreview(Editor* editor, Input* input, SDL_Renderer* renderer, LevelData* level, Camera* camera, Sprite* spriteBuffer);
}
```

We'll most likely be adding more variables to our `Editor` struct, but right now we just have to keep track of what type of entity or tile we're trying to place down.
`DrawObjectPanel` will create a new window for us from which we will display all the entities/tiles we can place on our level.
`PlaceObject` will put an entity or tile on the level
`Update` will be called each tick and checking our mouse inputs will decide what to do
`DrawPreview` accepts a whole bunch of parameters and will draw a transparent version of the entity/tile we're placing, to help us see that things are working properly.
We also need to add one of these new `Editor` structs to `gameData`

```cpp
// gameState.h
#include "leveleditor.h"

struct GameData {
  // other variables hidden for clarity
  bool edit_level;
  Editor editorData;
};
```

Lets start adding these to `leveleditor.cpp`

```cpp
// leveleditor.cpp
#include "leveleditor.h"
#include "imgui/imgui.h"
#include "rendering.h"

namespace EDITOR {
  // ...
}
```

First we do our #includes as usual then we make sure that all our functions are wrapped inside the same `EDITOR` namespace as we declared in `leveleditor.h`

```cpp
// leveleditor.cpp
void DrawObjectPanel(Editor* editor, Sprite* spriteBuffer) {
  ImGui::Begin("objects");
  ImVec2 size = {32, 32};
  if(ImGui::ImageButton("Ground", (ImTextureID)GetSpriteFromID(ID::GROUND, spriteBuffer)->texture, size)) {
    editor->object_to_place_id = ID::GROUND;
  }
  ImGui::SameLine();
  if(ImGui::ImageButton("Wall", (ImTextureID)GetSpriteFromID(ID::WALL, spriteBuffer)->texture, size)) {
    editor->object_to_place_id = ID::WALL;
  }
  ImGui::SameLine();
  if(ImGui::ImageButton("Rock", (ImTextureID)GetSpriteFromID(ID::ROCK, spriteBuffer)->texture, size)) {
    editor->object_to_place_id = ID::ROCK;
  }
  ImGui::SameLine();
  if(ImGui::ImageButton("Demon", (ImTextureID)GetSpriteFromID(ID::DEMON, spriteBuffer)->texture, size)) {
    editor->object_to_place_id = ID::DEMON;
  }
  ImGui::SameLine();
  if(ImGui::ImageButton("Medusa", (ImTextureID)GetSpriteFromID(ID::MEDUSA, spriteBuffer)->texture, size)) {
    editor->object_to_place_id = ID::MEDUSA;
  }
  ImGui::End();
}
```

This code, as you can tell, repeats itself identically for each `ImGui::ImageButton` . With the amount of tiles we have this is more than fine. We can look at creating a streamlined automatic solution later. But for now we just want to get things up and running. `ImGui::ImageButton` returns true if it was pressed this frame, it also allows us to add an `ImTextureID` to it to set what the button will display. And becuase we have included `imgui_impl_sdlrenderer3.h` in our `src_external` folder we have given ImGui the ability to convert an `SDL_Texture` into the format it needs.
So we use our `GetSpriteFromID()` function to retrieve the `Sprite*` pointer, then we take the texture stored within the struct. Now we have our `SDL_Texture` , but before we can use it we need to cast it to `ImTextureID` . We also set a size `ImVec2` at the top of the function and pass it in to `ImageButton` to set the size of the button. By calling `ImGui::SameLine()` after each button we make it so each `ImageButton` is layed out in a row rather than in a tall column. We do this as we want to place this horizontal bar at the bottom of our screen. We wrap all of our ImGui calls in `ImGui::Begin/End` to make this its own window.
As each button is pressed we update our `object_to_place_id` so that we can use it later.

```cpp
// leveleditor.cpp
void PlaceObject(const int x, const int y, Editor* editor, LevelData* level) {
  if(editor->object_to_place_id == ID::GROUND || editor->object_to_place_id == ID::WALL) {
    level->cells[y * level->w + x] = (int)editor->object_to_place_id;
  }
  else {
    AddEntity(editor->object_to_place_id, x, y, level);
  }
}
```

So this function might also undergo a refactoring step later as currently we're hard coding a check to see if what we're placing is a Tile or an Entity . This wont scale if we add Water, Lava, Dirt, Ice etc. But with just `GROUND` and `WALL` we can afford it.
So first we compare the ID of our `object_to_place_id` then if it was `GROUND` or `WALL` we set the specific cell in our `cells` array at that position (using our handy 2D->1D equation) to the integer value represented by the enum (with a cast to int using `(int)` ).
If we on the other hand had selected an Entity we go ahead and call `AddEntity()` that we prepared earlier.

```cpp
// leveleditor.cpp
void DrawPreview(Editor* editor, Input* input, SDL_Renderer* renderer, LevelData* level, Camera* camera, Sprite* spriteBuffer) {
  int x;
  int y;
  camera::WorldToGrid(input->mouse_x, input->mouse_y, &x, &y, level);
  Sprite* preview = GetSpriteFromID(editor->object_to_place_id, spriteBuffer);
  if(preview != nullptr) {
    RenderSprite_Grid(preview, level, renderer, camera, x, y, 1, 0.5);
  }
}
```

Even though our `DrawPreview` accepts a lot of parameters the actual function is very small. We get the grid position based on the world position of our mouse, then store the grid position in the int x/y variables we pass along by pointer reference. Then we use `object_to_place_id` to fetch the `Sprite*` accociated with it. And if this preview sprite had an actual value we call `RenderSprite_Grid()` The cool part is that in object-oriented systems we would probably have such tight coupling for our Sprites-to-Entities that it would be a lot less clean to just fetch and render a sprite, as IF it was a real entity/tile.
Note how we pass along `0.5` as a new final parameter to `RenderSprite_Grid` . This is a float value between 0 and 1 that we'll be using to control the alpha of the rendered texture. To do this we need to update `rendering.h/cpp`

```cpp
// rendering.h
void RenderSprite_World(Sprite* sprite, SDL_Renderer* renderer, const Camera* camera, float x, float y, float scale = 1, float alpha = 1);
void RenderSprite_Grid(Sprite* sprite, LevelData* lvl, SDL_Renderer* renderer, const Camera* camera, float x, float y, float scale = 1, float alpha = 1);
```

after our optional `scale` parameter we've added `float alpha` and given it a default value of `1` . Meaning that if it is not specified then it will be set to `1` automatically.

```cpp
// rendering.cpp
void RenderSprite_Grid(Sprite* sprite, LevelData* lvl, SDL_Renderer* renderer, const Camera* camera, float x, float y, float scale, float alpha) {
  camera::GridToWorld(&x, &y, lvl);
  RenderSprite_World(sprite, renderer, camera, x, y, scale, alpha);
}
```

Our `RenderSprite_Grid()` function is the same, except we pass along `alpha` as the final parameter to `RenderSprite_World()` .

```cpp
// rendering.cpp
void RenderSprite_World(Sprite* sprite, SDL_Renderer* renderer, const Camera* camera, float x, float y, float scale, float alpha) {
  SDL_FRect rect;
  rect.x = x;
  rect.y = y;
  rect.h = sprite->height * UPSCALE_FACTOR * scale;
  rect.w = sprite->width * UPSCALE_FACTOR * scale;
  rect.x -= camera->camera_x;
  rect.y -= camera->camera_y;
  SDL_SetTextureAlphaModFloat(sprite->texture, alpha);
  SDL_RenderTexture(renderer, sprite->texture, NULL, &rect);
}
```

With the only change being a call to `SDL_SetTextureAlphaModFloat` we're modifying the alpha of the teture before it's being drawn. Note that this change persists as it updates the texture being pointed to. So if we would do this only once and not repeatedly each time we call the function then every time the texture is rendered it would have the latest alpha value that was set.

```cpp
// leveleditor.cpp
void Update(Editor* editor, Input* input, LevelData* level) {
  if(MousePressed(input, MouseButtons::LEFT)) {
    if(camera::GetIsPointInsideGrid(input->mouse_x, input->mouse_y, level)) {
      int x;
      int y;
      camera::WorldToGrid(input->mouse_x, input->mouse_y, &x, &y, level);
      PlaceObject(x, y, editor, level);
    }
  }
  else if(MousePressed(input, MouseButtons::RIGHT)) {
    if(camera::GetIsPointInsideGrid(input->mouse_x, input->mouse_y, level)) {
      int x;
      int y;
      camera::WorldToGrid(input->mouse_x, input->mouse_y, &x, &y, level);
      RemoveEntity(x, y, level);
    }
  }
}
```

similarly we use the same code twice to either add an entity/tile or remove an entity. To remove a Wall we have to replace it with a Ground tile. So that's why we don't have a a `RemoveTile()` function or more logic inside `RemoveEntity` to handle these cases.
We retrieve the grid position of our mouse after we've determined that it is inside our game world, then we call the corresponding function.
This is everything needed to start working with our `leveleditor` now we just have to call these new functions we've created from our game.
We're calling this through `dev_gui.cpp`

```cpp
// dev_gui.cpp
void DEV::Draw(GameData* data, SDL_Renderer* renderer) {
  // code for drawing memory usage hidden for clarity
  if(data->edit_level) {
    EDITOR::DrawObjectPanel(&data->editorData, data->spriteBuffer);
    EDITOR::DrawPreview(&data->editorData, &data->input, renderer, data->GetCurrentLevel(), &data->camera, data->spriteBuffer);
  }
  ImGui::Render();
  ImGui_ImplSDLRenderer3_RenderDrawData(ImGui::GetDrawData(), renderer);
}
```

if our toggled `edit_level` was true, then we call `DrawObjectPanel()` and `DrawPreview()` from our `EDITOR` namespace.
and lastly we call the `EDITOR::Update` from `game.cpp`

```cpp
// game.cpp
void Update(GameData* data, float dt) {
  if(KeyPressed(&data->input, SDL_SCANCODE_F2)) {
    data->edit_level = !data->edit_level;
  }
  if(data->edit_level) {
    EDITOR::Update(&data->editorData, &data->input, data->GetCurrentLevel());
  }
}
```

Now our level editor is set up, we can now go ahead and test logic without having to enter Tiled and set up/export/parse!