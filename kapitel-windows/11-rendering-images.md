# Rendering images

So far we've just rendered an SDL_FRect to the screen. But we of course want to have our own .PNG or .BMP files and use those. To accomplish this we must do the following things:

1. Put an image into our assets folder
2. Expand our cmakelists.txt to copy over our assets folder
3. Prepare a portion of memory to store our images
4. Use SDL3_Image.dll from its corresponding SDL3_Image.h file downloaded earlier
   > [!NOTE]
> In case you don't have both these files, then go ahead and download SDL3_image-devel-3.2.4-VC from https://github.com/libsdl-org/SDL_image/releases
5. Swap our SDL_FRect to a texture

Opening any drawing software we can create a 32x32px square and fill it with whatever shapes and colors we please. I've created a red square with an 'X' running through it. I've saved it as "fallback.png" as this will be the sprite that gets loaded whenever I attempt to load a sprite that doesn't exist. I do this so I can continue testing and developing even if I lack the necessary assets still.

Inside my assets folder I've created a subdirectory `sprites` and added my `fallback.png` to it.

## Updating our cmakelists.txt

At the top of our cmakelists.txt we will be adding the following code to specify the location of our assets and the place we want to copy them to. We already have a few set instructions up there so it's a nice place to keep storing them:

```cmake
set(DIR_ASSETS_ORIGIN ${CMAKE_SOURCE_DIR}/assets)
set(DIR_ASSETS_DESTINATION $<TARGET_FILE_DIR:${PROJECT_NAME}>/assets)
```

The syntax is a little bit more confusing when we have to store a yet-to-be-known path. The location where our .EXE and files will end up is specified by our cmakepresets.json and when we add another preset for release we will be targeting another folder instead of `build`. So to not hard-code our paths we use `$<TARGET_FILE_DIR:${PROJECT_NAME}>` to fetch the directory where the module called `${PROJECT_NAME}` ended up after compilation finished. So in my case that text gets replaced with `D:\PROJECTS\HEARTBURNER\build` because my cmakepresets.json sets `"binaryDir": "${sourceDir}/build"`.

`DIR_ASSETS_ORIGIN` points to the known path of our assets. In a project, our assets folder could eventually grow pretty sizeable, holding images, sound effects and music. Copying them over each time we compile will dramatically slow down our compile times. To get around this we will increase our `cmake_minimum_required(VERSION X.XX)` from 3.25 to 3.26 as cmake version 3.26 adds a very handy instruction `copy_directory_if_different` — this according to the cmake documentation (https://cmake.org/cmake/help/latest/manual/cmake.1.html) does the following:

> "Copy changed content of <dir>... directories to <destination> directory. If <destination> directory does not exist it will be created."

Then at the very bottom of our cmakelists.txt we add:

```cmake
add_custom_command(TARGET ${PROJECT_NAME} POST_BUILD
  COMMAND ${CMAKE_COMMAND} -E copy_directory_if_different ${DIR_ASSETS_ORIGIN} ${DIR_ASSETS_DESTINATION}
  VERBATIM
)
```

`add_custom_command(TARGET ${PROJECT_NAME} POST_BUILD(...)` sets up a new command that triggers on `POST_BUILD` meaning that only if the build was successful will this command fire. Inside its scope we run:

```
COMMAND ${CMAKE_COMMAND} -E copy_directory_if_different <source> <destination> VERBATIM
```

- `COMMAND ${CMAKE_COMMAND}` gets converted to just `cmake` but makes sure that the same cmake we used during our pre-compilation step is the one being used here. We could swap out `COMMAND ${CMAKE_COMMAND}` for just `cmake` but that runs the risk of causing issues down the road. So we do this little trade-off adding some more syntax to spare us a massive headache later.
- `-E` is a flag that stops cmake from running its usual build actions and instead executes the command as an internal command. Without it we would get stuck in a build-loop.
- `copy_directory_if_different` checks the time when a file was modified and compares it to any file it finds with the same name. If the timestamp is newer it overwrites the file with the new one. Otherwise it does nothing.
- `${DIR_ASSETS_ORIGIN} ${DIR_ASSETS_DESTINATION}` we specify first the path to our files and then the path to where we want them to get copied over to.
- `VERBATIM` this is a safety flag that makes sure that the paths we provided are treated "as is" and nothing gets changed. Some operating systems will treat spaces and special symbols as something to modify or discard, possibly altering the paths we've set. `VERBATIM` stops any of this from happening.

With this updated cmakelists.txt we can start working with asset files. And with us having the same folder setup in our build folder as in our root we can use common sense paths to access these.

The next step is taking our big monolith memory arena and placing another arena inside of it, segmenting a section of memory to be the exclusive area to hold pointers to our sprites.

> [!NOTE]
> sprites aka textures live on the GPU inside our VRAM compared to our game data that lives on the CPU. We will need to convert each .PNG file into `SDL_GPUTexture` storing it in VRAM and accessing it by pointer reference inside our memory arena.

> [!NOTE]
> we could store our sprites as just pixel data in the data type `SDL_Surface` but this would not allow us to use our expensive GPUs to process our images. Instead putting all that work on the CPU. For simple projects this will work. But it will not scale well, due to the CPU being much slower when it comes to pushing a lot of these images to the screen at once — this is expressly the task that a GPU was created to do.

At the top of our `main()` we will be adding a new memory arena by allocating it directly inside our top-level memory arena. When we created our first memory arena we used:

```cpp
Memory::Arena* arena_main = new Memory::Arena();
```

The `new` keyword creates a struct of the specified type and returns a pointer to it. It places this piece of memory somewhere on the heap. But our new memory arena will not be created in the same way, instead we will allocate it directly into this first `arena_main`:

```cpp
void* game_memory = AllocateGameMemory();
if(game_memory == nullptr){
  return 1;
}
Memory::Arena* arena_main = new Memory::Arena();
Memory::Initialize(arena_main, game_memory, GAME_MEMORY_ALLOWANCE);
GameData* gameData = (GameData*)Memory::Allocate(arena_main, sizeof(GameData));
Memory::Arena* arena_image = (Memory::Arena*)Memory::Allocate(arena_main, sizeof(Memory::Arena));
void* image_memory_start = Memory::Allocate(arena_main, GAME_MEMORY_IMAGES);
Memory::Initialize(arena_image, image_memory_start, GAME_MEMORY_IMAGES);
```

Lets look closer at the allocation of the memory arena that will hold our images:

```cpp
Memory::Arena* arena_image = (Memory::Arena*)Memory::Allocate(arena_main, sizeof(Memory::Arena));
void* image_memory_start = Memory::Allocate(arena_main, GAME_MEMORY_IMAGES);
Memory::Initialize(arena_image, image_memory_start, GAME_MEMORY_IMAGES);
```

Our first `Allocate()` function returns a `void*` to the first byte of memory, we then use a cast to convert this `void*` to a `Memory::Arena*`. This means that the block of memory allocated will be stored in our variable `arena_image` and due to us specifying `sizeof(Memory::Arena)` we can be sure that we allocated enough space for it.

> [!NOTE]
> This first allocation only adds the arena struct itself into memory. The arena struct is just:

```cpp
struct Arena {
  unsigned char* base;
  size_t size;
  size_t used;
};
```

So we have not put the block of memory to store images, just the arena responsible for knowing about and adding to their place in memory.

The second `Allocate()` actually commits a block of memory inside `arena_main` to store the image arena. We then `Initialize()` the `arena_image` so it knows about this block of memory it has been given to manage.

Now we can free all memory inside our `arena_image` by just setting the `used` variable back to 0. Without this sub-arena we could only free the entire `main_arena` all at once.

Ok! Now we have a place in memory to store our image pointers. Now we have to create them and store them.

We will set up a new struct `Image` to hold the relevant variables. We will be storing this in a new `image.h` file:

```cpp
#pragma once
#include "SDL3/SDL_render.h"
#include "arena.h"
struct Image{
  SDL_Texture* texture;
  int width;
  int height;
};
namespace AssetManagement
{
  Image* LoadSprite(Memory::Arena* arena, SDL_Renderer* renderer, const char* path);
}
```

Our struct holds a pointer to an `SDL_Texture` — we will, inside our `LoadSprite()` function fetch a .PNG file and convert it to this `SDL_Texture` format. It is a required step in order for a `SDL_Renderer` to draw it on the screen. We also store the width and height of the image alongside the `SDL_Texture` so we know how large it is when drawing it. We are putting our `LoadSprite()` function inside a namespace to avoid the naming conflict from other headers having functions with the exact same name.

Our `LoadSprite()` function accepts 3 parameters:

1. The arena to store the `Image` struct in
2. A pointer to the renderer that will be used during conversion from PNG to Texture
3. The file path for the .PNG

Our textures will live in VRAM on our GPU, and our `arena_image` will hold `Image` structs in sequence, each pointing to a different texture in VRAM.

The problematic part about creating `SDL_Textures` on the GPU is that we are not directly in control of that memory — the GPU will place these textures in VRAM in places it finds appropriate. We then point at it to use it. If our game is very simple we could even forgo using the GPU entirely, relying instead on the CPU to push pixels to the screen: in this case we use the intermediate `SDL_Surface` instead of `SDL_Texture`. Though using the CPU to put pixels on the screen is a lot slower than having the GPU do it, but it would allow us to store each `SDL_Surface` in our memory arena directly. If we want to have the GPU do the heavy lifting AND have complete control of how our sprites are stored in VRAM then we would use a Graphics API like Vulkan — though this increases coding complexity significantly compared to the other two methods.

Next we will need to add the actual code to `LoadImage`, then use it to create our texture, store it in VRAM and add its corresponding `Image` struct to our `arena_image`:

```cpp
#include <cassert>
#include <string>
#include "SDL3/SDL_render.h"
#include "SDL3_Image/SDL_image.h"
#include "image.h"
#include "arena.h"
using namespace std;
const char* DIRECTORY = "assets/sprites/";
const char* FALLBACK = "assets/sprites/fallback.png";
Image* AssetManagement::LoadSprite(Memory::Arena* arena, SDL_Renderer* renderer, const char* name){
  string path = DIRECTORY;
  path = path.append(name);
  SDL_Surface* surface = IMG_Load(path.c_str());
  if(surface == nullptr){
    surface = IMG_Load(FALLBACK);
  }
  assert(surface != nullptr);
  SDL_Texture* texture = SDL_CreateTextureFromSurface(renderer, surface);
  Image* img = (Image*)Memory::Allocate(arena, sizeof(Image));
  img->texture = texture;
  img->height = texture->h;
  img->width = texture->w;
  SDL_DestroySurface(surface);
  return img;
}
```

A `string` is very similar to a `char*`, meaning that it stores a sequence of text. But the `string` data type has a lot of quality-of-life functionality. We will be using it to allow ourselves to not pass in the full path to our .PNG but instead just its name — e.g. `example_sprite.png` rather than `assets/sprites/example_sprite.png`.

First we create the full path to the file using a `string`. We first set it to be text stored in `DIRECTORY` aka `assets/sprites/` then we use the `append()` function to add our `name` to the end of the text. But `IMG_Load()` doesn't support us passing it a `string` as a parameter, it only accepts a `char*`. The `string` data type comes with a handy function `c_str()` — this converts the `string` to a `char*` allowing us to pass it into `IMG_Load()`.

> [!NOTE]
> if we don't type `using namespace std` after our `#includes` then we can only access the `string` data type by referencing its namespace first: `std::string`

We will be loading some (or maybe all) our sprites inside our .EXE and depending on our setup we might want to load some inside our .DLL as well. This means that our `image.cpp` and — in case we want to modify our arena(s) inside our .DLL — we will need to make sure that both our .EXE and our .DLL have access to these scripts. We will modify our cmakelists.txt to create a list of these shared scripts.

```cmake
# Flag .EXE specific files
set(EXE_EXCLUSIVE ${CMAKE_SOURCE_DIR}/src/main.cpp
)
# Flag files that both our .EXE and .DLL need to know about
set(SHARED_SOURCES
  ${CMAKE_SOURCE_DIR}/src/image.cpp
  ${CMAKE_SOURCE_DIR}/src/arena.cpp
)
```

I have put each script reference on its own line, but that is not any different from just creating the worlds longest one-liner.

We can then create a new .DLL that we declare to be STATIC instead of SHARED as our game .DLL is. When we make the .DLL static it will share its contents with all other things we create.

> [!NOTE]
> when we mark a DLL as STATIC it will be copied into both our .EXE and our .DLL, meaning that its contents will be found in both and the .DLL will not actually live inside our build folder, it is absorbed into them instead. We could mark our .DLL as SHARED instead of STATIC but then we would have to work with `__declspec(dllexport)` and stuff to access its functions. And that would not make any meaningful difference to our architecture right now.

Inside our cmakelists.txt we first need to create a variable to store its name:

```cmake
set(SHARED_LIB_NAME ${PROJECT_NAME}_common)
```

Then we create it:

```cmake
# SHARED STATIC DLL
add_library(${SHARED_LIB_NAME} STATIC ${SHARED_SOURCES})
target_include_directories(${SHARED_LIB_NAME} PRIVATE include)
```

This will make sharing scripts between our .EXE and .DLL easier as we can add them to the `SHARED_SOURCES` and then we're done.

With this setup we can now use our `image.cpp` and `arena.cpp` scripts inside both our EXE and our DLL — allowing us to create `Image` structs in both and pass the memory arenas to our DLL if we want our DLL to allocate pointers and structs to them.

`IMG_Load` is part of the SDL_image .DLL that we downloaded earlier, without it we don't have the ability to convert any other image format other than .BMP to a `SDL_Surface`.

We want to be able to create the loading logic for sprites that are yet to be added to our assets folder. We accomplish this by having a fallback sprite that we load each time our `LoadSprite()` function fails to find the specified .PNG.

```cpp
if(surface == nullptr) {...}
```

This if-statement is responsible for checking if what was returned from `IMG_Load()` was an `SDL_Surface` or if it failed and returned a `nullptr` instead. If that happens we instead try and load the fallback .PNG.

We then use something new: an **assertion**. An assertion is similar to an if-statement in that we ask a question, in this case we use the not-operator `!` to assert that `surface` is in fact not a `nullptr` any longer. If it was a `nullptr` that means that neither our specified .PNG or our fallback .PNG managed to get loaded. At this point we want our program to fail. The `assert` will do just that. We forcibly decide that we will terminate our program so that we can catch these fundamental issues rather than having them fly under the radar.

Afterwards we take our `SDL_Surface*` and using our `SDL_Renderer*` we convert it to an `SDL_Texture` that then gets automatically allocated to VRAM.

```cpp
Image* img = (Image*)Memory::Allocate(arena, sizeof(Image));
img->texture = texture;
img->height = texture->h;
img->width = texture->w;
```

We then store an `Image*` inside our `arena_image` and assign the `SDL_Texture` and its information to the variables held inside the `Image` struct.

Once we are done with that we have one job left to do — we have created a new `SDL_Surface` inside memory. Once this function terminates this point in memory is still allocated. Meaning that if this function ran enough times, our memory would be filled up with no longer necessary `SDL_Surfaces`. Calling `SDL_DestroySurface()` lets us free that part of memory. In a more robust setup we would actually create another memory arena to hold this temporary data. If we do that then no part of our program would allocate any memory on the CPU other than the initial block of memory. We'll do this later in the course, for now we accept this temporary allocation.

At the end of the `LoadSprite()` function we return our `Image*` so that whatever system wanted to use it can do so.

Inside our `GameData` struct we could store a series of `Image*` — for a very small game like asteroids or pong it would be simple to have all the sprites referenced right there:

```cpp
Image* ship;
Image* asteroid_big;
Image* asteroid_small;
Image* background;
```

> [!NOTE]
> we're not really adding these `Image*` — they are just here to illustrate a concept.

But for larger projects we would do one of the following instead:

1. Pass the image memory arena as part of the `GameData`, allowing whatever piece of code that wants to load and allocate an `Image*` to do so at runtime. It still lives in the memory held by our executable, so hot-reloading still works. But the VRAM costs would be unknown after initialization and we could overflow the VRAM memory in a (much) larger project.
2. We create a central storage for all our images, loaded into memory during initialization. This is just a fancier way of finding them later, and skips the part where we have to explicitly type out each individual image variable ourselves, instead looping over each .PNG in our assets folder and based on its name we can find the corresponding `SDL_Texture` by pointer reference later.

For right now, lets add our fallback texture to our `GameData` and expand our `Draw()` to render our .PNG instead of the default rectangle.

```cpp
#pragma once
#include "SDL3/SDL_rect.h"
#include "image.h"
struct GameData {
  SDL_FRect rect;
  float move_speed;
  Image* fallback;
};
```

Now lets use our memory arena and our new `LoadSprite()` function. Inside our `main()` just after we've initialized our `arena_image` and after we've done our `SDL_Setup()`. If we try to run the code before our `SDL_Setup()` then our `SDL_Renderer` is not initialized and we can't pass it as a parameter.

```cpp
gameData->fallback = AssetManagement::LoadSprite(arena_image, renderer, "fallback.png");
```

Note how we need to specify that the `LoadSprite()` function comes from our `AssetManagement` namespace.

We should also update the size of our `arena_image` during initialization to a more reasonable size, as we know that it will only store `Image`:

```cpp
size_t IMAGE_ARENA_SIZE = sizeof(Image) * 1024;
Memory::Arena* arena_image = (Memory::Arena*)Memory::Allocate(arena_main, sizeof(Memory::Arena));
void* image_memory_start = Memory::Allocate(arena_main, IMAGE_ARENA_SIZE);
Memory::Initialize(arena_image, image_memory_start, IMAGE_ARENA_SIZE);
```

This allows us to store 1024 `Image` structs inside the arena. If we ever had 1025 `Image` structs they would not fit and we would have to give it more memory.

Now we can move to the `Draw()` function inside our `game.cpp`. Here we can swap the code that drew our `Rect` for our fallback texture:

```cpp
void Draw(GameData* data, SDL_Renderer* renderer){
  SDL_SetRenderDrawColor(renderer, 0, 70, 8, 255);
  SDL_RenderClear(renderer);
  SDL_RenderTexture(renderer, data->fallback->texture, NULL, &data->rect);
  SDL_RenderPresent(renderer);
}
```

The third parameter of the `SDL_RenderTexture()` function allows us to specify a region of our texture to render instead of the whole thing. Passing in `NULL` is treated as us wanting to draw the entire thing. Our sprite, when converted to a `SDL_Texture` is presented on a quad — this can be stretched and modified a bunch. So right now we pass our `rect` from our `gameData` as the fourth parameter, this allows us to specify the width and height of the resulting quad meaning that we can shrink and grow the texture as we see fit. Which is great if we want to create small for example pixel-art that we then scale up 10x to actually be rendered in an appropriate size.

Lets update our logic to make a simplified rendering function that any object tasked with rendering textures to the screen can use. We will be creating a new file `rendering.h`:

```cpp
#pragma once
#include "SDL3/SDL_render.h"
void RenderSprite(Image* sprite, SDL_Renderer* renderer, int xPos, int yPos, float scale = 1);
```

We create the function signature for a Render function, passing all the relevant variables. As well as setting a default value for the `float` variable `scale`. This means that we can omit passing this when calling the function and the function will give it a default value of 1.

This allows the following two calls to be functionally identical:

```cpp
RenderSprite(sprite, renderer, 50, 20);
RenderSprite(sprite, renderer, 50, 20, 1);
```

The shorter call also has the scale set to 1 but this is done behind the scenes.

And in a newly created `rendering.cpp` we will write the `RenderSprite()` function:

```cpp
#include "rendering.h"
#include "SDL3/SDL_render.h"
void RenderSprite(Image* sprite, SDL_Renderer* renderer, int xPos, int yPos, float scale){
  SDL_FRect rect;
  rect.x = xPos;
  rect.y = yPos;
  rect.h = sprite->height * scale;
  rect.w = sprite->width * scale;
  SDL_RenderTexture(renderer, sprite->texture, NULL, &rect);
}
```

We create a `SDL_FRect` that will be passed as the fourth parameter to the `SDL_RenderTexture()` function. We position the rect at the specified `xPos` and `yPos` then fetch the size of the texture and multiply both height and width by `scale`. So if nothing was passed to the function then we will be multiplying them by 1, making the height and width be their unmodified defaults. `SDL_RenderTexture()` expects the destination rect to be passed as a `SDL_FRect*` so we need to pass a pointer to the rect by adding the address-of operator `&` along with the parameter. Reading the SDL Documentation is the best way of learning these rules.

In a later lecture we will create another identically named function to `RenderSprite` that lets us pass even more parameters that we can use to render only a portion of the texture to the destination rect. This will be useful to allow us to fit multiple frames of a sprite animation inside a single .PNG.

Inside `game.cpp` our `Draw()` function now reads:

```cpp
void Draw(GameData* data, SDL_Renderer* renderer){
  SDL_SetRenderDrawColor(renderer, 0, 70, 8, 255);
  SDL_RenderClear(renderer);
  RenderSprite(data->fallback, renderer, 50, 50);
  SDL_RenderPresent(renderer);
}
```

Short and sweet. Now we are able to keep allocating more `Image*` in our memory arena then passing them to our `game.cpp` living inside our .DLL by adding them to `GameData`. With only minimal work we could start to envision how we could make a visual novel if we could only render some text and control the game state more. So not quite there yet, but we are well on our way.

We're currently letting our application run wild, spinning our game loop as fast as it possibly can. Meaning that a few things are true:

1. Our framerate is as high as possible
2. Our framerate will vary more as the CPU is tasked with doing more on certain ticks and less on others
3. We tax our CPU maximally with minimal upside

We will be implementing an enforced framerate. To do this we need to know how long a frame took in milliseconds to process then we need to stop the program from just going to the next tick, instead waiting until the specified time has elapsed. For a 60FPS game each tick gets 1/60 seconds to process. We would rather enforce a stable framerate than hit frame-spikes.

Our CPU can `sleep()` for a specified number of milliseconds, but there are some nasty quirks of this system. The most egregious being that we can't ensure that the time we specify will be the exact time it sleeps — it will schedule the thread to continue as close to the specified millisecond as possible, but it might overshoot. Our delta time will offset the effect of this being less than absolutely consistent. But we can implement a helpful pattern that will help us begin the next tick exactly on time.

First we will check if we've already taken longer than 1/60 seconds to process the current tick. If we still have time remaining we will sleep for a portion of the remaining time, then spin for the last couple of milliseconds. This ensures that our sleep doesn't overshoot and we tax the CPU only the amount necessary. If we can't hit our target framerate then we should lower it, as a stable 30 is preferred to an unstable 60.

Out of the box our `sleep()` function has a very rough granularity, and even if we specify a very precise number it will not go out of its way to work with very fine-tuned numbers. We will need to give more fidelity to the `sleep()` function by expanding our cmakelists.txt to add `winmm.lib` — this is an old Windows API that allows us to call `timeBeginPeriod()` a function that controls the built in Windows timer resolution. The finest resolution is setting the granularity to 1ms:

```cpp
timeBeginPeriod(1)
```

In our `main()` function, after we've initialized our memory arenas we can add the following:

```cpp
MMRESULT result = timeBeginPeriod(1);
if(result == TIMERR_NOCANDO){
  printf("could not increase timer resolution");
  Sleep(2000);
  return 3;
}
```

We could add only `timeBeginPeriod(1)` without the extra defensive code, but then this change to the Windows timer resolution could fail silently. I find it's better to ensure the change worked and if not we can close the application. We could also use `assert` here instead: `assert(result != TIMERR_NOCANDO)`.

> [!NOTE]
> yes, the name of the result enum is very goofy.

Inside our cmakelists.txt we need to add `winmm.lib`:

```cmake
set(WINDOWS_LIBRARIES winmm)
```

But as this is part of Windows we don't have to add this .lib file to our include folder. We can just add it inside our cmakelists.txt:

```cmake
target_link_libraries(${PROJECT_NAME} PRIVATE
  ${LIB_FILES}
  ${WINDOWS_LIBRARIES}
  ${SHARED_LIB_NAME}
)
```

We store `winmm` inside a variable we name `WINDOWS_LIBRARIES` — we do this so we can add more of these libraries in one location then use the variable to add all of them at once. Then for only our .EXE we will add it. Because our game .DLL does not need to know about this code, we can skip linking this .lib file into it. We might come across a future where our .DLL needs this or another Windows library, but because we've written our cmakelists.txt from scratch such a change is no longer scary.

Now we need to know how much time a tick took to process. Inside our `while(running)` game loop, after we've called `dll.draw()` we can, just as in our `CalculateDeltaTime()`, fetch the time in nanoseconds since startup using `SDL_GetTicksNS()`:

```cpp
Uint64 frame_end_time_ns = SDL_GetTicksNS();
```

Then we remove the value stored in our `PREV` variable, which has the nanoseconds since startup stored just before we run our update and draw (its value gets set in our `CalculateDeltaTime()`). We store the result in a new variable `frame_time_spent_ns`. But in our `common.h` we store the target frame time in milliseconds not nanoseconds.

```cpp
double frame_time_spent_ns = frame_end_time_ns - PREV;
```

We could change our `common.h` to include the calculations for `FRAME_TIME_NS` alongside `FRAME_TIME_MS` but I think this is bad practice for 2 reasons — NS is very close to MS and I can very much see myself making that mistake over and over again. We will only be needing the nanosecond version in this one location, so why pollute our code base with this extra variable.

We then convert to the millisecond representation of our `frame_time_spent_ns` in a new variable:

```cpp
double frame_time_spent_ms = frame_time_spent_ns / 1e6;
```

`1e6` is the programmatic representation of a 1 followed by six zeroes `000000` aka one-million. Taking a number in nanoseconds and dividing it by 1,000,000 gives us the value in milliseconds instead.

We then, using our `FRAME_TIME_MS` as the maximum time to (hopefully) spend on a frame, then subtract the time in milliseconds that the frame actually took. Storing the result once again in a new variable. We could do some of these calculations all in a row on the same variable, but I've chosen to break it out like this for clarity.

```cpp
double time_to_sleep_ms = FRAME_TIME_MS - frame_time_spent_ms;
```

If we took longer than our allowed time budget then `frame_time_spent_ms` will be larger than `FRAME_TIME_MS` and our `time_to_sleep_ms` will hold a negative value. This allows us to use an if-statement to check the value before deciding what to do with it:

```cpp
if(time_to_sleep_ms > 0){
  SDL_Delay(time_to_sleep_ms);
}
else{
  printf("missed frame \n");
}
```

`SDL_Delay()` does the same thing as `Sleep()` but will work cross-platform. `Sleep()` only works on Windows.

Now we have, with some (granularity issues still) a program that tries to wait for the specified time to elapse before running the next frame. The longer we want the time between frames to be, the larger delta time grows, which we use to scale all movement, so that after 1 second has elapsed, no matter the framerate we still get the correct movement for all objects. You can experiment with this now by setting the FPS inside `common.h` to a really low value like 8 and watch as the program starts stuttering.

We can then move the code calculating our `time_to_sleep_ms` into a function, helping us keep our main loop clean. I happen to know that we will soon be needing this same code again, meaning that a function is very appropriate, but if this truly was a one-off block of code then having it live right where it is being used is often preferred.

```cpp
void CalculateRemainingFrameTime_MS(double* milliseconds){
  Uint64 frame_end_time_ns = SDL_GetTicksNS();
  double frame_time_spent_ns = frame_end_time_ns - PREV;
  double frame_time_spent_ms = frame_time_spent_ns / 1e6;
  *milliseconds = FRAME_TIME_MS - frame_time_spent_ms;
}
```

Then we could remove this calculation from our `while(running)` loop and just create a `double` to pass to the function. We create the `double` outside of the function and pass it in so the value can be assigned inside the function. Another method would be to change the signature of the function to return a `double` then instead of passing in a `double` we would assign to a `double` the value returned from the function. But in our case our new delay logic looks like this:

```cpp
double time_to_sleep_ms;
CalculateRemainingFrameTime_MS(&time_to_sleep_ms);
if(time_to_sleep_ms > 0){
  SDL_Delay(time_to_sleep_ms);
}
else{
  printf("missed frame \n");
}
```

The logic is the same, but we've decided that keeping our `while(running)` loop shorter is worth the obfuscation that comes with breaking the logic out into its own function. It's a trade-off and different programmers will value these things differently.

We have one more issue we need to fix. We can't be sure that our `SDL_Delay()` actually perfectly hits our specified millisecond — making the timer more granular has helped this but not completely fixed it. We will be expanding the logic to spin for the last couple of milliseconds and not relying on `SDL_Delay()` to hit our mark. A **spin** or **spinning** is just another `while()` loop that runs, checking if we've reached our desired timestamp yet. This costs more CPU as it will run as many times as it needs, but this increased CPU load is well worth it as hitting our framerate should be an absolute goal of our program.

```cpp
double time_to_sleep_ms;
CalculateRemainingFrameTime_MS(&time_to_sleep_ms);
if(time_to_sleep_ms > 0){
  if(time_to_sleep_ms > 1){
    SDL_Delay(time_to_sleep_ms - 1);
  }
  while (time_to_sleep_ms > 0) {
    CalculateRemainingFrameTime_MS(&time_to_sleep_ms);
  }
}
else{
  printf("missed frame \n");
}
```

We now check if we have more than 1 millisecond of sleeping to do and only then do we `SDL_Delay()` — but(!) we sleep for 1 millisecond less than we will need, leaving a portion of very granular time left. This value will almost never be exactly 1 as we can't get that specificity from our `SDL_Delay()`. Then inside a while-loop we recalculate `time_to_sleep_ms` and once we've hit our target (0 or lower) we stop our while loop allowing the program to move on to the next frame.

And here we can see how we further managed to reduce both reading complexity and code duplication by using our `CalculateRemainingFrameTime_MS` function in another location. Now we can confidently say that moving this logic into its own function was the right call.

`printf(...)` is actually a pretty expensive function, so this could cause us to keep missing our frame window. We will move on to a more robust system of viewing our framerate in a later part of the course.

Now our application enforces a stable framerate!

A lot of what we do in our `main.cpp` is boilerplate code that will be reusable in just about every project.
