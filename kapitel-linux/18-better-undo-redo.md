# 18 Better undo/redo

Currently you might have noticed that after we push a block and press undo. We end up in a state that we can't naturally create in game without undoing first. Our block is still pushed away but our player has taken an undo step backwards.
We will solve this by adding a new variable to `GameData` and our base `Command` called `timestamp` .
This variable will be the same for all commands created during the same `Update()` call. This will allow us to keep undoing/redoing until the next command either doesn't exist or has a different timestamp number assigned to it.

```cpp
// GameData struct inside gamestate.h
struct GameData {
  // other variables hidden for clarity
  uint32_t command_timestamp;
};
```

We will increase this `uint32_t` by 1 each time we run `Update()` inside `game.cpp` .

```cpp
// game.cpp
void Update(GameData* data, float dt) {
  // undo/redo keypress code hidden for clarity
  data->command_timestamp += 1;
  for (int i = 0; i < data->GetCurrentLevel()->entityCount; i++) {
    // ...
  }
}
```

at 60 fps this variable will fill to capacity only after hundreds of days, and at that point it just wraps back to 0 and continues again. So lets just not worry about it.
Lets add a similar variable to `Command`

```cpp
// command.h
struct Command {
  CMD_TYPE type;
  uint32_t timestamp;
};
```

Then we need to add a `uint32_t` parameter to our `Push()` command in `command.h/cpp`

```cpp
// command.h
void Push(CommandBuffer* buffer, AnyCommand cmd, uint32_t timestamp);
```

and in .cpp we update the signature and assign the command the provided timestamp

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

We can't assign the timestamp to `cmd` directly as that is a temporary variable that we copy to the `allCommands` array. As soon as we leave the `Push()` function `cmd` stops existing.
We also add the same parameter to our `TryMove()` function inside `game.h/cpp`

```cpp
// game.h
bool TryMove(Entity* mover, LevelData* level, CommandBuffer* cmd_buffer, int xDir, int yDir, int timestamp);
```

Now at all locations in `game.cpp` where we call `TryMove()` and `Push()` we need to provide the timestamp from our `GameData* data` .
Helix will provide us with errors at all locations where this has not been done yet. To go between errors in Helix we can find all the calls for `TryMove()` and `Push()` . We can also go to the function declaration and with the caret over the function name we can press `g-r` to get a list of everywhere the function is being used.
With this done we need to add logic to our `Undo()` and `Redo()` functions inside `command.cpp`

```cpp
// command.cpp
void Undo(CommandBuffer* buffer) {
  if(buffer->index == 0) {
    return;
  }
  buffer->index--;
  AnyCommand cmd = buffer->allCommands[buffer->index];
  uint32_t timestamp = cmd.command.timestamp;
  switch(cmd.command.type) {
    // cases inside switch case hidden for clarity
  }
  if(buffer->index > 0) {
    if(buffer->allCommands[buffer->index - 1].command.timestamp == timestamp) {
      Undo(buffer);
    }
  }
}
```

We check if `index` is larger than zero before comparing the timestamp of the earlier Command with the one we just undid. And if that was true we call `Undo` again recursively.

```cpp
// command.cpp
void Redo(CommandBuffer *buffer) {
  AnyCommand cmd = buffer->allCommands[buffer->index];
  if(buffer->index == buffer->head) {
    return;
  }
  Execute(cmd);
  buffer->index++;
  int timestamp = cmd.command.timestamp;
  if(buffer->index != buffer->head) {
    AnyCommand nextCommand = buffer->allCommands[buffer->index];
    if(nextCommand.command.timestamp == timestamp) {
      Redo(buffer);
    }
  }
}
```

and for redo we check if we are not already at the very latest Command by comparing `index` to `head` . Then we fetch the next Command and if the timestamps are the same we recursively call `Redo` .
And because our Dear ImGui does not itself change variables but only calls our gameplay functions like `Undo()` and `Redo()` this change already works with our undo-redo-slider !
Now our undo and redo can't put the game in an unnatural state.