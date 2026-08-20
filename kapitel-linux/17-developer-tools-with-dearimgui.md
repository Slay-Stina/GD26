# 17 Developer Tools with DearImGui

So far, we've added quite a few quality of life features to our game. We can store a game state , we can undo/redo actions , we can hot-reload our code by splitting our program into an exe and a shared library (`.so`).
But! The largest differentiating factor between our development environment and an off-the-shelf engine like Unity or Unreal Engine is the lack of a visual development ui. Something where info about our game and buttons, gizmos, sliders and text boxes could live.
We're going to solve that today by adding Dear ImGui to our project.

Dear ImGui is an immediate mode GUI framework that allows us to, with very very little code, get a developer window up and running.
This window is not meant to act as actual game UI, but is instead only meant to hold our development tools. Dear ImGui uses a game engine style approach where no state is copied over to the gui, instead all of the data is being fed to the gui each frame. This ensures that there is no desync between what the gui visualizes and what the data of the game is.
We will download Dear ImGui from: https://github.com/ocornut/imgui/releases
At time of writing the latest release was v1.92.8
We've come to expect that everything we download and add to our program is a bunch of .h files and .a or .so files. But this framework comes just as a series of .h/cpp files.
This is not really a problem and we'll have it up and running in no time.
In the root of our project I'll add a new directory called `src_external` this is because I don't want to have these new .cpp files mingle directly with my own. It also helps if I should decide I don't want to include these .cpp files in my build later.
inside `src_external` I'll add a new subdirectory just named `imgui` . Inside it I'll fetch the following files from the Dear ImGui .ZIP I downloaded earlier

- `imgui.h`
- `imconfig.h`
- `imstb_truetype.h`
- `imstb_rectpack.h`
- `imstb_textedit.h`
- `imgui_internal.h`
- `imgui_impl_sdl3.h`
- `imgui_impl_sdlrenderer3.h`
- `imgui.cpp`
- `imgui_draw.cpp`
- `imgui_tables.cpp`
- `imgui_widgets.cpp`
- `imgui_impl_sdl3.cpp`
- `imgui_impl_sdlrenderer3.cpp`

Note the six .cpp files, we'll need to reference these in our `cmakelists.txt` in order to give access to both our executable and shared library. We use a `GLOB` command to fetch all the .cpp files from our normal `src` folder, and we can do the same action for this new directory that we named `src_external` or we can manually reference them. To show how we would go about this, and because the files won't change after this point lets look at adding them by direct name reference.

```cmake
# cmakelists.txt
# Collect imgui cpp files
set(IMGUI
  src_external/imgui/imgui.cpp
  src_external/imgui/imgui_draw.cpp
  src_external/imgui/imgui_tables.cpp
  src_external/imgui/imgui_widgets.cpp
  src_external/imgui/imgui_impl_sdlrenderer3.cpp
  src_external/imgui/imgui_impl_sdl3.cpp
)
```

We create the variable `IMGUI` above that holds all the .cpp files we've added.

```cmake
# cmakelists.txt
add_executable(${PROJECT_NAME} ${EXE_EXCLUSIVE} ${IMGUI})
target_include_directories(${PROJECT_NAME} PRIVATE include src_external)
```

Then for both our executable and our shared library we make sure that `${IMGUI}` is added to the list of files that they can access as well as where they are allowed to look for .h files. In this case our newly created `src_external` folder.

```cmake
# cmakelists.txt
add_library(${DLL_NAME} SHARED ${DLL_EXCLUSIVE} ${IMGUI})
target_include_directories(${DLL_NAME} PRIVATE include src_external)
```

> [!NOTE]
> On Linux, shared libraries use the `.so` extension instead of `.dll`. The `add_library(... SHARED ...)` command in CMake handles this automatically.

Later we will look at how we can limit access to Dear ImGui if we build a Release version rather than a Debug version. But for now, these are all the additions we need to add to our `cmakelists.txt`
Next we'll create `dev_gui.h/cpp` .

```cpp
// dev_gui.h
#pragma once
#include <SDL3/SDL.h>
#include "imgui/imgui_impl_sdl3.h"
#include "gameState.h"

namespace DEV {
  void Initialize(SDL_Window* window, SDL_Renderer* renderer);
  void ProcessEvents(SDL_Event* event);
  void PreDraw();
  void Draw(GameData* data, SDL_Renderer* renderer);
}
```

Because of the generic names of the functions I've put them in their own namespace. The other option is naming them `gui_functionName` . Without one of these two solutions we get errors if we try and include two different .h files that both implement functions with the same name.
`Initialize` will be used to set up required Dear ImGui boilerplate. `ProcessEvents` will grab the `SDL_Event*` pointer that holds information about if the mouse was clicked or a key was pressed. This will hook into the ImGui code to make it so we can drag it around and interact with it. `PreDraw` is a step that we do before each `Draw` . the predraw sets up the frame . Finally in `Draw` we actually call all of our ImGui code responsible for putting our menues, buttons and sliders on the screen that's why we pass along a `GameData*` pointer to `Draw` .
Now lets implement them

```cpp
// dev_gui.cpp - part 1
#include "dev_gui.h"
#include "gameState.h"
#include "command.h"
#include "imgui/imgui_impl_sdlrenderer3.h"
#include <SDL3/SDL.h>
#include <string>

using namespace std;

void DEV::Initialize(SDL_Window* window, SDL_Renderer* renderer) {
  ImGui::CreateContext();
  ImGui_ImplSDL3_InitForSDLRenderer(window, renderer);
  ImGui_ImplSDLRenderer3_Init(renderer);
  ImGuiIO& io = ImGui::GetIO();
  int w, h;
  SDL_GetWindowSize(window, &w, &h);
  io.DisplaySize = ImVec2((float)w, (float)h);
}
```

The `Initialize()` function creates what is known as a context . This is required as it is what is responsible for holding all the info about our ImGui . Because Dear ImGui is code we haven't written ourselves it becomes a bit more difficult to break down every part of it, as what happens in the background is a bit beyond the scope of this lecture series. In Helix you can press `g-d` to jump to the function under the caret. If you're interested you can dive into the ImGui code and see what it does under the hood. But for brevity we need to actually have a context to have ImGui able to do anything.
`ImGuiIO` is a part of the context struct. It holds a lot of data that ImGui uses to understand how it is supposed to work. One thing it needs to know is how large the game window is. We use the handy SDL function called `SDL_GetWindowSize` to get the window size, the width and height are stored in the `w` and `h` variables that we pass along by reference . So the `GetWindowSize` function actually sets new values for `w` and `h` that we then pass along to the `ImGuiIO` . `ImVec2` is just a struct that ImGui has created that holds 2 floats but has some additionally functionality that helps ImGui check that everything is working behind the scenes. So we cast our ints to floats as that is the type `ImVec2` expects.
Next we do setup that comes ready-made with ImGui - send `window*` and `renderer*` to helper functions that hooks ImGui's backend up with SDL3 .
How were we supposed to know this in advance? we weren't. This is what example projects and code documentation is for.

```cpp
// dev_gui.cpp part 2
void DEV::ProcessEvents(SDL_Event* event) {
  ImGui_ImplSDL3_ProcessEvent(event);
}
```

Nice and simple, ImGui provides a function that we can use to pass along our `SDL_Event*` pointer. We could use their function directly, but using our `dev_gui` like a simplified remote control is helpful as it allows us to put all code that interfaces with ImGui in one spot.

```cpp
// dev_gui.cpp part 3
void DEV::PreDraw(SDL_Renderer* renderer) {
  ImGui::NewFrame();
}
```

We call the `NewFrame()` function that lives inside the `ImGui` namespace. This is to safeguard as the function name is very generic (very similar to our own naming standard)

```cpp
// dev_gui.cpp part 4
void DEV::Draw(GameData* data, SDL_Renderer* renderer) {
  ImGui::Begin("Dev Tools");
  // Our specific IMGUI code will go here
  ImGui::End();
  ImGui::Render();
  ImGui_ImplSDLRenderer3_RenderDrawData(ImGui::GetDrawData(), renderer);
}
```

between `Begin` and `End` is where we will add all our code that lets us add buttons, sliders etc to our Dev window . Each `Begin`+`End` pair will produce its own dev window.
Once we have created all of our stuff we call `Render()` and right afterwards we call the specific SDL3 helper function `RenderDrawData()` that takes (behind the scenes) everything that `Render()` set up in a generic way, and displays it using SDL3's render system.
Now lets set up three functions inside our .cpp that we'll call inside our `Begin+Draw` area.

```cpp
// dev_gui.cpp part 5
void Draw_Imgui_Arena_Usage(Arena* arena, std::string name_of_arena) {
  float fraction = (float)arena->used / (float)arena->size;
  string barText = name_of_arena;
  barText += " " + to_string(arena->used);
  barText += " / " + to_string(arena->size);
  ImGui::ProgressBar(fraction, ImVec2(-1,0), barText.c_str());
}
```

I split up the creation of `barText` to three rows to help with reading clarity. But it could all have been added together on one line.
We use common division to find out how much of an Arena's memory budget is being used. Then create a `ProgressBar` that is filled in to that percentage and writes `barText` inside of it. Passing in `ImVeck(-1,0)` allows the bar to stretch the entire width of the dev window. the `c_str()` function converts a string into a `const char*` which is the data type that `ProgressBar` expects.
With this we can add

```cpp
// inside Begin+End block
Draw_Imgui_Arena_Usage(data->arena_images, "images");
Draw_Imgui_Arena_Usage(data->arena_levels, "levels");
Draw_Imgui_Arena_Usage(data->arena_commands, "commands");
Draw_Imgui_Arena_Usage(data->arena_entities, "entities");
```

To visualize how much of each of these arenas are currently being used. I leveraged this to bump my capacity for the `commandBuffer` from 2000 to 20000 for example. Remember that `arena_commands` is a subarena inside `arena_levels`.
Next we'll do some magic with undo/redo

```cpp
// dev_gui.cpp part 6
void Draw_History(CommandBuffer* buffer) {
  int sliderPos = buffer->index;
  if(ImGui::SliderInt("history", &sliderPos, 0, buffer->head)) {
    while(buffer->index > sliderPos) {
      Undo(buffer);
    }
    while(buffer->index < sliderPos) {
      Redo(buffer);
    }
  }
}
```

We create a `SliderInt` which goes between 0 and the amount of Commands we've created. This lets us call `Undo` and `Redo` as we scrub the slider back and forth, letting us perform mass-undo and mass-redo operations by just sliding. Because over the course of a single tick, our `SliderPos` could jump more than 1 spot, we need to pt our undo/redo calls in while loops so that we keep calling them until `index` has caught up to the `sliderPos` . the `SliderInt()` function returns a bool that is true only if the slider has changed value since the last tick . This means that we only run the code inside the `{}` curly bracers of the if-statement if this was the case.
Lastly (for now) we'll display the games fps

```cpp
void DrawFPS(float dt) {
  ImGui::Text("FPS: %0.f", 1 / dt);
}
```

this formats the text to have no decimals and `1 / dt` gives us how many times `dt` goes into 1 aka how many frames can run per 1 second . also known as our frames per second
We have to pass along `dt` to this function somehow though. We will do this by adding a new variable to our `GameData`

```cpp
// gamestate.h
struct GameData {
  // other variables are just hidden for clarity
  const float* dt;
};
```

We make sure that our pointer to the memory where `dt` is stored is `const` this means that no code is allowed to change the value stored at the point in memory being pointed too. We do this as part of a defensive coding strategy, to help us catch if we would have accidentally modified this value through this pointer that we've just added for dev window conveniance.
in `main.cpp` right after we create our float `dt` before our main loop we assign its address to this pointer

```cpp
// main.cpp
float dt;
gameData->dt = &dt;
while(running) ...
```

this takes a pointer to `dt` and gives it to `gameData` by using the `&` get-pointer-to-symbol
finally with all this our `Draw()` function looks like this:

```cpp
// dev_gui.cpp
void DEV::Draw(GameData* data, SDL_Renderer* renderer) {
  ImGui::Begin("Dev Tools");
  ImGui::Text("memory arena usage");
  Draw_Imgui_Arena_Usage(data->arena_images, "images");
  Draw_Imgui_Arena_Usage(data->arena_levels, "levels");
  Draw_Imgui_Arena_Usage(data->arena_commands, "commands");
  Draw_Imgui_Arena_Usage(data->arena_entities, "entities");
  Draw_History(data->commandBuffer);
  DrawFPS(*data->dt);
  ImGui::End();
  ImGui::Render();
  ImGui_ImplSDLRenderer3_RenderDrawData(ImGui::GetDrawData(), renderer);
}
```

Now we need to call these `DEV::Functions()` from our `game.cpp` . So inside `game.cpp` we include `dev_gui.h` .
then we call `DEV::Initialize(window, renderer);` from our `void Initialize()` function.
But before we can do that we need to update our `Initialize()` in `game.h/cpp` to pass in `SDL_Window* window` as a new parameter

```cpp
// game.h
__declspec(dllexport) void Initialize(GameData* data, SDL_Window* window, SDL_Renderer* renderer);
```

> [!NOTE]
> On Linux, replace `__declspec(dllexport)` with `__attribute__((visibility("default")))` or use a CMake `set(CMAKE_CXX_VISIBILITY_PRESET hidden)` approach with explicit visibility. Alternatively, use a `-fvisibility=default` flag or a macro like `#define EXPORT __attribute__((visibility("default")))`.

and

```cpp
// game.cpp
void Initialize(GameData* data, SDL_Window* window, SDL_Renderer* renderer) {
  DEV::Initialize(window, renderer);
}
```

we call `DEV::ProcessEvents(&event);` at the top of `bool HandleEvents()`
and we update our `Draw()` to call both `DEV::PreDraw()` and `DEV::Draw()`

```cpp
// game.cpp
void Draw(GameData* data, SDL_Renderer* renderer) {
  DEV::PreDraw();
  SDL_SetRenderDrawColor(renderer, 120, 70, 120, 255);
  SDL_RenderClear(renderer);
  RenderLevel(data, renderer);
  RenderEntities(data, renderer);
  DEV::Draw(data, renderer);
  SDL_RenderPresent(renderer);
}
```

we need to do `DEV::Draw()` just before `SDL_RenderPresent()` and after our other Render functions to make sure the dev window gets rendered on top of everything else.
With this our new dev window works! we can drag it around and make it larger by dragging the bottom right corner.
there is only one problem now: if we perform a hot-reload the ImGui context that was created during `Initialize()` will dissapear, and because we don't rerun `Initialize()` during a hot-reload we won't have a context after the load and our program will crash.
Thankfully there's a simple fix!
We have to expand `GameData` with 1 new variable and put a safety check in during `PreDraw()`

```cpp
// gamestate.h
struct GameData {
  // other variables hidden for clarity
  ImGuiContext* imGui_context;
};
```

then in `game.cpp` during `Initialize()` we store a this context pointer in `GameData` that lives in our executable instead of our shared library

```cpp
// game.cpp
void Initialize(GameData* data, SDL_Window* window, SDL_Renderer* renderer) {
  DEV::Initialize(window, renderer);
  data->imGui_context = ImGui::GetCurrentContext();
}
```

Then we change the function signature of `PreDraw()` to take in a `ImGuiContext*` pointer

```cpp
// dev_gui.h
void PreDraw(ImGuiContext* saved_context);
```

> [!NOTE]
> we need to update the function signature in our `dev_gui.cpp` as well.

Then we pass the context we stored during `Initialize()` to `PreDraw()`

```cpp
// game.cpp
// inside void Draw()
DEV::PreDraw(data->imGui_context);
```

and finally in `PreDraw()` we check if our current context has been lost (is currently a pointer pointing to null aka nothing)

```cpp
// dev_gui.cpp
void DEV::PreDraw(ImGuiContext* saved_context) {
  if(ImGui::GetCurrentContext() == nullptr) {
    ImGui::SetCurrentContext(saved_context);
  }
  ImGui::NewFrame();
}
```

if the context was a `nullptr` we set it manually to the `ImGuiContext*` pointer we stored in `GameData` and passed to the `PreDraw()` function.
Now if we hot-reload our shared library and the context would get lost, we set it back.
And a `nullptr` check is very computationally cheap.
With this we've added Dear ImGui to our game engine and created our first dev tool!
Now as we expand our dev GUI we can visualize and help us build ANYTHING we want!