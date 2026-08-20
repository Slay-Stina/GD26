# 16 Command Pattern

Now that we have our desired functionality we will (once again) refactor it. We're going to take the concepts for moving our entities on the level and create a structure that allows us to undo and redo our movement.
When we move our player (or box) their x and y both update, but they have no memory of where they stood previously. We need to keep some sort of data that tracks entities and where they have gone. Then we need to be able to go back (and forth) in this chain using Z on our keyboard.
One could imagine many solutions on how to store all the previous positions of an entity though there exists a common solution, called a pattern , that we can leverage.
a pattern is just a way to structure code that is tailored made to solve a specific issue. These have been developed as a lot of codebases face similar challenges and these have proven to be a good fit to solve them.
We'll be implementing the Command Pattern . This is designed to store all the relevant data associated with the execution of some code, allowing us to keep the action as data to be accessed later.
Lets begin by setting up our command.h

```cpp
// command.h
#pragma once
#include <cstdint>

enum class CMD_TYPE : uint8_t {
  NONE = 0,
  MOVE = 1
};

struct Command {
  CMD_TYPE type;
};
```

Our Command struct does very little, it stores an enum that we can set to specify what type of Command it is going to be. Currently our logic only calls for a way to store and execute the movement of an entity, so our CMD_TYPE only has one relevant value `MOVE` . But as a game of this type would expand, so would this list.
We will be creating new Command structs that inherit from this base struct. By doing so all future command structs will have access to the `CMD_TYPE` type variable. We'll be using this to determine what type of Command it was that we tried to undo.

```cpp
// command.h
struct MoveCommand : Command {
  Entity* entity;
  int xDir;
  int yDir;
};
```

Here we have our MoveCommand responsible for holding an entity and the direction it is going to move in. This is the same data that we fetch from our `Update()` function in `game.cpp` and then use in the `TryMove` function.
We'll need a way of storing our different commands as an array as well as a way of knowing which command we're currently trying to undo/redo. We'll accomplish this by allocating all our commands all at once in an arena. We'll store them in a new struct inside `command.h`

```cpp
// command.h
struct CommandBuffer {
  AnyCommand* allCommands;
  int capacity;
  int index;
  int head;
};
```

You'll notice that `AnyCommand` is a new type that we haven't talked about yet. `capacity` holds the bounds of the array by setting a count. `index` is an indicator between 0 and `capacity` that tells us which command we're on. we'll also store the value of `index` in `head` . This will allow us to know how much we are allowed to redo once we begin walking `index` backwards as we undo our commands.
`AnyCommand` is a new type called `union` . It helps us solve an otherwise annoying problem. Our commands, depending on their variables, will be of different sizes. But the only way of pre-allocating them and accessing them with an array indicator is to have each command take up the same space in memory.

```cpp
// command.h
union AnyCommand {
  Command command;
  MoveCommand move;
};

AnyCommand(MoveCommand mv) {
  move = mv;
};
```

The `union` keyword makes this new `AnyCommand` have the same size as the largest struct it could represent. This means that when we allocate `AnyCommand`s we allocate the largest command meaning that we're sure that each slot in memory is large enough to fit any of the commands we're using. Without this we would get less data than we needed when fetching large commands if we had allocated the base Command struct.
We're also creating what is called a constructor for `AnyCommand` this is needed to allow a command like `MoveCommand` to be cast into `AnyCommand` . We'll be needing this in order to simplify creating a new `MoveCommand`. Lets compare the syntax needed if we use or skip this constructor

```cpp
// without the constructor
AnyCommand command;
command.move.xDir = 1;
Push(command);

// with the constructor
MoveCommand move;
move.xDir = 1;
Push(move);
```

without the constructor we have to explicitly create `AnyCommand`s then access the correct command from it.
The constructor is a function without a custom name, just the type directly. In this case `AnyCommand` , we then pass in necessary data, the `MoveCommand` we'll be creating. Inside the Constructor function we then assign the values of the variable `move` with the provided `mv` .
We could continue without these constructors, but it makes the code we'll write later easier as we can more or less forget about the `AnyCommand` struct and work with the commands directly.
Finally we need to create three functions, these are responsible for adding a new command to the array, undoing a command and redoing a command

```cpp
// command.h
void Push(CommandBuffer* buffer, AnyCommand cmd);
void Undo(CommandBuffer* buffer);
void Redo(CommandBuffer* buffer);
```

We've opted for calling the function that adds a new command to the array `Push()` as this is the normal syntax we'll find if we work with something called a queue data type. This logic we've set up imitates the same logic as a queue .
Inside `command.cpp` we'll add the bodies to these functions as well as creating a new `Execute()` function that takes `AnyCommand` and runs the logic that we want. In our case, moving the player and box(es). The reason we don't have our `Execute()` function in our `command.h` is because we don't want any script other than `command.cpp` to be able to call this function.

```cpp
// command.cpp
void Execute(AnyCommand cmd) {
  switch(cmd.command.type) {
    case CMD_TYPE::NONE:
      break;
    case CMD_TYPE::MOVE:
      MoveCommand mv = cmd.move;
      mv.entity->x += mv.xDir;
      mv.entity->y += mv.yDir;
      break;
  }
}
```

the `Execute()` function uses the `.type` enum held in the Command base class to determine which type of Command we've passed in. We then use a switch case to run the correct code.
a switch case allows us to define multiple cases that something could be, then only run the code inside the relevant case.
for example

```cpp
// example
switch(player.health) {
  case <= 0:
    cout << "you're dead" << endl;
    break;
  case player.maxHealth:
    cout << "you feel great" << endl;
    break;
  default:
    cout << "you're hurt but alive" << endl;
    break;
}
```

in this example `player.health` is checked to be either at or below zero or at the maximum `player.maxHealth` . We also use the `default` syntax. This is selected when there is no other case that fits. The `break` sets the end of a case, so the code doesn't continue into the next case.
in our `Execute` function we can use the switch to determine what command we're working with.

```cpp
// command.cpp
case CMD_TYPE::MOVE:
  MoveCommand mv = cmd.move;
  mv.entity->x += mv.xDir;
  mv.entity->y += mv.yDir;
  break;
```

We take the `MoveCommand` from the `AnyCommand` union and then work with its data. The `MoveCommand` holds a pointer to an entity as well as the direction to move it. We used to update the entity x and y inside `game.cpp` but we're moving it here instead.
If we create more Commands we need to add them to our `AnyCommand` union, create their constructor then use their variables inside our `Execute` function to actually do something.
It is extra code, but it's actually very manageable. But(!) it's very very important to understand that this code has made our game logic less simple, we've created a layer of abstraction in our system. We're doing this because this makes the logic responsible for undo/redo trivially easy - that is why it is worth it.
Next, lets look at our `Push()` function

```cpp
// command.cpp
void Push(CommandBuffer* buffer, AnyCommand cmd) {
  buffer->allCommands[buffer->index] = cmd;
  buffer->index++;
  buffer->head == buffer->index;
  Execute(cmd);
}
```

We take the `AnyCommand` that we've passed in as a parameter and assign it to the specific element in our array indicated by our current `index` . We do this storage step to later allow us to undo the command.
We then increment our `index` by 1 by using the increment operator `++` . the code below does the same thing.

```cpp
// increment operator example
buffer->index += 1;
buffer->index++;
```

We then store this new value of `index` in `head` . We only update the value of `head` when we `Push()` a new Command, meaning that it is always synced with the last pushed command in our chain.
lastly we call `Execute()` and pass along our `cmd` .
Lets look at our `Undo()` function next

```cpp
// command.cpp
void Undo(CommandBuffer* buffer) {
  if(buffer->index == 0) {
    return;
  }
  buffer->index--;
  AnyCommand cmd = buffer->allCommands[buffer->index];
  switch(cmd.command.type) {
    case CMD_TYPE::NONE:
      break;
    case CMD_TYPE::MOVE:
      MoveCommand mv = cmd.move;
      mv.entity->x -= mv.xDir;
      mv.entity->y -= mv.yDir;
      break;
  }
}
```

First we check if we're currently at the very first command `index` is 0 if we are then there is nothing left to undo and we return early.
otherwise we decrement `index` by 1 using the decrement operator `--` .
Like with the increment operator this removes 1 just as if we'd written `index -= 1`
Once we have set our `index` to point to the previous command, which is actually the last command we executed we do the very same switch case syntax but this time we use the variables stored in our `MoveCommand` to reverse what happened during `Execute()` . In the case of a `MoveCommand` we move the entity back in the opposite direction using `-=` instead of `+=` .
Then as we add new commands we will make sure to add the actual logic to both switch cases in `Execute()` and `Undo()` .
Lastly we have `Redo()`

```cpp
// command.cpp
void Redo(CommandBuffer *buffer) {
  AnyCommand cmd = buffer->allCommands[buffer->index];
  if(cmd.command.type == CMD_TYPE::NONE) {
    return;
  }
  if(buffer->index == buffer->head) {
    return;
  }
  buffer->index++;
  Execute(cmd);
}
```

it is almost exactly the same as our `Push` except we don't pass in a Command to assign to the array. We just fetch the current one by `index` . if we already are at our furthest point aka our `head` then we return early.
then we increment `index` and `Execute()` the command again. The command stored at `index` is the last one we undid. By not syncing `head` to `index` as we do in `Push()` we maintain the furthest point we're allowed to redo. only when we push a new command does `head` update. This means that if we have made 100 moves, then undone 40 of those our `head` is at 100 and our `index` is at 40. meaning we have 60 commands that we are allowed to redo. but if we push a new Command our `index` will increment to 41 and our `head` will sync back with `index` making the commands between 41-100 unaccesible as we've started on a totally new path and the old commands beyond 40 are no longer relevant. This solution allows us to overwrite the contents of the Commands stored after 41.
Now we need to add our `CommandBuffer` pointer and a new `Memory::Arena*` to our `GameData` struct

```cpp
// gameState.h
struct GameData {
  // other variables inside struct removed from clarity
  Memory::Arena* arena_commands;
  CommandBuffer* commandBuffer;
};
```

We will allocate our `CommandBuffer` to our `arena_main` so that it is never removed. Then our `AnyCommand*` array will be allocated to our new `arena_commands` that itself is a sub-arena of `arena_levels`
so inside our `main.cpp` we add these allocations

```cpp
// main() inside main.cpp
Memory::Arena* arena_main = new Memory::Arena();
Memory::Initialize(arena_main, game_memory, GAME_MEMORY_ALLOWANCE);
GameData* gameData = (GameData*)Memory::Allocate(arena_main, sizeof(GameData));
size_t IMAGE_ARENA_SIZE = sizeof(Image) * 100;
gameData->arena_images = Memory::CreateSubArena(arena_main, IMAGE_ARENA_SIZE);
gameData->arena_levels = Memory::CreateSubArena(arena_main, MEGABYTES(3));
gameData->arena_entities = Memory::CreateSubArena(gameData->arena_levels, MEGABYTES(1));
gameData->arena_commands = Memory::CreateSubArena(gameData->arena_levels, MEGABYTES(1));

gameData->levelCount = 5;
gameData->levels = (LevelData*)Memory::Allocate(gameData->arena_levels, sizeof(LevelData) * gameData->levelCount);
gameData->keys_previous = (bool*)Memory::Allocate(gameData->arena_levels, sizeof(bool) * SDL_SCANCODE_COUNT);

gameData->commandBuffer = (CommandBuffer*)Memory::Allocate(arena_main, sizeof(CommandBuffer));
gameData->commandBuffer->capacity = 2000;
size_t COMMAND_SIZE = sizeof(AnyCommand) * gameData->commandBuffer->capacity;
gameData->commandBuffer->allCommands = (AnyCommand*)Memory::Allocate(gameData->arena_commands, COMMAND_SIZE);
```

We have just hard-coded capacity to be 2000. Our biggest command `MoveCommand` holds 2 integers and a pointer. this gives us a total of 16 bytes of memory to store a single Command . With 1 megabyte of memory (1 million bytes) allocated to the `arena_command` we can actually store closer to 62500 commands. I've just lazily set the current bounds at 2000.
Now we just have to worry about `game.h/cpp` where we will be using this new logic.
First we have to update our `TryMove()` function signature inside `game.h` to also pass in a `CommandBuffer*` pointer

```cpp
// game.h
// bool TryMove(Entity* mover, LevelData* level, int xDir, int yDir); // old
bool TryMove(Entity* mover, LevelData* level, CommandBuffer* cmd_buffer, int xDir, int yDir);
```

then inside `game.cpp` inside our `TryMove()` function we update the signature to match then remove the code that updated the x and y of the entity (this happens in two places) and instead we create a new `MoveCommand` and passes it to our `Push()` function.
Note: we need to `#include "command.h"` to access these.

```cpp
// game.cpp
bool TryMove(Entity* mover, LevelData* level, CommandBuffer* cmd_buffer, int xDir, int yDir) {
  // some code hidden from clarity
  if(stepInto_entity == nullptr) {
    if(stepInto_tile_id == ID::GROUND) {
      MoveCommand mv;
      mv.type = CMD_TYPE::MOVE;
      mv.entity = mover;
      mv.xDir = xDir;
      mv.yDir = yDir;
      Push(cmd_buffer, mv);
      return true;
    }
    return false;
  }
  if(stepInto_entity->HasBehaviour(CAN_MOVE)) {
    if(TryMove(stepInto_entity, level, cmd_buffer, xDir, yDir)) {
      MoveCommand mv;
      mv.type = CMD_TYPE::MOVE;
      mv.entity = mover;
      mv.xDir = xDir;
      mv.yDir = yDir;
      Push(cmd_buffer, mv);
      return true;
    }
  }
  return false;
}
```

So we create a new `MoveCommand` assign its variables and pass it into `Push()` . Note that `mv.type` comes from `Command` and is accessible becasue `MoveCommand` inherits from `Command` .
Now at our callsite for `TryMove()` we have to update the signature as well. This is done inside `Update()`

```cpp
// game.cpp
if(xChange != 0 || yChange != 0) {
  // TryMove(entity, data->GetCurrentLevel(), xChange, yChange); // old
  TryMove(entity, data->GetCurrentLevel(), data->commandBuffer, xChange, yChange);
}
```

and to use our undo/redo functionality we only have to check if we are pressing or holding the right keys in `Update()`

```cpp
// game.cpp
if(KeyPressed(SDL_SCANCODE_Z, keys, data->keys_previous)) {
  if(KeyHeld(SDL_SCANCODE_LSHIFT, keys, data->keys_previous)) {
    Redo(data->commandBuffer);
  }
  else {
    Undo(data->commandBuffer);
  }
}
```

So if we press Z we undo, and if we press Z whilst holding Left Shift we redo.
With this we have implemented undo/redo functionality by leveraging the battle-tested Command Pattern and despite there being quite a lot of text in this chapter to help explain what we're doing there is surprisingly little actual new code and we only had to make changes to a handful of our previously existing script files.