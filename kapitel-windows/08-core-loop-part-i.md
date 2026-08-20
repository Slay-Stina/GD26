# Core Loop - Part I

With the skillset we have currently we can begin constructing a core loop for a program.

> [!NOTE]
> As it lacks a win-state or any sense of actual logic we'll call it a program for now and a game once we add those elements.

We're going to create a skeleton version of our core loop, including:

1. Receiving inputs
2. Updating logic based on time and our inputs
3. Displaying our game state

We will be setting up parts of this in our `main.cpp` but we'll also create other files that our `main.cpp` will call into.

> [!NOTE]
> Once we have gotten this core loop to work we will be changing a lot (almost all) of how we structure our program in the next couple of lectures. We will be re-writing things a couple of times, each time digging deeper into performance-focused C++ code!

At the end of this lecture we will have a colored rectangle that we can control on the screen using the arrow keys.

We could do all the following code inside our `main()` function, inside our while-loop. But we will be creating a new pair of .h and .cpp files and inside our while-loop we will call their functions.

Let's look at our `main.cpp` function:

```cpp
// main.cpp
#include "SDL3/SDL_init.h"
#include "SDL3/SDL_events.h"
#include "SDL3/SDL_timer.h"
#include "game.h"
SDL_Window* window;
SDL_Renderer* renderer;
Uint64 NOW;
Uint64 PREV;
bool HandleRunning(SDL_Event event){
    if(event.type != SDL_EVENT_KEY_DOWN){
        return true;
    }
    if(event.key.key == SDLK_ESCAPE){
        return false;
    }
    else{
        return true;
    }
}
int main() {
    SDL_Init(SDL_INIT_EVENTS);
    bool running = true;
    window = SDL_CreateWindow("pilot", 650, 400, 0);
    renderer = SDL_CreateRenderer(window, NULL);
    Core::Initialize();
    while(running){
        NOW = SDL_GetTicksNS();
        float dt = NOW - PREV;
        dt = SDL_NS_TO_SECONDS(dt);
        PREV = NOW;
        SDL_Event event;
        while(SDL_PollEvent(&event)){
            running = HandleRunning(event);
        }

        Core::Update(dt);
        Core::Draw(renderer);
    }

    Core::OnQuit(renderer);
    SDL_Quit();
    return 0;
}
```

A lot of new stuff happening:

1. We `#include` new .h files — the `game.h` we wrote ourselves. We've also written a `game.cpp` file that has the actual implementations of each function outlined in `game.h`.
2. We store pointer variables to both a **window** and a **renderer**. The renderer is tasked with taking textures and bitmap images and placing them into our window — this is how we will color our window and render a rectangle inside of it. This logic is found inside the `Draw()` function we've written inside `game.cpp` and declared inside `game.h`. It is because we `#include game.h` that we can find and call this function. Note that we pass our renderer pointer to the `Draw()` function.
3. We have 2 new variables `NOW` and `PREV`. These are used to track how much time elapsed between the current and previous frame. We check this by subtracting one from the other. The `Uint64` is like an `int` but can only hold positive values. It is also 64 bits in memory compared to the (usually) 32 bits of an `int`, meaning that it can store larger numbers. The `U` stands for **"unsigned"** — this means it only holds positive numbers. Note how we pass deltatime (aka `dt`) to our `Update` function.
4. We use SDL functions `SDL_GetTicksNS()` and `SDL_NS_TO_SECONDS()` to work with a central part of all game logic: `dt` standing for **deltatime**. Deltatime is used to scale values in relation to how quickly the computer can finish processing a tick. The more ticks, the higher the framerate and the smaller our deltatime is. Delta time is the time between the current and the last tick. Meaning that if it took a long time between ticks, then any equation that is multiplied by deltatime will be larger than if the time between ticks was very small. The result of this is that no matter how strong or how slow our computers are, our bullets will still fly at the same speed. Without deltatime, a gun on a fast computer would shoot faster bullets.
5. We call `Initialize`, `Update` and `Draw` from a namespace we've named `Core`.

Let's begin by looking at our `game.h`:

```cpp
#include "SDL3/SDL_render.h"
namespace Core{
    void Initialize();
    void Update(float dt);
    void Draw(SDL_Renderer* renderer);
    void OnQuit(SDL_Renderer* renderer);
}
```

This .h file outlines the functions that we will be writing the bodies for inside our `game.cpp`. It tells us what parameters will be passed in and what type of function they are. `void` means that the function doesn't return any value. Because we know we will need to pass a pointer to the renderer in two of these functions we have to `#include` the `SDL_render.h` inside our .h file. This means that all files that include `game.h` also include `SDL_render.h`.

All functions are collected in a **namespace** — a namespace acts as a container for code, allowing multiple scripts to have the same name for functions. Imagine if we include 2 .h files, each with their own `Initialize()` function. Without a namespace we would get an error during compilation telling us that it is unclear which function should be called. But keeping our `Initialize()` function inside a namespace forces us to specify the namespace as we call the function. We have already encountered a namespace earlier in this lecture series, when we decided to write the handy `using namespace std;` — this allowed us to call the functions inside the namespace named `std` without first writing `std::`.

We can write `using namespace Core;` at the top of our `main.cpp` and remove the `Core::` prefix from all function calls if we want.

> [!NOTE]
> In this project we don't have any other functions with these same names, so removing the namespace entirely would not cause compile errors.

Each function will have the following job:

- **Initialize()** → Set up the necessary stuff
- **Update()** → Perform changes to the game using deltatime and keyboard inputs
- **Draw()** → With the changes from Update, render the relevant stuff to the screen
- **OnQuit()** → Clean up before the application quits

Let's look at our `game.cpp` to find how each of these functions are implemented:

```cpp
// game.cpp
#include "game.h"
#include "SDL3/SDL_keyboard.h"
#include "SDL3/SDL_render.h"
float xPos;
float yPos;
constexpr float SPEED = 100;
SDL_FRect box;
void Core::Initialize(){
    xPos = 100;
    yPos = 100;
    box.h = 50;
    box.w = 50;
    box.x = xPos;
    box.y = yPos;
}
void Core::Update(float dt){
    auto keys = SDL_GetKeyboardState(NULL);
    if(keys[SDL_SCANCODE_RIGHT]){
        xPos += SPEED * dt;
    }
    if(keys[SDL_SCANCODE_LEFT]){
        xPos -= SPEED * dt;
    }
    if(keys[SDL_SCANCODE_UP]){
        yPos -= SPEED * dt;
    }
    if(keys[SDL_SCANCODE_DOWN]){
        yPos += SPEED * dt;
    }
    box.x = xPos;
    box.y = yPos;
}
void Core::Draw(SDL_Renderer* renderer){
    SDL_SetRenderDrawColor(renderer, 0, 70, 0, 255);
    SDL_RenderClear(renderer);
    SDL_SetRenderDrawColor(renderer, 150, 0, 30, 255);
    SDL_RenderFillRect(renderer, &box);
    SDL_RenderPresent(renderer);
}
void Core::OnQuit(SDL_Renderer* renderer){
    SDL_DestroyRenderer(renderer);
}
```

At the top of `game.cpp` we write our list of variables that will be used: two for the position of our box, one for the actual box itself, and one is a `constexpr` variable called `SPEED`. Adding `constexpr` to a variable means that its value can't change when the program is running. Meaning that nothing or no one could accidentally write code that changes this value. The only place where the value of a `constexpr` can be set is at the same line it is being created.

Our `Initialize()` function takes all of our variables (except `SPEED`) and gives them an initial value. It also sets the internal values of `x` and `y` of the `SDL_FRect` to match our `xPos` and `yPos` variables as well as its width and height `w` and `h`.

An `SDL_FRect` is a **struct** holding the following data:

- `float x;` — horizontal position
- `float y;` — vertical position
- `float w;` — width
- `float h;` — height

A struct is just a collection of variables that we want to bundle together. An apple struct could have the following variables:

```cpp
struct Apple {
    int price;
    float weight;
    string speciesName;
    bool isRotten;
};
```

Using structs is how we pass more complex data around without sending each variable one after the other.

SDL can, using the info inside a `SDL_FRect` and the renderer, display a rectangle on the screen using a function called `SDL_RenderFillRect()` inside our `Draw()`.

Just like how we can create a new function or a new variable we can also create structs ourselves. The syntax is very simple:

```cpp
struct the_structs_name{
    int a_number;
    float a_decimal_number;
    bool a_true_or_false_thing;
};
```

That's it, now we can pass and store multiple variables together. This will be used later to define things like bullets, enemies, players and more!

For example:

```cpp
struct Bullet {
    int damage;
    bool fired;
    float travel_speed;
    float size;
};
```

When we work with a struct, we can access its different internal variables by just using a `.`:

```cpp
Bullet a_bullet;
a_bullet.damage = 5;
a_bullet.fired = false;
a_bullet.travel_speed = 100;
// and so on
```

And if a function expects a `Bullet` struct we can pass it like so:

```cpp
void FireBullet(Bullet* a_bullet){
}
```

But as we're passing our bullet struct as a pointer instead of by value we do have to learn a bit more unintuitive C++ syntax. To access the values saved inside a pointer we can't use `.` — we need to use `->`:

```cpp
void FireBullet(Bullet* a_bullet){
    a_bullet->fired = true;
}
```

So we use:

- `.` when accessing the values on the actual variable
- `->` when accessing the values the pointer is pointing to
- `::` when accessing functions from a namespace

Let's look at our `Update()` function:

```cpp
void Core::Update(float dt){
    const bool* keys = SDL_GetKeyboardState(NULL);
    if(keys[SDL_SCANCODE_RIGHT]){
        xPos += SPEED * dt;
    }
    if(keys[SDL_SCANCODE_LEFT]){
        xPos -= SPEED * dt;
    }
    if(keys[SDL_SCANCODE_UP]){
        yPos -= SPEED * dt;
    }
    if(keys[SDL_SCANCODE_DOWN]){
        yPos += SPEED * dt;
    }
    box.x = xPos;
    box.y = yPos;
}
```

The first line holds a lot of new info for us: `const bool* keys` is similar to `bool* keys` which is a pointer to a place in memory where we store a bool. Adding `const` to it makes it so that the value of the bool stored in memory at the address the pointer points to can't be changed — SDL does this because we are not supposed to change the status of the keyboard in code, we just read what it is.

But there is one more wrinkle. As SDL is written in C and not C++ we have some unintuitive syntax here. `bool* keys` (note the plural) is actually a pointer to the first bool out of many stored in sequence in memory. So if we had a micro-keyboard with only A B C D E, then if we held the B key down our memory would look like this:

```
TrueOrFalse:  [0] [1] [0] [0] [0]
Key:           A   B   C   D   E
Index:         0   1   2   3   4
```

Our `bool*` points to the first memory address (A) then we can check the value of a specific memory address using `[]` and an index:

- `keys[0]` (this is the true or false for `A`)
- `keys[1]` (this is the true or false for `B`)
- `keys[4]` (this is the true or false for `E`)

The first memory address is at index 0, not index 1. C++ and C# begin indexing at 0. Other languages like LUA for example start at 1.

But it would be really hard to remember the index of let's say the letter `U`. Thankfully SDL has an enum that helps us write in plain English and have the compiler substitute that for a number:

```cpp
keys[SDL_SCANCODE_B]
```

Scancodes — it turns out the index for B was 5 and that is exactly what the number for `SDL_SCANCODE_B` was given.

So each tick SDL checks the status of all the keys on our keyboard then sets their value in memory to `true` or `false`, then we can check their status by pointing to the first place in memory then shifting by the specific index to find the value of the key we are looking for.

```cpp
if(keys[SDL_SCANCODE_RIGHT]){
    xPos += SPEED * dt;
}
```

Here we check if the right arrow key is held down, and if it is we add to the `xPos` variable equal to our `SPEED` variable scaled by deltatime. This ensures that a fast and a slow computer will have the same speed applied to `xPos` over time.

We do the same for the other arrow keys. The only quirk is that the top left corner of our window is at position `0,0`. So when we increase the value of `yPos` we actually move downwards. So we need to remember that when selecting to add `+=` or to subtract `-=`.

Once we have changed the value of `xPos` and/or `yPos` we update the `.x` and `.y` value of our `SDL_FRect` "box".

The `Draw()` function passes the render pointer to a bunch of SDL functions found in the `SDL_render.h`:

```cpp
void Core::Draw(SDL_Renderer* renderer){
    SDL_SetRenderDrawColor(renderer, 0, 70, 0, 255);
    SDL_RenderClear(renderer);
    SDL_SetRenderDrawColor(renderer, 150, 0, 30, 255);
    SDL_RenderFillRect(renderer, &box);
    SDL_RenderPresent(renderer);
}
```

- `SDL_SetRenderDrawColor` sets the color through RGB (and alpha for transparency of the renderer) — each of these are represented as a value between 0 and 255.
- `SDL_RenderClear` clears whatever was previously drawn to the screen and fills it with the color we set for the renderer previously.
- `SDL_RenderFillRect` passes our `SDL_FRect` as a pointer, it then takes the values of the struct and uses those to draw a rectangle on the screen. We've changed the RenderDrawColor so the rectangle shows up as a different color than the background.
- `SDL_RenderPresent` is what actually puts every pixel on the window. First all pixels are prepared, then all of them are drawn at once using this function.

Documentation: "SDL's rendering functions operate on a backbuffer; that is, calling a rendering function such as `SDL_RenderLine()` does not directly put a line on the screen, but rather updates the backbuffer. As such, you compose your entire scene and present the composed backbuffer to the screen as a complete picture. Therefore, when using SDL's rendering API, one does all drawing intended for the frame, and then calls this function once per frame to present the final drawing to the user."

So our basic draw loop is:

1. `RenderClear`
2. Draw everything
3. `RenderPresent`

There we go — we have created our first **core loop**, with input handling, updating game logic and finally drawing things to the screen!