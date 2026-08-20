# 23 Mouse input

Before we can write our level editor we're going to need a way of accessing the state of our mouse. We can do this very naively by not storing the mouse state between frames, but then any code that would check if a mouse button was pressed would fire every single tick.
We're going to return to our `input.h/cpp` and add the relevant variables to track our `mouseState` and then a few functions to simplify checking the current mouse state. In a lot of ways this will be similar to how we work with keys, but a bit less "cool".

```cpp
// input.h
struct Input {
  const bool* keys_current;
  const bool* keys_previous;
  float* keys_held_time;
  SDL_MouseButtonFlags mouse_current;
  SDL_MouseButtonFlags mouse_previous;
  float* mouse_held_time;
  float mouse_x;
  float mouse_y;
};
```

We will track and compare `_current` and `_previous` just as with our keyboard keys. We're also storing the position of the mouse as `mouse_x/y` . These two floats will be passed in to an SDL function that updates their values for us. the `SDL_MouseButtonFlags` are bitwise flags that we will be able to use bitwise operators to check against.

```cpp
// input.h
enum class MouseButtons {
  LEFT = 0,
  MIDDLE = 1,
  RIGHT = 2,
};
```

We're going to be using SDL's bitmasks to compare our mouse buttons, but I've opted for this more human readable version as an in-between to make calling our new functions a bit simpler. You can decide for yourself if you find this extra step harder or easier to parse.

```cpp
// input.h
bool MousePressed(const Input* input, MouseButtons button);
bool MouseReleased(const Input* input, MouseButtons button);
bool MouseHeld(const Input* input, MouseButtons button);
bool MouseHeld_ForTime(const Input* input, MouseButtons button, float min_length);
void UpdateMouse(Input* input, float dt);
```

very similarly to our `KeyPressed()` functions we're creating one for each of the relevant checks `Pressed, Held` and `Released` as well as a way of checking for how long our mouse buttons have been held down.
In our `input.cpp` we'll be adding a helper function to handle the `MouseButton` to `SDL_flag` convertion. We're keeping this in the .cpp so no file that includes our .h file can access it.

```cpp
// input.cpp
SDL_MouseButtonFlags ButtonToFlag(MouseButtons button) {
  switch(button) {
    case MouseButtons::LEFT:
      return SDL_BUTTON_LMASK;
    case MouseButtons::MIDDLE:
      return SDL_BUTTON_MMASK;
    case MouseButtons::RIGHT:
      return SDL_BUTTON_RMASK;
      break;
  }
}
```

We pass in a button and return the corresponding `SDL_BUTTON_L/M/RMASK` . You can check out the definition of each Mask by putting the caret over them and pressing `space+d` . The bitwise logic is a bit more forced in my opinion. I'll be happy to have this tv-remote style interface to simplify accessing them.

```cpp
// input.cpp
bool MousePressed(const Input* input, MouseButtons button) {
  SDL_MouseButtonFlags flag = ButtonToFlag(button);
  return (input->mouse_current & flag) != 0 && (input->mouse_previous & flag) == 0;
}

bool MouseReleased(const Input* input, MouseButtons button) {
  SDL_MouseButtonFlags flag = ButtonToFlag(button);
  return (input->mouse_current & flag) == 0 && (input->mouse_previous & flag) != 0;
}

bool MouseHeld(const Input* input, MouseButtons button) {
  SDL_MouseButtonFlags flag = ButtonToFlag(button);
  return (input->mouse_current & flag) != 0 && (input->mouse_previous & flag) != 0;
}

bool MouseHeld_ForTime(const Input* input, MouseButtons button, float min_length) {
  SDL_MouseButtonFlags flag = ButtonToFlag(button);
  return input->mouse_held_time[flag] >= min_length;
}
```

We fetch the `SDL_MouseButtonFlag` from our helper function (that we have to have above these functions as it only exists in our .cpp and is compiled top-to-bottom). Then we do a slightly different kind of boolean comparison here. We are checking if the bits of the Mask of our `mouse_current/previous` and `flag` overlap. If the result was not 0 aka `!= 0` then at least one bit remained. if the result was `0 == 0` then the two flags shared no bits. Because the `L/M/RMASKS` only set one bit to 1 each, then we can treat the `(mouse & flag) == 0` as false and `(mouse & flag) != 0` as true. With this the logic is identical to our keys.

```cpp
// input.cpp
void UpdateMouse(Input* input, float dt) {
  if(MouseHeld(input, MouseButtons::LEFT)) {
    input->mouse_held_time[(int)MouseButtons::LEFT] += dt;
  }
  else {
    input->mouse_held_time[(int)MouseButtons::LEFT] = 0;
  }
  if(MouseHeld(input, MouseButtons::MIDDLE)) {
    input->mouse_held_time[(int)MouseButtons::MIDDLE] += dt;
  }
  else {
    input->mouse_held_time[(int)MouseButtons::MIDDLE] = 0;
  }
  if(MouseHeld(input, MouseButtons::RIGHT)) {
    input->mouse_held_time[(int)MouseButtons::RIGHT] += dt;
  }
  else {
    input->mouse_held_time[(int)MouseButtons::RIGHT] = 0;
  }
  input->mouse_previous = input->mouse_current;
}
```

Ok, this is not the prettiest function, but trying to make aesthetically pleasing code is not something to strive for in and of itself. I don't envision this code changing for the forseable future and even though it repeats itself three times it's easy to parse. We check if we're holding down a mouse button, if we are then we increment the `mouse_held_time` element at that position in the array using the number associated with the `MouseButton` enum. If the mouse button was not held we reset the value back to 0.
Lastly we take the contents of `mouse_current` and set `mouse_previous` to match.
Inside `main.cpp` we need to do some allocation and then call the relevant functions.

```cpp
// main.cpp
gameData->arena_input = Memory::CreateSubArena(arena_main, INPUT_ARENA_SIZE);
gameData->input.keys_current // old (shortened for clarity)
gameData->input.keys_previous // old (shortened for clarity)
gameData->input.keys_held_time // old (shortened for clarity)
gameData->input.mouse_held_time = (float*)Memory::Allocate(gameData->arena_input, sizeof(float) * 3);
```

We have hard-coded the value 3 as that is how many mouse buttons we're working with. This could warrant either a small comment or an actual variable. So lets add one to show our options

```cpp
// main.cpp documentation example
// either we write a comment telling us that `3` represent the three mouse buttons (Left, Middle, Right)
gameData->input.mouse_held_time = (float*)Memory::Allocate(gameData->arena_input, sizeof(float) * 3);
// or we add a reminder variable
int mouseButtonCount = 3;
gameData->input.mouse_held_time = (float*)Memory::Allocate(gameData->arena_input, sizeof(float) * mouseButtonCount);
```

now inside our main loop we can add the logic to fetch and update our mouse data

```cpp
// main.cpp
gameData->input.keys_current = SDL_GetKeyboardState(nullptr);
gameData->input.mouse_current = SDL_GetMouseState(&gameData->input.mouse_x, &gameData->input.mouse_y);
dll.update(gameData, dt);
UpdateKeys(&gameData->input, dt);
UpdateMouse(&gameData->input, dt);
dll.draw(gameData, renderer);
```

`SDL_GetMouseState` both returns the `SDL_MouseButtonFlags` for the current tick and accepts a pointer to a x float and a y float. These values will be set inside the function itself to reference the current position of the mouse. By passing the `mouse_x/y` we can ensure that these variables are accessible from inside our `Update` and `Draw` loops by fetching them from `gameData->input.mouse_x/y` .
Now our game engine can handle basic mouse inputs. In the next chapter we'll be using this to help us create a level editor