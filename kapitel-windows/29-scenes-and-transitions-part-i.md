# 29 Scenes and transitions Part I

We can't start our game inside gameplay forever. We're going to create a titlescreen and transition between it and gameplay. We'll also lay some groundwork to simplify adding more of these scenes. (like game credits and a main menu).
Right now our `GameData` struct has everything the game could be interested in inside this growing monolothic struct. We're going to make some changes that will require updating a lot of our code. We're taking variables inside the struct that are part of the different scenes and breaking them into their own "substructs"

```cpp
// gameState.h
struct GameData {
  // new
  SCENE_TYPES scene_current;
  SCENE_TYPES scene_previous;
  Scenes scenes;
  Transition transition;
  EditorData editor_data;
  // old
  Input input;
  Sprite* spriteBuffer;
  Memory::Arena* arena_main;
  Memory::Arena* arena_levels;
  Memory::Arena* arena_entities;
  Memory::Arena* arena_images;
  Memory::Arena* arena_commands;
  Memory::Arena* arena_input;
  Memory::Arena* arena_scratch;
  Camera camera;
  ImGuiContext* imGui_context;
  const float* dt;
};
```

We have taken all the variables that are scene agnostic and kept them inside the main struct. Then we've added a new enum to help us keep track of the current and previously active scenes. We'll need to know about both to help us with fade-in-and-out-from-black transitions between the scenes.
All of our editor variables are also collected in a new `EditorData` struct.
There is also a new `Transition` struct. This will be filled with variables to help us cover the screen in black to hide our scene transitions. We'll look into it a bit later.
We've created a `Scenes` struct that will act as an intermediary, holding all of the new structs related to each scene.

```cpp
// gameState.h
struct Scenes {
  Gameplay gameplay;
  MainMenu mainMenu;
  TitleScreen titlescreen;
  Credits credts;
};
```

Each of these new structs will need to be declared above our `Scenes` struct.

```cpp
// gameState.h
struct Gameplay {
};
struct MainMenu {
};
struct TitleScreen {
};
struct Credits {
};
```

We'll fill these soon.

```cpp
// gameState.h
enum class SCENE_TYPES : uint8_t {
  NONE,
  TITLESCREEN,
  MAINMENU,
  GAME,
  CREDITS,
};
```

We create our enum to have the same elements as our `Scenes` struct. As well as `NONE` . We're never going to be using it directly as a scene we go to. But we're using it along with `assert()` to catch bad code easier.
We're moving `GetCurrentLevel()` out of the function to below our `GameData` struct. This will require us to make some changes to it

```cpp
// gameState.h
inline LevelData* GetCurrentLevel(Gameplay* game) {
  return &game->levels[game->currentLevelIndex];
}
```

We now pass along a `Gameplay*` pointer instead of fetching this through the implicit connection between the structs data and its function.
We've set this function to be `inline` this means that it will be the same function for all files that implement this .h file. If we didn't have this each file that implemented it would get its own copy of the function and as soon as two of these functions included each other there would be a compilation conflict. The other option would be to create a `gameState.cpp` file and add this function to it. Totally legit, but as this is just a small helper function I've opted to have it live inside my .h file.
Lets look at `Gameplay`

```cpp
// gameState.h
struct Gameplay {
  CommandBuffer* commandBuffer;
  LevelData* levels;
  int levelCount;
  int currentLevelIndex;
  Position* input_buffer;
  int input_buffer_capacity;
  int input_buffer_write_count;
  int input_buffer_read_count;
  bool initialized;
};
```

besides `intialized` all of the variables inside `Gameplay` are just the gameplay specific variables previously found inside `GameData` .
These changes means that everywhere where we could previously write `data->commandBuffer` or `data->levels` now have to go through our intermediary `Scenes` struct then into the specific struct.

```cpp
// example
data->levels // old
data->scenes.gameplay.levels // new
```

This feels like a lot more indirection. And it is. But we're allowing for this additional hurdle to help our project grow. With just a flat struct containing everything we'll have to be very careful with how we name files. And it will become easier and easier to misunderstand what the purpose of a variable is. But we have a lot of functions that we pass multiple variables from `data` to. These function calls will become extremely long if we need to go thorugh `scenes` then `gameplay` for each variable.
This is how we'll fix this issue

```cpp
// example
// old
Undo(data->commandBuffer, data->GetCurrentLevel())
// new
Gameplay* gameplay = &data->scenes.gameplay;
Undo(gameplay->commandBuffer, GetCurrentLevel(gameplay));
```

So, by fetching `Gameplay* gameplay` once we can collapse the function calls back to their original size. In this example we can also see the new way we need to call `GetCurrentLevel()` .
We're going to be a bit brutalistic at this stage and for `Gameplay` and `Titlescreen` (our first two scenes we'll be working with) we'll add some functions directly into `game.cpp` as these are not supposed to be able to be called by outside files.

```cpp
// game.cpp
void InitializeGame(Gameplay* gameplay, Arena* arena_levels) {
  // ... code will go here
}
void UpdateTitlescreen(TitleScreen* titlescreen, const float dt) {
  // ... code will go here
}
void UpdateGame(Gameplay* gameplay, Input* input, const float dt) {
  // ... code will go here
}
```

We can see how we pass along `Gameplay* gameplay` to `UpdateGame()` this means that the variables inside `TitleScreen` will not be accessible to this function as we have not passed along the complete `GameData* data` . This will help reduce complexity, improve readability and reduce the chance of us creating hard to understand bugs. I've also taken the time to make the `dt` (deltatime) parameter a `const` as we're not supposed to make any changes to it, just read its value.
Our `InitializeGame()` lives only inside our `game.cpp` and is called from our `Initialize()` function. Inside it we've just placed the gameplay specific code that previously lived inside `Initialize()` . This step was not strictly necessary but it will help with readability later in the project.

```cpp
// game.cpp
void InitializeGame(Gameplay* gameplay, Arena* arena_levels) {
  assert(gameplay->initialized == false);
  gameplay->currentLevelIndex = 1;
  CreateLevel(arena_levels, &gameplay->levels[0], "assets/levels/testLevel.tmj");
  CreateLevel(arena_levels, &gameplay->levels[1], "assets/levels/testLevel_box.tmj");
  gameplay->initialized = true;
}

void Initialize(GameData* data, SDL_Window* window, SDL_Renderer* renderer) {
  DEV::Initialize(window, renderer);
  AssetManagement::LoadAllSprites(data->spriteBuffer, renderer);
  data->imGui_context = ImGui::GetCurrentContext();
  SDL_Texture* blackfade = GetSprite(SPRITE_ID::black_1x1, data->spriteBuffer)->texture;
  SDL_SetTextureBlendMode(blackfade, SDL_BLENDMODE_BLEND);
  InitializeGame(&data->scenes.gameplay, data->arena_levels);
  ChangeScene(data, SCENE_TYPES::GAME);
}
```

We'll be using a 1x1 size black pixel as our texture for our fade in/out from black. But due to it having no transparent pixels in the image itself SDL defaults to giving it `SDL_BLENDMODE_NONE` . Without us setting it to `SDL_BLENDMODE_BLEND` we wont be able to update its alpha value to make it transparent.
We're adding some new sprites, so first we'll add them as `SPRITE_ID`s , then we'll load them lastly we'll write the `GetSprite()` helper function.

You'll find the required sprites in the `chapter 30 sprite assets.zip` file.

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
  Dropshadow,
  titlescreen_background,
  black_1x1
};

Sprite* GetSprite(SPRITE_ID sprite_id, Sprite* spriteBuffer);
```

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
  {SPRITE_ID::Dropshadow, "assets/sprites/dropshadow.png", 8, 8},
  {SPRITE_ID::black_1x1, "assets/sprites/1x1_black.png", 0, 0},
  {SPRITE_ID::titlescreen_background, "assets/sprites/titlescreen.png", 0, 0}
};

Sprite* GetSprite(SPRITE_ID sprite_id, Sprite* spriteBuffer) {
  return &spriteBuffer[(int)sprite_id];
}
```

We set the pivot of the new sprites to 0,0 as we do not want to shift them at all. The `GetSprite()` function is a one-liner and you could just as easily substitute and use the code directly. I am of two minds about these types of helper functions, but I've kept it as I often find students respond well to functions that help with contextualisation. The reason we can do this simple lookup inside the `spriteBuffer` is because when we loaded the sprites we looped over them in `SPRITE_ID` order. Meaning that the `SPRITE_ID` with enum value 0 was put into `spriteBuffer[0]` .
Back in our `Initialize()` in `game.cpp` we grab the texture and update its `BLEND_MODE` . after that we call `InitializeGame()` and finally call `ChangeScene()` . We'll come back to `ChangeScene()` for now you can think of it as just setting our `current_scene` to the appropriate value.
Lets look at our `Update()` function that we call from `main.exe`

```cpp
// game.cpp
// new Update() part 1
void Update(GameData* data, float dt) {
  Gameplay* gameplay = &data->scenes.gameplay;
  TitleScreen* titlescreen = &data->scenes.titlescreen;
  EditorData* editorData = &data->editor_data;
  Transition* transition = &data->transition;
  if(KeyPressed(&data->input, SDL_SCANCODE_F2)) {
    editorData->edit_level = !editorData->edit_level;
  }
  if(editorData->edit_level) {
    EDITOR::Update(&editorData->editor, &data->input, GetCurrentLevel(gameplay), gameplay->commandBuffer);
  }
  if(KeyPressed(&data->input, SDL_SCANCODE_5)) {
    ChangeScene(data, SCENE_TYPES::TITLESCREEN);
    return;
  }
  // ... more code to follow
}
```

A lot of changes here. Lets break them down one by one
First we fetch pointer references to `Gameplay` , `TitleScreen` , `EditorData` and `Transition` to simplify passing their variables along. We update our old call sites to use the new way we find variables We also add a quick testbutton `5` to call `ChangeScene()` .
before moving forward we should look at our new `Transition` struct inside `gameState.h`

```cpp
// gameState.h
struct Transition {
  enum States {
    Inactive,
    FadeTo,
    FadeFrom
  };
  States state;
  float fade_time_elapsed;
  float fade_time_duration = 1;
};
```

We've opted for one single struct that can handle both the fade-in and the fade-out. We also set `fade_duration` to have a default value of `1` . We'll be controlling the alpha of a black texture by comparing `time_elapsed` with `time_duration` . Note: `fade_time_elapsed` is a tiny bit exessive with the context that the variable lives inside `Transition` . If you want you can change the name of these to just `time_elapsed` and `fade_duration` . I'll keep the verbose versions.
The `States` enum helps us track what the `Transition` is supposed to be doing using simple `switch-statements` .

```cpp
// game.cpp
// New Update() part 2
if(transition->state != Transition::Inactive) {
  transition->fade_time_elapsed += dt;
  if(transition->fade_time_elapsed >= transition->fade_time_duration) {
    transition->fade_time_elapsed = 0;
    switch (transition->state) {
      case Transition::Inactive:
        break;
      case Transition::FadeTo:
        transition->state = Transition::FadeFrom;
        break;
      case Transition::FadeFrom:
        transition->state = Transition::Inactive;
        break;
    }
  }
}

switch(data->scene_current) {
  case SCENE_TYPES::TITLESCREEN:
    UpdateTitlescreen(titlescreen, dt);
    if(AnyKeyPressed(&data->input)) {
      if(transition->state == Transition::FadeTo || transition->state == Transition::Inactive) {
        ChangeScene(data, SCENE_TYPES::GAME);
      }
    }
    break;
  case SCENE_TYPES::MAINMENU:
    break;
  case SCENE_TYPES::GAME:
    UpdateGame(gameplay, &data->input, dt);
    break;
  case SCENE_TYPES::CREDITS:
    break;
  case SCENE_TYPES::NONE:
    assert(false);
    break;
}
```

We check if our `Transition` state is not `Inactive` . meaning that it is currently running a fade. If it is we

1. add `dt` to `time_elapsed` then if `time_elapsed` has reached our `duration` we reset it and depending on the State of our Transition we either make the transition `Inactive` or transition from `FadeTo` to `FadeFrom`

after that we check which scene we're currently in and call the appropriate `Update` function.
`AnyKeyPressed()` is a new function that we need to add to `input.h/.cpp` .

```cpp
// input.h
bool AnyKeyPressed(const Input* input);
```

```cpp
// input.cpp
bool AnyKeyPressed(const Input *input) {
  for (int i = 0; i < SDL_SCANCODE_COUNT; i++) {
    if(KeyPressed(input, (SDL_Scancode)i)) {
      return true;
    }
  }
  return false;
}
```

It loops over the entire keyboard array and checks if any of the buttons where pressed that frame, if not it returns false.
Lets finally look at our `ChangeScene()` function

```cpp
// game.h
void ChangeScene(GameData* data, SCENE_TYPES new_scene);
```

```cpp
// game.cpp
void ChangeScene(GameData* data, SCENE_TYPES new_scene) {
  assert(new_scene != data->scene_current);
  data->scene_previous = data->scene_current;
  data->scene_current = new_scene;
  data->transition.state = data->scene_previous == SCENE_TYPES::NONE ? Transition::FadeFrom : Transition::FadeTo;
  data->transition.fade_time_elapsed = 0;
  switch (data->scene_current) {
    case SCENE_TYPES::TITLESCREEN:
      data->transition.fade_time_duration = 1;
      break;
    case SCENE_TYPES::MAINMENU:
      break;
    case SCENE_TYPES::GAME: {
      data->transition.fade_time_duration = 0.5f;
      Gameplay* gameplay = &data->scenes.gameplay;
      assert(gameplay->initialized);
      StartLevel(gameplay, data->arena_commands, data->arena_entities);
      break;
    }
    case SCENE_TYPES::CREDITS:
      break;
    case SCENE_TYPES::NONE:
      assert(false);
      break;
  }
}
```

We assert that we didn't try and change the scene to the scene we were already in. This behaviour should never happen and we're fine with crashing the program at this point.
If we get past our assert then we know that current and new are different and we can then safely store the old version that `current` has at the moment in `previous` then update `current` .
We use our handy `?` operator to decide which `Transition` state to select based on if we are entering the first ever scene of the game whether or not the fade should instantly begin as fading out or if we should fade in first.
We reset `_time_elapsed` then depending on the scene we're entering we do scene specific setups. We also `assert(false)` if we ever tried to change to `NONE` . a `(false)` assert will always crash our program.
We'll continue to add logic here as it becomes necessary.
`StartLevel()` is also a new function exclusive to `game.cpp` that does the following

```cpp
// game.cpp
void StartLevel(Gameplay* gameplay, Arena* arena_commands, Arena* arena_entities) {
  Reset(arena_commands);
  CreateEntities(&gameplay->levels[gameplay->currentLevelIndex], arena_entities);
}
```

We make sure we have no commands from a previous level sitting around in our `arena_command` then we create the entities for the level set by `currentLevelIndex` .
Our codebase in currently littered with error messages. All of these are due to the fact that we try and access our variables from `data` directly. These all require the same changes to begin working again.

1. we fetch a pointer to the struct that actually holds the variable
2. we substitute `data->` with `the_struct_we_fetched_in_step_01->`

This is simple but boring work. We can drastically speed up this process by making good use of the multi-caret editing from the previous chapter.

1. mark a block of code inside select-mode using `v`
2. press `s` to select based on the search phrase
3. press `enter` to finish selecting
4. make changes as normal
5. press `,` to remove all but the normal caret

There is very little in the way of creativity at this part of refactoring.
Next lets look at our `Draw()`

```cpp
// game.cpp
void Draw(GameData* data, SDL_Renderer* renderer) {
  DEV::PreDraw(data->imGui_context);
  SDL_SetRenderDrawColor(renderer, 120, 70, 120, 255);
  SDL_RenderClear(renderer);
  switch(data->transition.state) {
    case Transition::Inactive:
      DrawScene(data, data->scene_current, renderer);
      break;
    case Transition::FadeTo: {
      DrawScene(data, data->scene_previous, renderer);
      float alpha = data->transition.fade_time_elapsed / data->transition.fade_time_duration;
      RenderSprite_World(GetSprite(SPRITE_ID::black_1x1, data->spriteBuffer), renderer, &data->camera, 0, 0, SCREEN_WIDTH, alpha);
      break;
    }
    case Transition::FadeFrom: {
      DrawScene(data, data->scene_current, renderer);
      float alpha = 1 - data->transition.fade_time_elapsed / data->transition.fade_time_duration;
      RenderSprite_World(GetSprite(SPRITE_ID::black_1x1, data->spriteBuffer), renderer, &data->camera, 0, 0, SCREEN_WIDTH, alpha);
      break;
    }
  }
  DEV::Draw(data, renderer);
  SDL_RenderPresent(renderer);
}
```

We check the state of transition and depending on the current state we either just draw the current scene or we draw the previous-or-current scene along with an overlayed fade texture. We are actually grabbing our 1x1 black pixel and scaling it up to be as alrge as our `SCREEN_WIDTH` . By doing this we ensure that it covers the entire screen. (at least for as long as our screen is wider than it is tall)
The alpha calculations inside `FadeTo` and `FadeFrom` are very similar, except that for `FadeFrom` we take the value and we subtract it from `1` . Meaning that we start at 1 and go down towards zero, as opposed to counting up from zero.
Our `DrawScene()` takes the `scene_current` and draws the appropriate "stuff"

```cpp
// game.cpp
void DrawScene(GameData* data, SCENE_TYPES scene, SDL_Renderer* renderer) {
  switch(scene) {
    case SCENE_TYPES::TITLESCREEN: {
      Sprite* background = GetSprite(SPRITE_ID::titlescreen_background, data->spriteBuffer);
      RenderSprite_World(background, renderer, &data->camera, 0, 0);
      break;
    }
    case SCENE_TYPES::MAINMENU:
    case SCENE_TYPES::GAME:
      RenderLevel(data, renderer);
      RenderEntities(data, renderer);
      break;
    case SCENE_TYPES::CREDITS:
      break;
    case SCENE_TYPES::NONE:
      assert(false);
      break;
  }
}
```

We can see how we've just lifted the `RenderLevel()` and `RenderEntities()` to the `GAME` case.
With these changes we can start our game from the titlescreen then press any key, watch the screen fade to black before putting us into gameplay!