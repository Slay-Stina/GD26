# 19 Animation Part I

This chapter covers code related to animating our entities, as well as how to buffer inputs for a smoother gameplay experience.
Before we do that we will do a small piece of housekeeping. We're moving our `memcpy()` function from the bottom of `Update()` in `Game.cpp` to `main.cpp` . We'll call `memcpy()` on the line after we call `dll->Update()` . The reasoning being that this is part of the foundation of our game engine and should never be accidentally removed or skipped due to us making big changes to `game.cpp`

```cpp
// main.cpp
while(running) {
  // other code hidden for clarity
  dll.update(gameData, dt);
  memcpy((void*)gameData->keys_previous, SDL_GetKeyboardState(nullptr), SDL_SCANCODE_COUNT * sizeof(bool));
}
```

This was the same function we called inside `game.cpp` but we pass along `SDL_GetKeyboardState` directly instead of having it saved to the earlier variable we named `keys`
With that out of the way, what we want to do is having our entities slide across the screen instead of teleport to their new location.
To do this we'll need to store two sets of variables

1. where they are currently
2. where they previously were

with this we can linearly interpolate between them. This is a way of making a third value that slides between the two extremes. Linear interpolation is almost always refered to as `lerp` and has 3 basic components, `a`, `b`, and `t` .
here's some pseudo-code

```cpp
float milesTravelled = 0;
milesTravelled = lerp(0, 1000, 0.4);
```

in this example, `milesTravelled` can have any value between 0 and 1000. the variable `t` set to `0.4` makes the `lerp()` function return the 40% point between `a` and `b` . In this case `400`.
First we'll add four new variables to `struct Entity`

```cpp
// entity.h
struct Entity {
  ID id;
  int x;
  int y;
  int x_prev;
  int y_prev;
  float progress_01;
  Behaviour behaviour;
};
```

`x_prev` and `y_prev` will store the previous position of our entity. `progress_01` will go between 0 and 1 and act as the value we assign to `t` in our `Lerp()` later.
With this we can go to our `RenderEntities()` function in `LevelRenderer.cpp`

```cpp
// levelRenderer.cpp
#include <cmath>
```

we need this header to be included to get access to a `Lerp()` function.

```cpp
// levelRenderer.cpp
// inside RenderEntities()
int xPos = 0;
int yPos = 0;
xPos += SCREEN_WIDTH / 2.0;
yPos += SCREEN_HEIGHT / 2.0;
xPos -= data->levels[data->currentLevelIndex].w * CELL_SIZE_PX / 2;
yPos -= data->levels[data->currentLevelIndex].h * CELL_SIZE_PX / 2;
float x_animated = std::lerp(entity.x_prev, entity.x, entity.progress_01);
float y_animated = std::lerp(entity.y_prev, entity.y, entity.progress_01);
xPos += x_animated * CELL_SIZE_PX;
yPos += y_animated * CELL_SIZE_PX;
RenderSprite(img, renderer, xPos, yPos);
```

previously we used `entity.x` and `entity.y` directly when calculating `xPos` and `yPos` . We now `Lerp()` between `x/y_prev` and `x/y` using `progress_01` and store the moving position in `x/y_animated` . Then we use that to adjust `x/yPos` .
Now we need to make sure that `Progress_01` increases whenever we issue a move command.
Before we make some large scale changes to `Update()` in `game.cpp` there is some more logic we need to set up.
We're going to be constructing a new way of storing our data. We'll be using a ring buffer to hold all our arrow key inputs. But we might be pressing the arrow keys thousands of times per level and we're only really interested in the 2-5 next inputs that are yet to be animated. Once these have been animated we are free to discard this info.

A ring buffer is a limited sized array that loops back on itself once its full - therefore overwriting its oldest elements.
We'll add this input ring buffer to our `GameData` in `gameState.h`

```cpp
// gameState.h
struct GameData {
  // other variables hidden for clarity
  Position* input_buffer;
  int input_buffer_capacity;
  int input_buffer_write_count;
  int input_buffer_read_count;
};
```

You'll notice that `Position` is a new variable. We'll take a quick detour to `entity.h` and add this very (very) simple struct first.

```cpp
// entity.h
struct Position {
  int x;
  int y;
};
```

As a position is always both an x and a y value we've collapsed them into a single struct to make reasoning about them simpler.
our four new variables in `GameData` are:
`Position* input_buffer` : this is an array like we're used to. `input_buffer_capacity` : the size of this array. We'll be keeping this very small on purpose `input_buffer_write_count` : this is a running talley of how many inputs have ever been added to the ring buffer so if the buffer can hold 5 input elements and we've added 500. We can still only access the latest 5, but this integer will let us know how many we've ever added. `input_buffer_read_count` : each input is read only once its time for that input to be processed. Meaning that if we press up twice, we'll immediatly begin moving up. But only after we've arrived at our destination will we move up for the second time. `read_count` will lag behind `write_count` and with each move performed by our entities this will increase by 1 until `read` and `write` are at the same value.
To leverage our ring buffer we'll be using our `_capacity` variable alongside our `_read` and `_write` to know which element out of the looping few in the ring buffer to use. To do this we'll use the modulo operator - `%` .
The modulo operator takes the first value and loops it over the second value as many times as it can. And once it can't loop the value any longer it returns whatever was left.

If we have 5 spots in a ring buffer and we are adding 7 things to it. We'll modulo 7 into 5
`7 % 5 = 2`
If we wanted to add 24 we'll modulo 24 into 5
`24 % 5 = 4`
this strips away 5 from 24 for as long as is possible before returning the remaining value. So 5 is stripped away four times for a total of 20. before we return 4.
With this we can reason about `_capacity` and `_read/_write`

```cpp
front = input_buffer_read_count % input_buffer_capacity;
```

the value of `front` would be the total value ever added to `_read_count` that has been modulo'd with `_capacity` .
Lets look at an example

```cpp
front = 1536 % 20;
```

in this scenario our made up variable `front` gets a value of `16` . As we can strip away 20 from 1536 a total of 76 times `20 * 76 = 1520` then we have `16` remaining which is the value stored in `front` .
Inside `main.cpp` we'll allocate the ring buffer to our `arena_levels`

```cpp
// main.cpp
gameData->input_buffer_capacity = 50;
size_t RING_BUFFER_SIZE = sizeof(Position) * gameData->input_buffer_capacity;
gameData->input_buffer = (Position*)Memory::Allocate(gameData->arena_levels, RING_BUFFER_SIZE);
```

This allows our ring buffer to hold 50 inputs before looping. Should we ever find that we need more, we can just increase this number. The memory footprint of our `Position` struct is extremely(!) small.
We'll be making large changes to our `Update()` function inside `game.cpp` . The only code we'll save for now is:

```cpp
// game.cpp
void Update(GameData* data, float dt) {
  const bool* keys = SDL_GetKeyboardState(nullptr);
  if(KeyPressed(SDL_SCANCODE_Z, keys, data->keys_previous)) {
    if(KeyHeld(SDL_SCANCODE_LSHIFT, keys, data->keys_previous)) {
      Redo(data->commandBuffer);
    }
    else {
      Undo(data->commandBuffer);
    }
  }
}
```

everything else we can safely delete. This way of programming where we first ensure we get something working, and only once we have a concrete need for a new feature do we actually code that system is by far the most reasonable way of working and the act of rewriting code aka refactoring is a cornerstone of programming. So with the rest of `Update()` removed we can no longer move our entities in game.
What we'll do next is start filling our ring buffer

```cpp
// game.cpp
// Update() function
if(KeyPressed(SDL_SCANCODE_RIGHT, keys, data->keys_previous)) {
  data->input_buffer[data->input_buffer_write_count++ % data->input_buffer_capacity] = {1, 0};
}
else if(KeyPressed(SDL_SCANCODE_LEFT, keys, data->keys_previous)) {
  data->input_buffer[data->input_buffer_write_count++ % data->input_buffer_capacity] = {-1, 0};
}
else if(KeyPressed(SDL_SCANCODE_UP, keys, data->keys_previous)) {
  data->input_buffer[data->input_buffer_write_count++ % data->input_buffer_capacity] = {0, -1};
}
else if(KeyPressed(SDL_SCANCODE_DOWN, keys, data->keys_previous)) {
  data->input_buffer[data->input_buffer_write_count++ % data->input_buffer_capacity] = {0, 1};
}
```

the `KeyPressed` calls are very similar to our older code, but now we do some fancier stuff with it. We utilize something called syntactic sugar to make the creation of our `Position` struct smaller.
there is a lot going on in this code but you'll notice that it's actually repeating four times with just a change to the if-statement and the `= {int, int}` at the end.
Lets start with the small `= {1, 0}` .

This is the syntactic sugar I mentioned earlier.

`Position(1, 0)` can be simplified down to just `{1, 0}` . If you find it confusing to not write the type then you can easily substitute out the sugar'ed version.
We assign the relevant `Position` to the correct element in our Ring Buffer by taking the `_write_count` and adding a Modulo with `_capacity` . The increment operator aka `++` will increase `_write_count` by 1 after the line of code has resolved. This is the same as writing:

```cpp
data->input_buffer[data->input_buffer_write_count % data->input_buffer_capacity] = {1, 0};
data->input_buffer_write_count += 1;
```

If you find the single line to be "doing to much" you can easily remove the increment operator and add a `+= 1` on a line below.
With each arrow key adding its own `Position` to the buffer we can begin looking at taking these inputs and one-by-one resolving them - to actually make our entities move by adding the following code after we've checked what keys are pressed

```cpp
// game.cpp
// update() function
bool are_entities_moving = false;
for (int i = 0; i < data->GetCurrentLevel()->entityCount; i++) {
  Entity* entity = &data->GetCurrentLevel()->entityBuffer[i];
  if(entity->HasBehaviour(CAN_MOVE) && IsMoving(entity)) {
    entity->progress_01 += MOVE_SPEED * dt;
    if(entity->progress_01 >= 1) {
      entity->progress_01 = 0;
      entity->x_prev = entity->x;
      entity->y_prev = entity->y;
    }
    if(IsMoving(entity)) {
      are_entities_moving = true;
    }
  }
}
```

First, by creating `are_entities_moving` and setting it to false at the start, we can know that if we get past the upcoming for-loop and it is still `false` , then we found no entity that was moving. And if no entity was moving, we can check if we have any more buffered inputs to resolve.
We loop over all entities in the current level and then we check if they are allowed to move `CAN_MOVE` and has a move its currently performing `IsMoving()` .
`IsMoving()` is a new function we've added to `entity.h` . But we've added it outside of our `Entity` struct . Instead it is added to a newly created `Entity.cpp` . Later we'll move more of the functions found inside our `Entity` struct to our `entity.cpp` instead. The logic will be very similar, but it will be more in line with our overall code structure.
lets look at our newly created `entity.cpp`

```cpp
// entity.cpp
#include "entity.h"
bool IsMoving(Entity* e) {
  return e->x != e->x_prev || e->y != e->y_prev;
}
```

This means that `IsMoving()` returns true of either the x or y values were different from `x/y_prev` .

This is only the case when a move is under way. Once we have arrived at our location `x/y_prev` will catch up to `x/y` and have the same value.
back in our `Update()` we can now see that our if-statement asks that the entity both is moving and is allowed to move. If this is the case we update its `progress_01` by adding `MOVE_SPEED` multiplied by delta time .
`MOVE_SPEED` is a new variable added to `common.h`

```cpp
// common.h
const float MOVE_SPEED = 6.0;
```

This means that anytime we multiply delta time aka `dt` by this value, we make it go from 0 to 1 in `1/6` of a second. and because `dt` aligns with our framerate we can ensure that it takes `1/6` of a second no matter how powerful the computer running the game is.
if `progress_01` is at or above a `1` we catch up `x/y_prev` to `x/y` and reset `progress_01` so that it can begin a new move sequence later.
Then we check if `IsMoving()` is still true after having added to `progress_01` and if so we flip the `are_entities_moving` boolean to `true` . Note that nothing inside this for-loop can set it to `false` .
The next step of our `Update()` is to call `TryMove()` again using our Ring Buffer

```cpp
if(are_entities_moving == false) {
  if(data->input_buffer_read_count == data->input_buffer_write_count) {
    return;
  }
  data->command_timestamp += 1;
}

for (int i = 0; i < data->GetCurrentLevel()->entityCount; i++) {
  Entity* entity = &data->GetCurrentLevel()->entityBuffer[i];
  if(entity->HasBehaviour((Behaviour)(RESPOND_TO_INPUT | CAN_MOVE))) {
    int xDir = data->input_buffer[data->input_buffer_read_count % data->input_buffer_capacity].x;
    int yDir = data->input_buffer[data->input_buffer_read_count % data->input_buffer_capacity].y;
    TryMove(entity, data->GetCurrentLevel(), data->commandBuffer, xDir, yDir, data->command_timestamp);
  }
}
data->input_buffer_read_count++;
```

First we check if `are_entities_moving` was `false` . Meaning that all entities have arrived at their location.
Then we compare our `_read` with our `_write` . If `_read` has caught up then we have no further inputs to resolve and we can return early. This means that if we hit our return we leave our `Update()` function and no code below it will run.

If we manage to get past the read / write if statement we know that we have at least one input to process. We can therefore increase `command_timestamp` by 1 - giving all Commands created during this tick a new timestamp. Note that with this system our `command_timestamp` only increases in this case, meaning that it won't increase each tick on its own as it did in our previous code.
We then for-loop over all our entities again this time checking for `RESPOND_TO_INPUT` as well as `CAN_MOVE` . if we find an entity with both these behaviours we collect the x/y from the `Position` currently being pointed to in our Ring Buffer . We do this by taking `_read` and using modulo on it with our `_capacity` . This gives us the `Position` struct that we can then fetch x/y from.
We then call `TryMove()` as normal.
once we have looped over all our entities we increase `_read_count` by using the increment operator `++` . Meaning that the next time we check `are_entities_moving` we will be evaluating the next element in the ring buffer .
Finally we have to update `command.cpp` to assign our `x_prev` and `y_prev` values. As well as adding what is known as an optional parameter

```cpp
// command.cpp
void Execute(AnyCommand cmd, bool from_redo = false) {
  switch(cmd.command.type) {
    case CMD_TYPE::NONE:
      break;
    case CMD_TYPE::MOVE:
      MoveCommand mv = cmd.move;
      mv.entity->x_prev = mv.entity->x;
      mv.entity->y_prev = mv.entity->y;
      mv.entity->x += mv.xDir;
      mv.entity->y += mv.yDir;
      if(from_redo) {
        mv.entity->progress_01 = 1;
      }
      break;
  }
}
```

we're adding an optional parameter to `Execute()` meaning that we can skip passing this bool when we call the function if we want. In that case the value will default to the value set in the Function itself. In this case `false` .
Before we update x/y we store the old values in `x/y_prev` .
Then we check if the `from_redo` optional parameter was `true` and if it was we set `progress_01` to `1` immediately. Meaning that it will instantly arrive at its new destination. We will be setting this optional parameter to `true` when calling `Execute()` from our `Redo()` function.
We'll do something similar in `Undo()`

```cpp
// command.cpp
void Undo(CommandBuffer* buffer) {
  // code above hidden for clarity
  switch(cmd.command.type) {
    case CMD_TYPE::NONE:
      break;
    case CMD_TYPE::MOVE:
      MoveCommand mv = cmd.move;
      mv.entity->x -= mv.xDir;
      mv.entity->y -= mv.yDir;
      mv.entity->progress_01 = 1;
      break;
  }
  // code below hidden for clarity
}
```

we also put `progress_01` to `1` if we call `Undo()` .
Finally our change to `Redo()` looks like

```cpp
// command.cpp
void Redo(CommandBuffer *buffer) {
  // code above hidden for clarity
  Execute(cmd, true);
  // code below hidden for clarity
}
```

we have only added `true` as an optional parameter when calling `Execute` . if we look at our `Push()` function we can see that because we added this as an optional parameter we didnt have to make any changes to the call to `Execute()` inside it.

```cpp
// command.cpp
void Push(CommandBuffer* buffer, AnyCommand cmd, uint32_t timestamp) {
  buffer->allCommands[buffer->index] = cmd;
  buffer->allCommands[buffer->index].command.timestamp = timestamp;
  buffer->index++;
  buffer->head = buffer->index;
  Execute(cmd);
}
```

With these final changes our entities now slide across the game board and our inputs can be buffered, meaning that we can press our arrow keys as fast as we want and inputs will be registered and acknowledged once the animations have caught up to them!