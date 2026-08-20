# 25 Sokoban programming IV

We're going to be adding functionality specific for the certain game we're making. To outline it we're creating a cast of characters that have different gameplay abilities. The starting point will be the game Heroes of Sokoban 1, 2 and 3 by Jonah Ostroff (https://sites.math.washington.edu/~ostroff/puzzles/Heroes_of_Sokoban.html)
The heroes of sokoban are:
Red (warrior) pushes blocks Green (thief) drags blocks Blue (wizard) swaps position with blocks in view yellow (priestess) the priestess is unkillable purple (bard) pushesx and drags along entities in the squares around her green (druid) turns blocks into foilage and vice versa
on a level one or more of these characters will be present, and the player will be allowed to swap between them using an action button. Then each level is cleared when all characters present on the level are standing on a designated goal square. Each of these abilities are compulsory meaning that they are not activated by the player and is instead an intrinsic part of the character - for good and for bad.
We'll be adding another cast of characters with their own abilities instead

Golem
> Can push blocks
> Can push any amount of blocks
> Can't be pushed

Medusa
> Can push 1 block
> Turns objects she looks at into pushable rocks
> then they transform back if she no longer looks at them

Siren
> objects on the same row or column attempt to move in her same direction
> can't push objects herself

Demon
> Can walk on lava
> Can push 1 block

So for the Golem we need the logic to control how many blocks a character is allowed to push.
For Medusa we need to keep track of the facing direction of entities, and update these as they turn to move. We also need a Petrified state to help us transform objects
For the siren we need to complicate our TryMove to then issue new moves on all objects found. We're also adding a Charmed state to track this.
For the Demon we need specific allowances to make lava situationally walkable.
First, before we start on all this fun stuff (sorry) I want to refactor some of the code in `entity.h/cpp` .
We're currently working with the C-standard paradigm with the goal of keeping structs as plain data and pass them along to functions to modify them.
So currently inside of `struct Entity` in `entity.h` we have 5 functions related to working with `Behaviour` . We're going to move them out of the struct and add a first parameter to each of them where we pass along an `Enitity*` pointer.

```cpp
// entity.h
bool HasBehaviour(Entity* entity, Behaviour flags);
void InitializeBaseBehaviour(Entity* entity);
void SetBehaviour(Entity* entity, Behaviour flags);
void AddBehaviour(Entity* entity, Behaviour flags);
void RemoveBehaviour(Entity* entity, Behaviour flags);
```

We create the function declarations inside `entity.h` . Then we add the same content of these functions that we used to have inside the struct to `entity.cpp` .

```cpp
// entity.cpp
bool HasBehaviour(Entity* entity, Behaviour flags) {
  return (entity->behaviour & flags) == flags;
}

void InitializeBaseBehaviour(Entity* entity) {
  assert(entity->id != ID::NONE);
  switch (entity->id) {
    default:
      SetBehaviour(entity, NONE);
      break;
    case ID::DEMON:
      SetBehaviour(entity, (Behaviour)(CAN_MOVE | IS_PLAYER | RESPOND_TO_INPUT));
      entity->strength = 2;
      break;
    case ID::ROCK:
      SetBehaviour(entity, (Behaviour)CAN_MOVE);
      break;
  }
}

void SetBehaviour(Entity* entity, Behaviour flags) {
  entity->behaviour = flags;
}

void AddBehaviour(Entity* entity, Behaviour flags) {
  entity->behaviour = (Behaviour)(entity->behaviour | flags);
}

void RemoveBehaviour(Entity* entity, Behaviour flags) {
  entity->behaviour = (Behaviour)(entity->behaviour & ~flags);
}
```

so instead of calling our functions from our `Entity` struct itself we had to refactor our code to pass the enitty along instead. Other than that, the code is identical.
Now on each call site inside `game.cpp` and `levels.cpp` where we used to call these functions from the struct we instead pass the struct along.
for example:

```cpp
// example of refactored changes
// old
if(entity->HasBehaviour(CAN_MOVE) && IsMoving(entity)) { ... }
// new
if(HasBehaviour(entity, CAN_MOVE) && IsMoving(entity)) { ... }
```

If you try and build the game you'll get an error message if you still have remaining places to update the syntax.
This change is just to keep the logic more self-similar across our files.
Next, lets add the features for the Golem
we add an `int strength` to our `Entity` struct

```cpp
// entity.h
struct Entity {
  ID id;
  int strength;
  int x;
  int y;
  int x_prev;
  int y_prev;
  float progress_01;
  Behaviour behaviour;
};
```

I also noticed that for both Entity ID and SPRITE_ID I had `GHOST/Ghost` set up instead of `SIREN/Siren` . So I went to both these sites in `entity.h` and `spriteLibrary.h` and renamed it.
Then in the switch case inside our `IntializeBaseBehaviour()` we make sure to set the `strength` values and add the four characters we've outlined. We'll be returning to this function as we keep adding more logic. For now we're giving Medusa and Demon a 1 in strength, the Siren a 0 and 999 for our Golem (should be enough I imagine).

```cpp
// entity.cpp
void InitializeBaseBehaviour(Entity* entity) {
  assert(entity->id != ID::NONE);
  switch (entity->id) {
    default:
      SetBehaviour(entity, NONE);
      break;
    case ID::DEMON:
      SetBehaviour(entity, (Behaviour)(CAN_MOVE | IS_PLAYER | RESPOND_TO_INPUT));
      entity->strength = 1;
      break;
    case ID::GOLEM:
      SetBehaviour(entity, (Behaviour)(CAN_MOVE | IS_PLAYER | RESPOND_TO_INPUT));
      entity->strength = 999;
      break;
    case ID::MEDUSA:
      SetBehaviour(entity, (Behaviour)(CAN_MOVE | IS_PLAYER | RESPOND_TO_INPUT));
      entity->strength = 1;
      break;
    case ID::SIREN:
      SetBehaviour(entity, (Behaviour)(CAN_MOVE | IS_PLAYER | RESPOND_TO_INPUT));
      entity->strength = 0;
      break;
    case ID::ROCK:
      SetBehaviour(entity, (Behaviour)CAN_MOVE);
      break;
  }
}
```

Now lets update our `TryMove()` to demand we pass along a `strength` value. We can then decrease this value by 1 each time be call a `TryMove()` recursively inside itself, meaning that each time a block pushes a block that pushes a block we reduce this value by 1 . If we ever reach a block in our chain and `strength` is no longer above 0 then we know we're trying to push to many things at once and we can return false , meaning that the move fails.

```cpp
// game.h
bool TryMove(Entity* mover, LevelData* level, CommandBuffer* cmd_buffer, int xDir, int yDir, int timestamp, int strength);
```

then inside `game.cpp` we update by passing the `entity->strength` to `TryMove()` inside our `Update()` function.

```cpp
// game.cpp
for (int i = 0; i < data->GetCurrentLevel()->entityCount; i++) {
  Entity* entity = &data->GetCurrentLevel()->entityBuffer[i];
  if(HasBehaviour(entity, (Behaviour)(RESPOND_TO_INPUT | CAN_MOVE))) {
    int xDir = data->input_buffer[data->input_buffer_read_count % data->input_buffer_capacity].x;
    int yDir = data->input_buffer[data->input_buffer_read_count % data->input_buffer_capacity].y;
    TryMove(entity, data->GetCurrentLevel(), data->commandBuffer, xDir, yDir, data->command_timestamp, entity->strength);
  }
}
data->input_buffer_read_count++;
```

Then inside the `TryMove()` function itself, at the recursive call site we pass along `strength` but only after we've reduced its value by 1 using the decrement operator aka `--` . Meaning that each time we recursively call `TryMove()` we will be passing along a lower value for `strength` .

```cpp
// game.cpp
if(HasBehaviour(stepInto_entity, CAN_MOVE)) {
  if(TryMove(stepInto_entity, level, cmd_buffer, xDir, yDir, timestamp, --strength)) {
    MoveCommand mv;
    mv.type = CMD_TYPE::MOVE;
    mv.entity = mover;
    mv.xDir = xDir;
    mv.yDir = yDir;
    Push(cmd_buffer, mv, timestamp);
    return true;
  }
}
```

then at the top of `TryMove()` we'll make an if-statement that reacts to the value of `strength` . But notably its not the strength of the entity, its the value of the variable named `strength` that was passed into the function as one of its parameters

```cpp
// game.cpp
if(strength < 0) {
  return false;
}
```

That's it, now the Golem is really strong, the Siren can't push at all and the Demon and Medusa can push one block.
We're also doing some refactoring to `GetSpriteFromID()` inside `spriteLibrary.cpp` . We want to always make sure we return fallback if we didn't find a sprite at the specified index of our `spriteBuffer` . Previously we could have invisible objects. Now they all at least show up. I've also taken the liberty to prepare for when we have art for the different characters.

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
      sprite_to_return = &spriteBuffer[(int)SPRITE_ID::Medusa];
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

So we create a `sprite_to_return` pointer and assign to it using our `switch-case` . Then at the end if we didn't enter a switch case to give it a value or whatever was returned to us didn't have a texture set aka was `nullptr` we can set it to the `ID::Fallback` sprite.
We are allowed to stack our conditions inside our if-statement to include `->texture` in the second condition. This would cause a crash if we changed from `||` to `&&` as `&&` evaluates all statements and would sometimes not find the texture pointer in the second condition as the `sprite_to_return` might be a `nullptr` and can therefore not have any variables set. the `||` operator will evaluate the leftmost condition first, and if it was false it never checks the next condition. Making it so that at the evaluation of the second condition we can be sure that `sprite_to_return` was in fact not a `nullptr` .
Lets tackle the fact that we want to limit Medusas petrification ability to only her facing direction, as well as actually making the ability work as intended. This is a larger feature and will require more new code than the Golem did.
We're going to start with a series of refactoring steps to improve our codebase a bit, as well as help us with the foundation for the Medusa ability inclusion.
We're adding a new Command a `RotateCommand` and a new `CMD_TYPE` enum. This will be created inside of our `command.h` file.

```cpp
// command.h
enum class CMD_TYPE : uint8_t {
  NONE = 0,
  MOVE = 1,
  ROTATE = 2
};

struct RotateCommand : Command {
  Entity* entity;
  Direction from;
  Direction to;

  RotateCommand(Entity* entity, Direction from, Direction to) {
    this->entity = entity;
    this->from = from;
    this->to = to;
    type = CMD_TYPE::ROTATE;
  }
};
```

We inherit from `Command` (just as we do with `MoveCommand` ). We store an entity pointer and two `Direction` enums to keep track of where we used to look and where we are looking now. then we create a constructor for this Command . A constructor is a function with no return type that has the exact same name as the actual struct. This function, once it exists, is suddenly a required function to call when creating a new struct of this type. We pass in three parameters that correspond with those stored in the struct itself. Because the parameters we pass in have the same name as our structs parameters we have to use the `this->` keyword to specify which is from the struct and which one is a parameter being passed in. If this is confusing you can add a prefix to the parameters aka `Entity* _entity` or similar.
We also set the type directly from the constructor .
lets go ahead and give `MoveCommand` a constructor as well

```cpp
// command.h
struct MoveCommand : Command {
  Entity* entity;
  int xDir;
  int yDir;

  MoveCommand(Entity* entity, int xDir, int yDir) {
    this->entity = entity;
    this->xDir = xDir;
    this->yDir = yDir;
    type = CMD_TYPE::MOVE;
  }
};
```

same variables as before, now just with a constructor that passes in the specified variables and sets type on its own.
We're also making an addition to `AnyCommand` so it holds a `RotateCommand` and so that our new `RotateCommand` can be treated as an `AnyCommand` when passed along.

```cpp
// command.h
union AnyCommand {
  Command command;
  MoveCommand move;
  RotateCommand rotate;

  AnyCommand(MoveCommand mv) {
    move = mv;
  };

  AnyCommand(RotateCommand rc) {
    rotate = rc;
  };
};
```

This is exactly what we did for `MoveCommand` earlier in the series.
If we return to our base `Command` struct we can give `type` a default value of `NONE` . This will help us catch a nasty bug.

```cpp
// command.h
struct Command {
  CMD_TYPE type = CMD_TYPE::NONE;
  uint32_t timestamp;
};
```

Now unless a new Command remembers to set its type it will be set to `NONE` . Then in our `command.cpp` we can abort our program if this ever happens. Because if we ever push a command without a type we know that we have screwed up and need to fix the issue

```cpp
// command.cpp
void Push(CommandBuffer* buffer, AnyCommand cmd, LevelData* level, uint32_t timestamp) {
  assert(cmd.command.type != CMD_TYPE::NONE);
  // other code hidden for clarity
}
```

This will just terminate our entire program and flag the issue for us. A good safeguard against forgetting to set up our struct correctly. This is a defensive coding habit that we can leverage to help us spend less time hunting strange bugs that we don't know the cause of.
Note how we've added `LevelData*` as a parameter to our `Push` function. We are refactoring our `Push, Execute` and `Redo` functions inside `command.h/cpp` to have `LevelData*` as a parameter, we'll be needing this later to help our entities use their abilities on the board after a move or rotation.

```cpp
// command.h
void Push(CommandBuffer* buffer, AnyCommand cmd, LevelData* level, uint32_t timestamp);
void Redo(CommandBuffer* buffer, LevelData* level);
```

Then inside `command.cpp` we need to update the same parameters as well.

```cpp
// command.cpp
void Execute(AnyCommand cmd, LevelData* level, bool from_redo = false) {
  // code hidden for clarity
}

void Push(CommandBuffer* buffer, AnyCommand cmd, LevelData* level, uint32_t timestamp) {
  // code hidden for clarity
  Execute(cmd, level);
}

void Redo(CommandBuffer *buffer, LevelData* level) {
  // a lot of code hidden for clarity
  Execute(cmd, level, true);
  // a lot of code hidden for clarity
  Redo(buffer, level);
}
```

With this change we need to update the code where we call our `Push` and `Redo` functions in `game.cpp` and `dev_gui.cpp` to also pass along `level` .

```cpp
// dev_gui.cpp
void Draw_History(CommandBuffer* buffer, LevelData* level) {
  int sliderPos = buffer->index;
  if(ImGui::SliderInt("history", &sliderPos, 0, buffer->head)) {
    while(buffer->index > sliderPos) {
      Undo(buffer);
    }
    while(buffer->index < sliderPos) {
      Redo(buffer, level);
    }
  }
}
```

we needed to pass along `LevelData*` to our `Draw_History` function as `Redo()` needs this parameter.
Also Inside `game.cpp` we're creating two `MoveCommand`s lets update those callsite to instead use our new constructor

```cpp
// game.cpp
bool TryMove(Entity* mover, LevelData* level, CommandBuffer* cmd_buffer, int xDir, int yDir, int timestamp, int strength) {
  // code above hidden for clarity
  if(stepInto_entity == nullptr) {
    if(stepInto_tile_id == ID::GROUND) {
      MoveCommand mv(mover, xDir, yDir);
      Push(cmd_buffer, mv, level, timestamp);
      return true;
    }
    return false;
  }
  if(HasBehaviour(stepInto_entity, CAN_MOVE)) {
    if(TryMove(stepInto_entity, level, cmd_buffer, xDir, yDir, timestamp, --strength)) {
      MoveCommand mv(mover, xDir, yDir);
      Push(cmd_buffer, mv, level, timestamp);
      return true;
    }
  }
  return false;
}
```

previously we set each variable (and the type manually) on individual rows, now we pass the variables all in one place from the constructor call.

```cpp
// example
ATypeOfStruct aVariableName(variable_1, variable_2, ... etc);
```

This is the syntax for creating a struct and passing along variables to its constructor.
Next we're going to `levels.h` where we will move the functions from inside the `Level` struct outside of it and just placing their declarations in the .h and their bodies in the `levels.cpp` file.
With this change we also have to add a `LevelData* level` parameter to each function as we now have to pass the `LevelData*` into it rather than having it placed inside our struct. We're also creating a new function `RaycastFirstEntity()`

```cpp
// levels.h
#pragma once
#include "arena.h"
#include "entity.h"
#include <cstdint>

using namespace Memory;

struct LevelData {
  int w;
  int h;
  uint8_t* cells;
  const char* level_path;
  Entity* entityBuffer;
  int entityCount;
};

void CreateLevel(Arena* arena, LevelData* level, const char* level_name);
void CreateEntities(LevelData* lvl_data, Arena* arena);
Entity* GetNextAvailableEntity(Entity* entityBuffer, int bufferSize);
void AddEntity(ID entity_id, int x, int y, LevelData* level);
void RemoveEntity(int x, int y, LevelData* level);
uint8_t GetCellID(LevelData* level, int x, int y);
Entity* GetEntity(LevelData* level, int x, int y);
Entity* RaycastFirstEntity(int x_origin, int y_origin, Direction direction, LevelData* level, bool ignore_walls = false);
```

We'll get back to our new `Raycast` function soon. But first we'll update our `levels.cpp` with the functions previously inside our `LevelData` struct.

```cpp
// levels.cpp
uint8_t GetCellID(LevelData* level, int x, int y) {
  return level->cells[y * level->w + x];
}

Entity* GetEntity(LevelData* level, int x, int y) {
  for (int i = 0; i < level->entityCount; i++) {
    if(level->entityBuffer[i].x == x && level->entityBuffer[i].y == y) {
      return &level->entityBuffer[i];
    }
  }
  return nullptr;
}
```

They are identical except that we now have to use `level->` to fetch the necessary variables.
With this update to `levels.h/cpp` all the places where we call `GetCellID()` and `GetEntity()` are now broken. This is because none of these call sites pass along `LevelData*` as a parameter and all of them try and access the function from the level variable itself `level->GetCellID()` .
Here's a list of the affected files

- `levels.cpp`
- `game.cpp`
- `levelrenderer.cpp`

In each of these files we need to make the following change

```cpp
// example of change to level functions
// old
level->GetCellID(x, y);
// new
GetCellID(level, x, y);
// old
level->GetEntity(x, y);
// new
GetEntity(level, x, y);
```

Open each file and find the broken callsites then adjust them to match the new syntax.
With that done we can create our new `Raycast` function inside `levels.cpp` . A raycast is a "laser" that we fire from a point in space in a specific direction (also called a vector ) then if that "laser" hits something then we return what it was. With us having four directions we're not duplicating our code four times to handle all of these cases. Instead we do a bit of clever coding to allow all directions to work with the same function

```cpp
// levels.cpp
Entity* RaycastFirstEntity(int x_origin, int y_origin, Direction direction, LevelData* level, bool ignore_walls) {
  Position facingVector;
  switch (direction) {
    case Direction::RIGHT:
      facingVector = {1, 0};
      break;
    case Direction::LEFT:
      facingVector = {-1, 0};
      break;
    case Direction::UP:
      facingVector = {0, 1};
      break;
    case Direction::DOWN:
      facingVector = {0, -1};
      break;
  }
  int x_search = x_origin + facingVector.x;
  int y_search = y_origin + facingVector.y;
  while(x_search > 0 && x_search < level->w && y_search > 0 && y_search < level->h) {
    ID cellID = (ID)GetCellID(level, x_search, y_search);
    if(cellID == ID::WALL && !ignore_walls) {
      break;
    }
    Entity* entity_search = GetEntity(level, x_search, y_search);
    if(entity_search != nullptr) {
      return entity_search;
    }
    x_search += facingVector.x;
    y_search += facingVector.y;
  }
  return nullptr;
}
```

we're creating a `Position` variable that we set to store a different x and y value depending on the `Direction` variable we provided. This will be the direction our laser travels. We then use our `x/y_search` integers to act as the point we're evaluating. These numbers will increase by the contents of our `facingVector` each time the loop runs. We start of by immediatly adding `facingVector.x/y` to it as we don't want to evaluate the cell that we started from, as that would mean that we always shot our laser into the origin cell and return back the entity that is standing there.
Our while-loop has four condtions that all have to be true for the loop to continue. because this function handles movement in all directions we have to check that x and y remain inside the level bounds both in the positive and negative directions.
Once inside the while-loop we check if we have encountered a wall, and if we're not allowed to `ignore_walls` then we `break` the loop causing us to move down and return `nullptr` . If we do not stop at a wall then we continue and check if there is an Entity at the specific position. If it did we can return that entity and stop the raycast - we found our closest target from the origin moving in the specified vector/direction.
If we didn't encounter a wall or an entity we add the value of `facingVector` to the `x/y_search` variables. And with only one of these always being 0 and the other being either 1 or -1 we ensure that we continue searching in the same direction.
We will need a Behaviour to toggle whether or not an Entity has been turned to stone. Lets update our enum inside `entity.h`

```cpp
// entity.h
enum Behaviour : uint32_t {
  NONE = 0,
  CAN_MOVE = 1 << 0,
  IS_PLAYER = 1 << 1,
  RESPOND_TO_INPUT = 1 << 2,
  IS_PETRIFIED = 1 << 3,
  CAN_ROTATE = 1 << 4,
  UNPUSHABLE = 1 << 5
};
```

We're adding 3 new Behaviours at this stage, one to control if we are allowed to rotate the entity (rocks won't rotate). One to check if the entity is petrified aka turned-to-stone and the last is a future addition that we'll use to make the Golem impossible to push due to being very heavy.
Next we're going to add our `RotateCommand` to `Game.cpp` . We want to rotate an entity only if they moved due to a player input and we want to rotate them even if they did not manage to actually perform a move.

```cpp
// game.cpp
for (int i = 0; i < data->GetCurrentLevel()->entityCount; i++) {
  Entity* entity = &data->GetCurrentLevel()->entityBuffer[i];
  if(HasBehaviour(entity, (Behaviour)(RESPOND_TO_INPUT | CAN_MOVE))) {
    if(HasBehaviour(entity, Behaviour::IS_PETRIFIED)) {
      continue;
    }
    int xDir = data->input_buffer[data->input_buffer_read_count % data->input_buffer_capacity].x;
    int yDir = data->input_buffer[data->input_buffer_read_count % data->input_buffer_capacity].y;
    Direction new_facing = DirectionFromXY(xDir, yDir);
    if(new_facing != entity->facing) {
      RotateCommand rotate(entity, entity->facing, new_facing);
      Push(data->commandBuffer, rotate, data->GetCurrentLevel(), data->command_timestamp);
    }
    TryMove(entity, data->GetCurrentLevel(), data->commandBuffer, xDir, yDir, data->command_timestamp, entity->strength);
  }
}
data->input_buffer_read_count++;
```

We can now stop petrified entities from being allowed to move/rotate on their own due to player inputs by checking if the `IS_PETRIFIED` flag was set an if so `continue` immediately to the next entity.
We then use a helper function `DirectionFromXY` to convert the direction we're moving in into our `Direction` enum. As this is a super small function I've opted to place it inside the `entity.h` directly which is possible using the `inline` attribute

```cpp
// entity.h
enum class Direction {
  RIGHT,
  LEFT,
  UP,
  DOWN
};

inline Direction DirectionFromXY(int xDir, int yDir) {
  assert(xDir * yDir == 0);
  if(xDir == 1) { return Direction::RIGHT; }
  if(xDir == -1) { return Direction::LEFT; }
  if(yDir == 1) { return Direction::UP; }
  else { return Direction::DOWN; }
}
```

This small function first does a safety assertion (defensive coding style) where we make sure that only either `xDir` or `yDir` had a non-zero value. We can do this by multiplying them together. Only if both had a non-zero value will the result not be zero. We have also added `class` to our `Direction` enum so that we need to type `Direction::` in order to use them. We do this to reduce the chance of accidental mixups.
Then we check which values were passed in and return the relevant `Direction` enum.
Back in `game.cpp` we compare this `new_facing` with the current facing direction of the entity. If they were different we go ahead and create a new `RotateCommand` using its constructor and then `Push` it to the `commandBuffer` to be executed.
After the rotation we call our `TryMove` as usual.
Right now the `Execute()` of our `RotateCommand` does nothing as we've not updated our `Execute()` function inside `command.cpp` yet

```cpp
// command.cpp
void Execute(AnyCommand cmd, LevelData* level, bool from_redo = false) {
  switch(cmd.command.type) {
    case CMD_TYPE::NONE:
      break;
    case CMD_TYPE::MOVE: {
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
    case CMD_TYPE::ROTATE: {
      RotateCommand rotate = cmd.rotate;
      if(!HasBehaviour(rotate.entity, CAN_ROTATE)) {
        break;
      }
      rotate.entity->facing = rotate.to;
      break;
    }
  }
}
```

We've added our case for `CMD_TYPE::ROTATE` and we've also placed all the content of both of our cases inside curly bracers `{}` we have to place these curly bracers around our logic if we have more than 1 case and a case adds a new variable. In this case `MoveCommand mv` and `RotateCommand rotate` .
The fact is that the compiler can't guarantee that a case won't fall into another case and as such all switch cases are actually in the same scope. This means that we could in theory reach `case CMD_TYPE::ROTATE` to work with `MoveCommand mv` without first having created it. Our compiler won't let us write code like this without the curly bracers to set the bounds for each case. It's a bit messy but necessary.
We use our Behaviour flag `CAN_ROTATE` to ensure that only relevant entities perform rotations.
Now besides actually doing our rotation (and move) we need a way of adding our ability to these events. We'll add three new functions to `entity.h/cpp` .

```cpp
// entity.h
void PostMove(Entity* entity, LevelData* level);
void PostRotation(Entity* entity, LevelData* level, Direction from, Direction to);
void PreRotation(Entity* entity, LevelData* level, Direction from, Direction to);
```

As we add these function we are getting a compile error.

Because to have `LevelData` in `entity.h` we need to `#include "levels.h"` but as `levels.h` includes `entity.h` we have a circular include chain that breaks compilation. But with `entity.h` not using the `LevelData` directly, we can forward-declare our `LevelData` to fix the circular dependency.

```cpp
// entity.h
// #include "levels.h" <- removed
struct LevelData;
```

with `entity.h` having a struct declared with the same name we can have function declarations that use the struct. the header file itself doesn't use the struct so it doesn't need to know how it works. Then `entity.cpp` can `#include "levels.h"` and in their functions the `LevelData` will be mapped to the `LevelData` from he included `levels.h` header file.
right now the only logic we need to create is for our Medusa petrification ability, so lets go ahead and add our logic in `entity.cpp`

```cpp
// entity.cpp
void PostMove(Entity *entity, LevelData* level) {
  if(entity->id == ID::MEDUSA) {
    Entity* entity_looked_at = RaycastFirstEntity(entity->x, entity->y, entity->facing, level);
    if(entity_looked_at != nullptr) {
      if(!HasBehaviour(entity_looked_at, Behaviour::IS_PETRIFIED)) {
        AddBehaviour(entity_looked_at, Behaviour::IS_PETRIFIED);
      }
    }
  }
}
```

we check if the entity that moved was Medusa and if so we use our new `Raycast` function along with her facing direction to get the first entity she's lookinb at. If we found an entity we check if it already was petrified, and if not we make it petrified.

```cpp
// entity.cpp
void PostRotation(Entity* entity, LevelData* level, Direction from, Direction to) {
  if(from == to) {
    return;
  }
  if(entity->id == ID::MEDUSA) {
    Entity* entity_looked_at = RaycastFirstEntity(entity->x, entity->y, to, level);
    if(entity_looked_at != nullptr) {
      if(!HasBehaviour(entity_looked_at, Behaviour::IS_PETRIFIED)) {
        AddBehaviour(entity_looked_at, Behaviour::IS_PETRIFIED);
      }
    }
  }
}

void PreRotation(Entity* entity, LevelData* level, Direction from, Direction to) {
  if(from == to) {
    return;
  }
  if(entity->id == ID::MEDUSA) {
    Entity* entity_previously_looked_at = RaycastFirstEntity(entity->x, entity->y, from, level);
    if(entity_previously_looked_at != nullptr) {
      if(HasBehaviour(entity_previously_looked_at, Behaviour::IS_PETRIFIED)) {
        RemoveBehaviour(entity_previously_looked_at, Behaviour::IS_PETRIFIED);
      }
    }
  }
}
```

for our rotations we check if we actually had a change in rotation by comparing `from` and `to` . Then using `to` for `PostRotation` and `from` for `PreRotation` we raycast once again. and in the case of `PostRotation` we petrify the entity we found and for `PreRotation` we remove petrification from the entity as we know that after our rotation has completed we're no longer looking at that entity.
Make sure you pay attention to the fact that our two rotation functions are almost entirely similar except if they `Add` or `Remove` the flag and if the raycast uses `from` or `to` .
Now we can go to `command.cpp` and add these function calls to `Execute()`

```cpp
// command.cpp
void Execute(AnyCommand cmd, LevelData* level, bool from_redo = false) {
  switch(cmd.command.type) {
    case CMD_TYPE::NONE:
      break;
    case CMD_TYPE::MOVE: {
      MoveCommand mv = cmd.move;
      mv.entity->x_prev = mv.entity->x;
      mv.entity->y_prev = mv.entity->y;
      mv.entity->x += mv.xDir;
      mv.entity->y += mv.yDir;
      if(from_redo) {
        mv.entity->progress_01 = 1;
      }
      PostMove(mv.entity, level);
      break;
    }
    case CMD_TYPE::ROTATE: {
      RotateCommand rotate = cmd.rotate;
      if(!HasBehaviour(rotate.entity, CAN_ROTATE)) {
        break;
      }
      PreRotation(rotate.entity, level, rotate.from, rotate.to);
      rotate.entity->facing = rotate.to;
      PostRotation(rotate.entity, level, rotate.from, rotate.to);
      break;
    }
  }
}
```

next we add the logic in `Undo()`

```cpp
// command.cpp
void Undo(CommandBuffer* buffer) {
  switch(cmd.command.type) {
    case CMD_TYPE::NONE:
      break;
    case CMD_TYPE::MOVE: {
      // move undo logic hidden for clarity
      break;
    }
    case CMD_TYPE::ROTATE: {
      RotateCommand rotate = cmd.rotate;
      if(!HasBehaviour(rotate.entity, CAN_ROTATE)) {
        break;
      }
      rotate.entity->facing = rotate.from;
      break;
    }
  }
}
```

In `levelRenderer.cpp` we'll check if an entity is petrified and if so overwrite the its sprite to be the rock sprite instead

```cpp
// levelRenderer.cpp
void RenderEntities(GameData* data, SDL_Renderer* renderer) {
  // above code hidden for clarity
  Sprite* sprite = GetSpriteFromID(entity.id, data->spriteBuffer);
  if(HasBehaviour(&entity, Behaviour::IS_PETRIFIED)) {
    sprite = GetSpriteFromID(ID::ROCK, data->spriteBuffer);
  }
  // code hidden for clarity
}
```

Lastly we need to fix our Undo/Redo logic to work with the changes to Behaviour . Currently our code works as we want when moving and rotating our entities. but as we Undo our steps our game breaks.
This is because our Add/Remove Behaviour functions are not part of our Command structure yet.
We're going to do some refactoring then resolve this.
As I was programming the BehaviourCommand logic I found myself passing `command_timestamp` to a bunch of functions to pass it along to the `PostMove` and `PostRotate` functions. This was not a dramatic issue but I still felt that it was unecessarily prone to mistakes so lets go ahead and include our `command_timestamp` in our `CommandBuffer` struct and remove the `uint32_t command_timestamp` from our `GameData` struct

```cpp
// gameState.h
struct GameData {
  // uint32_t command_timestamp // <---- removed
};
```

then add it in our `CommandBuffer` instead

```cpp
// command.h
struct CommandBuffer {
  AnyCommand* allCommands;
  int capacity;
  int index;
  int head;
  uint32_t timestamp;
};
```

Now of course all parts of our code base where we previously passed in `command_timestamp` will break and we need to fetch `timestamp` from our `CommandBuffer` struct instead. And any function declaration and parameters in .h and .cpp files need to remove the `timestamp` parameter. A list of affected functions

- `TryMove()` in `game.h/cpp`
- `Push()` in `command.h/cpp`
- `Update()` inside `game.cpp` increases timestamp by 1

Lets set up a new Command that will handle changes to Behaviour .

```cpp
// command.h
struct ModifyBehaviourCommand : Command {
  enum Mode {
    ADD,
    REMOVE
  };
  Entity* entity;
  Behaviour flag;
  Mode mode;

  ModifyBehaviourCommand(Entity* entity, Behaviour flag, Mode mode) {
    this->entity = entity;
    this->flag = flag;
    this->mode = mode;
    type = CMD_TYPE::MODIFY_BEHAVIOUR;
  }
};
```

the internal enum `Mode` is just to make it clearer if the command is adding or removing a flag. in a previous iteration it was just a bool. It worked but when constructing the Command the bool was not immediately understood.
I've opted for this approach instead of having an `AddBehaviourCommand` and a `RemoveBehaviourCommand` as that felt more prone to create divergent behaviour if one is updated and we forget to change the other.
As per usual with a new command we:

1. set it up
2. add a constructor to it
3. add another `CMD_TYPE` . in this case `MODIFY_BEHAVIOUR`
4. add the command to `union AnyCommand`
5. create a constructor inside `AnyCommand` that takes in our `ModifyBehaviourCommand`

```cpp
// command.h
union AnyCommand {
  Command command;
  MoveCommand move;
  RotateCommand rotate;
  ModifyBehaviourCommand modify;

  AnyCommand(MoveCommand mov) {
    move = mov;
  };
  AnyCommand(RotateCommand rot) {
    rotate = rot;
  };
  AnyCommand(ModifyBehaviourCommand mod) {
    modify = mod;
  }
};
```

Now we need to pass along `CommandBuffer` to `PostMove()` , `PreRotate()` and `PostRotate()`

```cpp
// entity.h
void PostMove(Entity* entity, LevelData* level, CommandBuffer* commandBuffer);
void PostRotation(Entity* entity, LevelData* level, CommandBuffer* commandBuffer, Direction from, Direction to);
void PreRotation(Entity* entity, LevelData* level, CommandBuffer* commandBuffer, Direction from, Direction to);
```

then inside those functions where we previously just called `Add/RemoveBehaviour` we now create our `ModifyBehaviourCommand` and push it.

```cpp
// entity.cpp
void PostMove(Entity *entity, LevelData* level, CommandBuffer* commandBuffer) {
  if(entity->id == ID::MEDUSA) {
    Entity* entity_looked_at = RaycastFirstEntity(entity->x, entity->y, entity->facing, level);
    if(entity_looked_at != nullptr) {
      if(!HasBehaviour(entity_looked_at, Behaviour::IS_PETRIFIED)) {
        ModifyBehaviourCommand modify(entity_looked_at, Behaviour::IS_PETRIFIED, ModifyBehaviourCommand::ADD);
        Push(commandBuffer, modify, level);
      }
    }
  }
}

void PostRotation(Entity* entity, LevelData* level, CommandBuffer* commandBuffer, Direction from, Direction to) {
  if(from == to) {
    return;
  }
  if(entity->id == ID::MEDUSA) {
    Entity* entity_looked_at = RaycastFirstEntity(entity->x, entity->y, to, level);
    if(entity_looked_at != nullptr) {
      if(!HasBehaviour(entity_looked_at, Behaviour::IS_PETRIFIED)) {
        ModifyBehaviourCommand modify(entity_looked_at, Behaviour::IS_PETRIFIED, ModifyBehaviourCommand::ADD);
        Push(commandBuffer, modify, level);
      }
    }
  }
}

void PreRotation(Entity* entity, LevelData* level, CommandBuffer* commandBuffer, Direction from, Direction to) {
  if(from == to) {
    return;
  }
  if(entity->id == ID::MEDUSA) {
    Entity* entity_previously_looked_at = RaycastFirstEntity(entity->x, entity->y, from, level);
    if(entity_previously_looked_at != nullptr) {
      if(HasBehaviour(entity_previously_looked_at, Behaviour::IS_PETRIFIED)) {
        ModifyBehaviourCommand modify(entity_previously_looked_at, Behaviour::IS_PETRIFIED, ModifyBehaviourCommand::REMOVE);
        Push(commandBuffer, modify, level);
      }
    }
  }
}
```

Then we need to change `Execute()` to also have `CommandBuffer*` as a parameter so it can be passed to the three functions. Besides this we are also adding the new Command to `Execute()` and `Undo()`

```cpp
// command.cpp
void Execute(AnyCommand cmd, LevelData* level, CommandBuffer* commandBuffer, bool from_redo = false) {
  // code hidden for clarity
  case CMD_TYPE::MODIFY_BEHAVIOUR: {
    ModifyBehaviourCommand modify = cmd.modify;
    if(modify.mode == ModifyBehaviourCommand::ADD) {
      AddBehaviour(modify.entity, modify.flag);
    }
    else {
      RemoveBehaviour(modify.entity, modify.flag);
    }
    break;
  }
}
```

because our enum is part of our struct we can only access it by specifying the struct name first `ModifyBehaviourCommand::ADD/REMOVE` .
Then we canjust invert the `Add/RemoveBehaviour` in our `Undo()`

```cpp
// command.cpp
void Undo(CommandBuffer* buffer) {
  // code hidden for clarity
  case CMD_TYPE::MODIFY_BEHAVIOUR: {
    ModifyBehaviourCommand modify = cmd.modify;
    if(modify.mode == ModifyBehaviourCommand::ADD) {
      // note that this is flipped compared to Execute()
      RemoveBehaviour(modify.entity, modify.flag);
    }
    else {
      AddBehaviour(modify.entity, modify.flag);
    }
    break;
  }
}
```

As a final step we'll return to our Golem and do something simple. Lets stop any entity from pushing him. We have to give him the `UNPUSHABLE` behaviour in `InitializeBaseBehaviour`

```cpp
// entity.cpp
void InitializeBaseBehaviour(Entity* entity) {
  assert(entity->id != ID::NONE);
  switch (entity->id) {
    default:
      // ...
      break;
    case ID::DEMON:
      // ...
      break;
    case ID::GOLEM:
      SetBehaviour(entity, (Behaviour)(CAN_ROTATE | CAN_MOVE | IS_PLAYER | RESPOND_TO_INPUT));
      AddBehaviour(entity, Behaviour::UNPUSHABLE);
      entity->strength = 999;
      break;
    case ID::MEDUSA:
      // ...
      break;
    case ID::SIREN:
      // ...
      break;
    case ID::ROCK:
      // ...
      break;
  }
}
```

and then its just a single line change inside `TryMove()`

```cpp
// game.cpp
bool TryMove(Entity* mover, LevelData* level, CommandBuffer* cmd_buffer, int xDir, int yDir, int strength) {
  // other code hidden for clarity
  // here we also check against `UNPUSHABLE`
  if(HasBehaviour(stepInto_entity, CAN_MOVE) && !HasBehaviour(stepInto_entity, UNPUSHABLE)) {
    if(TryMove(stepInto_entity, level, cmd_buffer, xDir, yDir, --strength)) {
      // ...
    }
  }
}
```

That's it. Nice to end on a win.
This chapter was long, and probably more difficult, great job getting to the end of it!