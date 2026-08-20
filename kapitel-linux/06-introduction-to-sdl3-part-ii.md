# Introduction to SDL3 - Part II (Linux)

> **Linux:** This chapter is adapted for Linux.

We've installed clangd and it's available system-wide. Now if we make a typo in our generic C++ code we will get helpful warnings. But as we try and `#include` files from our include folder that we created as we started this project, then clangd will tell us that it can't find them.

We need to reopen our `CMakeLists.txt` and add some more build instructions. We are going to have clang generate a `compile_commands.json` file for us. Up until now we've hand-rolled our .JSON files, but the far more common way is using a JSON serializer to convert our data into a JSON format for us.

Adding this to our `CMakeLists.txt`:

```cmake
set(CMAKE_EXPORT_COMPILE_COMMANDS ON)
```

...will create a `compile_commands.json` in our build directory each time we compile our project using clang++.

> **Linux:** Add `CMAKE_EXPORT_COMPILE_COMMANDS` to your `CMakePresets.json` under `cacheVariables`.

Now we have 2 ways of ensuring that clangd can read this .json and use it to support our editor:

1. We copy the file from our build folder into our root each time
2. We write a small `.clangd` file that tells clangd where to find our `compile_commands.json`

We will create the `.clangd` file (option 2).

> **NOTE:** `clangd` is used as a file extension here, and the lack of any text before the `.` tells us that the file has no name.

In our root directory (`goto projectname`) we add the `.clangd` file using `touch .clangd`. We then open it inside your editor (e.g. nvim) with `your-editor .clangd`.

We will write our `.clangd` file using a different data format than JSON. The format is called **YAML** and has even less boilerplate than JSON, making it extremely human readable. But without braces to set the scope of a piece of data, YAML instead relies on **indentation** to sort data. An indentation is what pushes new lines of text to the right on your monitor.

> **NOTE:** Like when we put an if-statement inside another if-statement, then we began drifting to the right.

For now, our `.clangd` file will be very small:

```yaml
CompileFlags:
  CompilationDatabase: build/
```

Make sure that you adhere to the indentation, pushing `CompilationDatabase` one tab-step to the right. After saving our `.clangd` file we should head back to our `~/.bashrc` to add some new functionality to our build function.

We want to add the ability to completely empty our `build/` folder before compilation, just to ensure that everything we're doing is working as intended and not relying on a previous compilation step adding necessary things for us that are now no longer present.

By adding a new parameter and new logic to our function `build` we can pass it as a parameter when calling `build projectname clean`. The type of parameter inside bash is called an **argument** — when it is not passed the check is skipped and when present it triggers the clean.

> **NOTE:** This true/false data type is what we call a `bool` (boolean) in C++.

> **WARNING:** Before we look at the function below: we are going to start using pretty strong commands to add or delete files from our computer. It's a good idea to, in the future, have backups for important stuff if you start experimenting with these functions on your own. Our function below is not an issue as we are doing a lot to ensure that we are in the proper directory.

Here's our updated `build` function:

```bash
build() {
  local project="$1"
  local clean="$2"
  local preset="${3:-default}"

  goto "$project" || return 1
  local config
  config=$(GetConfig "$project")

  local sourceDir
  sourceDir=$(echo "$config" | jq -r '.path')
  if [[ ! -d "$sourceDir" ]]; then
    echo "root directory not found. Aborting..."
    return 1
  fi

  local buildDir="$sourceDir/build"
  if [[ ! -d "$buildDir" ]]; then
    echo "No build directory found. Aborting..."
    return 1
  fi

  if [[ "$clean" == "clean" ]]; then
    echo "Cleaning build directory..."
    rm -rf "$buildDir"
  fi

  cmake --preset "$preset"
  cmake --build --preset "$preset"
}
```

We've added a few new things. Like in other functions we've made we've sorted the `sourceDir` and `buildDir` in two variables. We have also added the `clean` parameter check — if the second argument is `"clean"`, we delete the build directory.

> **NOTE:** In both bash syntax and C++ we can skip the `== true` in an if-statement, as it is implied unless specified otherwise.

`echo` is a function that writes the parameter string to the console. This is a common method we will use in SDL3 later to check what is going on when our program is running.

`rm -rf`, like `touch`, is a built-in bash command. The additional parameters `-r` and `-f` make sure that:

- **A)** The files in subdirectories (folders) are also deleted (`-r` for recursive)
- **B)** Even hidden files and read-only files are removed without prompting (`-f` for force)

> **NOTE:** "Recurse" stands for **recursive** — a common programming method we will use in later course material.

Knowing more about if-statements and scope we can better understand that with this function we only do the `rm -rf` calls if `clean` was set to true.

We are using `[[ -d "$sourceDir" ]]` to ensure that a directory exists before we try and use it. And if `sourceDir` is not found we `return`, just like in SDL/C++. This way of adding checks and not just letting code run "willy-nilly" is standard practice.

Now with our build function updated and our `CMakeLists.txt` instructing us to generate a `compile_commands.json`, we can get much more helpful info from clangd — we just need to compile our program once so the `compile_commands.json` is populated with the info we need.

Now we can begin programming in our `main.cpp` and adding a few SDL3 specific lines of code to spawn a window and fill it with a nice background color.

Remember how we previously had to `SDL_Delay()` our program for 2000 milliseconds in order to even see that our program ran before reaching the `return 0` line at the end of our entry point? We will be using a new form of control flow statement to "run back over" the same piece of code over and over again, creating what is called a **program loop**.

For a game this loop is broken down into 3 distinct steps: **Input**, **Update** and **Draw**:

- **Input** handles key presses from a keyboard, controller or the input/motion of a mouse
- **Update** takes the information about what the game state is and updates it based on both player inputs and the current state
- **Draw** takes all objects that need to be rendered to the screen and draws them to the window

Then the loop starts over again. The more times this loop can be finished in a second, the higher our FPS is.

> **NOTE:** This loop is present in all games and every game engine is built on it — even though Unity hides the Draw part of the loop from us.

The control flow statement is known as a **while** and its syntax looks like this:

```cpp
while (true) {
    // Do stuff
    // then more stuff
}
```

Just like an if-statement it has the parenthesis where an expression is evaluated. As long as that expression is true, the code inside the while loop runs. At the end of the while loop's scope it jumps back to the beginning of the scope and runs it again.

A naive version would look like:

```cpp
while (game_is_running){
    // 1. Handle_Input
    // 2. Update_The_Game
    // 3. Draw_The_Game
    // ... and then repeat from step #1
}
```

Let's update our `main.cpp` with everything we will need to test our while loop:

```cpp
#include <SDL3/SDL.h>
int main() {
    bool running = true;
    while(running){
        SDL_Log("running...");
    }
    return 0;
}
```

> **Linux:** We use `#include <SDL3/SDL.h>` instead of separate includes like `"SDL3/SDL_log.h"` — `SDL.h` includes everything from SDL3.

Alright, so now our program doesn't quit automatically. Note how we've `#include` a new .h file. The reason the .h file is not added on its own but instead we've passed a path is because SDL3's system headers are in `/usr/include/SDL3/` so our `#include` points at a specific file by passing in its path.

> **NOTE:** The reason why our path is `<SDL3/SDL.h>` and not `<include/SDL3/SDL.h>` is because the compiler knows to look in system include paths. In our `CMakeLists.txt` we also set the include directory with:
> `target_include_directories(${PROJECT_NAME} PRIVATE include)`

> **NOTE:** Remember the not-so-elegant syntax of the standard library header `<iostream>` or `cout`? Well now we can use the much more convenient `SDL_Log()` instead!

We have no way of interrupting our program as there is nothing we can do within the scope of our while that will turn `running` from `true` to `false`.

> **NOTE:** A while loop that never terminates or pauses will hog 100% of our CPU and make the program appear frozen, as it tries to keep running the same code over and over. It's up to us to either terminate a while loop or in this case, add logic to it so it has to yield.

All code is eventually represented as binary machine code. All languages like C# and C++ are an intermediary step that lets us write instructions in a far(!) more readable format. The chain is actually that C++ gets compiled into assembly first, and then the assembly instructions are compiled into machine code. Our `while()` loop eventually becomes a `jmp` assembly instruction telling the program to jump to another line. Another way of coding a while loop is like this:

```cpp
int main(){
    Start:
    SDL_Log("Running...");
    goto Start;
}
```

The `goto` instruction tells the code to jump (jmp) to the location specified by us. In this case the line we've added `Start:` to.

Using the handy-dandy [godbolt.org](https://godbolt.org/) website we can actually compare our C++ code for both examples (`while()` and `goto`) and running both `main()` functions (without SDL stuff) we get the same output:

```
main:
.L2:
    push    rbp
    mov     rbp, rsp
    jmp     .L2
```

What this means is that when our program compiles, there is 0% difference between the while-loop and the goto solution. I touch on this to begin teaching you about what happens during the compilation step, introduce assembly and help empower you to dispel a lot of noise coming out of the programming community.

> **NOTE:** It is considered bad practice to use `goto` and recruiters will probably not like seeing it. But just to be clear, it's the exact same thing.

We will add some SDL boilerplate code to allow pressing Escape in order to flip `running` to `false` and terminate the while loop and the program. Though before we can do this, we need to learn about one of the most essential parts of C++: a **pointer**.

> **NOTE:** Pointers can be difficult to understand at first, re-reading is recommended.

Let's look at a basic `int`:

```cpp
int number = 5;
```

So far so good. Now let's take a function that accepts an `int` as a parameter and adds 1 to it:

```cpp
void AddOne (int theNumber) {
    theNumber += 1;
}
```

Then let's run this small program:

```cpp
int number = 5;
AddOne(number);
SDL_Log("%d", number);
```

Now the question is: is the number logged by `SDL_Log` a 5 or a 6? The answer is still **5** even though our `AddOne()` function seemingly increased it by 1. So what we need to understand is that we can pass a variable **by value** or **by pointer**. When we pass by value we just send the number, not the place in memory that stores that number. If we instead pass the variable **by pointer** we pass the location in memory that holds the variable, updating that will persist as we exit the scope of the function.

```cpp
int number = 5;
AddOne(&number);
SDL_Log("%d", number);

void AddOne(int* theNumber) {
    *theNumber += 1;
}
```

The `&` before the variable name will pass the variable **by pointer** instead of **by value**. The `int*` with the `*` after the variable type indicates that we are working with a pointer (pointing to a place in memory) rather than a number. At this point we can't actually add 1 to the variable as it is not itself a number, but rather something that points to a location in memory where a number lives. The `*` before the variable name within the scope will **dereference** the pointer, allowing us to modify the value stored in memory at that location.

Let's break it down:

- `int number = 5;` — this is a newly created variable. It is stored in memory somewhere with the value 5.
- `int* aNumber` — this is a pointer variable. It points not to a number, but the place in memory where we will store a number. To pass a point in memory to a function that expects a pointer we must pass the variable by pointer using `&`.

> **NOTE:** Without passing our `number` variable as a pointer we are actually just performing operations on a new number that lives only within the scope of the `AddOne()` function. As soon as we leave that scope the number stops existing.

- `&number` — we take `number`, and instead of passing it by value we get its place in memory and pass that along instead.
- `*number += 1;` — this takes the reference to the memory address that the pointer was pointing to (where `number` lives) and dereferences it, grabbing the value stored in it so we can manipulate and change it.

There is another way of passing by reference that adds less new symbols, keeps things tidier and makes the compiler handle more of the heavy lifting for us:

```cpp
int number = 5;
AddOne(number);
SDL_Log("%d", number);

void AddOne(int& theNumber){
    theNumber += 1;
}
```

Here we make it so the function is expecting an `int&`. This handles the pass-by-reference and dereferencing for us during compilation. Just by adding this single `&` the program knows that any value being passed to the function is really passed by reference, and any changes to the passed variable in the function will be made on the dereferenced value stored in that reference we passed along. It's more invisible and has both its advantages and disadvantages — it's less explicit because we hide the fact that `AddOne()` works by reference until we go to the function itself and spot the `int&`.

The three ways we can pass something in C++ to a function:

```cpp
// 1. Pass by value - original unchanged
void AddOne(int theNumber) {
    theNumber += 1;
}
AddOne(number);
Result: number = 5

// 2. Pass by pointer - explicit, you can see the & at call site
void AddOne(int* theNumber) {
    *theNumber += 1;
}
AddOne(&number);
Result: number = 6

// 3. Pass by reference - cleaner, compiler handles it
void AddOne(int& theNumber) {
    theNumber += 1;
}
AddOne(number);
Result: number = 6
```

The "simplified" way of passing by reference requires less specific C++ syntax. And that convenience styled syntax is often referred to as **syntactic sugar**. Understanding how we pass something as either the memory address or the actual value is key to working with C++. So we will mostly be working with #2 as a learning tool. Though much of the C++ you will come across uses #3. So eventually you will still have to learn and understand both.

We can also convert our `AddOne()` function from a `void` to an `int` function to return whatever number we've created inside its scope, allowing us to assign the new value to our `number` variable:

```cpp
int main(){
    int number = 5;
    number = AddOne(number);
    SDL_Log("%d", number);
}
int AddOne(int theNumber){
    theNumber += 1;
    return theNumber;
}
```

With this setup the Log function will output a 6, as the updated value of 5 → 6 lives inside the scope of the `AddOne` function but the value is later returned and then using the `=` operator assigned back into the `number` variable inside `main()`.

> **NOTE:** A `=` operator always assigns the value on its right to whatever is on its left.

Now that we know more about pointers and while loops we can better understand the SDL syntax (boilerplate) necessary to start working on a more proper example. What follows is a lot of new code, but we've touched on this syntax in many cases. We will break it all down once we've seen it in its entirety.

```cpp
#include <SDL3/SDL.h>
SDL_Window* window;
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
    SDL_Init(SDL_INIT_VIDEO);
    bool running = true;
    window = SDL_CreateWindow("pilot", 650, 400, 0);
    while(running){
        SDL_Event event;
        while(SDL_PollEvent(&event)){
            running = HandleRunning(event);
        }
    }

    return 0;
}
```

> **Linux:** `SDL_Init(SDL_INIT_VIDEO)` initializes the video subsystem, which is needed for creating a window. `SDL_INIT_EVENTS` is implicit when using video.

This program does a few new things:

1. It actually creates a game window!
2. It Initializes SDL video so we can get keyboard inputs
3. It stops running if we press the Escape key
4. It nests a while loop within another while loop
5. It passes multiple parameters to a single function
6. It includes SDL3 header files

Let's begin by looking at our entry point.

The `main()` function has a lot of new parts to it. A lot of it is boilerplate that SDL3 requires in order to start communicating with our computer. `SDL_Init()` accepts a series of so-called **flags** as parameters that tell it what systems from SDL3 to activate. In our case we've told it to initialize the "VIDEO" subsystem. This is required in order to create a window and get keyboard inputs to register as `SDL_Event`s we can query.

> **NOTE:** "Query" means "ask questions about".

A flag is actually a datatype known as an **enum**. It is a series of numbers that are all represented as a name. What makes it a flag as well as an enum is that the numbers associated with each enum are one power of 2 larger than the previous.

```cpp
enum WeaponBuffs {
    NONE = 0,
    POISON = 1,
    FIRE = 2,
    ICE = 4,
    DARK = 8
};
```

An enum that is not used as a flag does not need to specify its numbers, they are then treated as just growing by 1:

```cpp
enum WeaponType {
    NONE,
    SWORD,
    AXE,
    HAMMER,
    BOW
};
```

When working with flags, by having each enum element as its own power of 2 we can combine them together and the number that is the result of that addition can only be the result of that exact combination. In our "WeaponBuffs" example, if we wanted to have a weapon with both a POISON and an ICE buff then this would be stored (in the background) as a `5`, and no other combination of flags can produce that number.

Enums are a great way of storing similar attributes in a way that is very easy to work with and very easy to read.

SDL3's `SDL_Init()` function accepts multiple flags if we want. To send in multiple flags at once as a parameter to the function we use something called a **bitwise operator**, specifically `|` — it is called a **bitwise OR**. The OR operator combines bits if at least one of them is a 1. Let's take a little detour and learn about bits.

When we are down in machine code, everything is represented as either a 0 or a 1. Our computer calls these **bits** — each bit can either be **on** or **off** (AKA 1 or 0). The `bool` variables we've created earlier can also either be `true` or `false` so in that sense they are similar, though a `bool` is stored in memory as a **byte** which is the same thing as **8 bits**. Our computer works on bytes rather than bits so even though all the necessary info about a bool just requires 1 bit we are still required to store all 8.

In the case of our WeaponBuffs we can take each enum and because we used a sequence of powers of 2 we get a very pleasing pattern of bits:

```
NONE   00000000
POISON 00000001
FIRE   00000010
ICE    00000100
DARK   00001000
```

Look how each bit that is turned on occupies its own column. If we use a "Bitwise OR" operator on these (the `|`) we combine them, keeping a 1 if ANY of the bits in the byte was a 1:

```
POISON  00000001
ICE     00000100
=       00000101
```

And now we can clearly see why an enum that is used as a flag needs to have each entry as a power of 2. And the Result above `00000101` is actually the byte representation of the number **5**.

If we want to pass both `SDL_INIT_EVENTS` and `SDL_INIT_VIDEO` to our `SDL_Init()` function then we add a bitwise OR between the flags:

```cpp
SDL_Init( SDL_INIT_EVENTS | SDL_INIT_VIDEO );
```

> **NOTE:** You can find all the SDL_INIT flags here: [SDL_INIT flags](https://wiki.libsdl.org/SDL3/SDL_Init)

Let's keep looking at our program:

```cpp
#include <SDL3/SDL.h>
SDL_Window* window;
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
    SDL_Init(SDL_INIT_VIDEO);
    bool running = true;
    window = SDL_CreateWindow("pilot", 650, 400, 0);
    while(running){
        SDL_Event event;
        while(SDL_PollEvent(&event)){
            running = HandleRunning(event);
        }
    }

    return 0;
}
```

`#include <SDL3/SDL.h>` at the top adds the entire SDL3 API to our program.

Just below our `#include` we store a pointer in memory, the type being an `SDL_Window`. We know we're storing a pointer and not the actual data as the `SDL_Window` is followed by a `*`. This line on its own has not created a window for us. We need to actually create that window using the `SDL_CreateWindow()` function.

This function accepts 4 parameters. The first being the name of the window, then its width and height, followed by whatever option flags we want to pass along. Let's look at those now:

[SDL_CreateWindow documentation](https://wiki.libsdl.org/SDL3/SDL_CreateWindow)

We should become comfortable with reading documentation, as this is really the only way of knowing how these kind of systems work. As we can see, the `SDL_WindowFlags` enum can accept either a `0` (for NONE) or one or more flags combined with a bitwise OR (`|`). We can also read that the function returns a `SDL_Window*` — this is then stored in the `SDL_Window* window` pointer variable we created at the top of our program.

We have another function besides our `main()`. It has the return type of `bool` and is tasked with checking each event that SDL creates and when it finds that we've pressed the Escape key it returns `false` instead of `true`. This flips `running` to `false` and the infinite repeating while loop is terminated and the program quits by returning 0.

The nested while loop inside `main()` first stores a variable of the type `SDL_Event`, then it passes that place in memory to the `SDL_PollEvent()` function, and by passing it by reference the `SDL_PollEvent` can make changes to the variable that is stored in that same variable that we passed into the function. So that when we call our `HandleRunning()` function we pass along that same `event` variable, now potentially modified by our `PollEvent()`.

> **NOTE:** The documentation for PollEvent can be found here: [SDL_PollEvent](https://wiki.libsdl.org/SDL3/SDL_PollEvent)

Looking at our `HandleRunning()` function we can see a series of `if` and `if-else` statements asking questions about (querying) the `SDL_Event` parameter that was passed into it. As we are only interested in if the Escape key was pressed we can avoid nesting by returning `true` (meaning **keep running**) if the event was not a keyboard event to begin with. Then if we did not return we know for a fact that it is a keyboard event we're querying. Then we check the pretty nasty looking `key.key` and compares the key value with the SDL enum called `SDLK_ESCAPE` and if it is a match we return `false`.

`SDLK_` is the prefix for all keyboard events. The full list of buttons can be found here: [SDL_Keycode](https://wiki.libsdl.org/SDL3/SDL_Keycode)

The value returned from the function is then stored into our `running` variable and if it was `false` it will stop the while-loop from continuing to run. Now we can actually quit our game by pressing Escape.

> **NOTE:** In the future we would of course not just close the program anytime someone accidentally presses Escape — but for now we'll use this brutish approach.

So now we do the following things inside our program:

1. We initialize SDL
2. We create a window
3. We run our game's core loop inside `main()`
4. We allow the core loop to terminate using Escape

With this we've laid the foundation for a core loop (**Input, Update, Draw, repeat**)!