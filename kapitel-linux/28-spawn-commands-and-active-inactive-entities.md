# 28 Spawn Commands and active/inactive entities

Currently our game breaks if we move with a character then remove it from our dev menues. We don't get the figure back when we undo/redo. Lets fix that. The issue is that as we undo an action the unit that we spawned doesn't go away. It stays on the board and the undo no longer represent the actual game state we previously had.
We'll need two new Commands . `AddCommand` and `RemoveCommand` .

```cpp
// command.h
enum class CMD_TYPE : uint8_t {
  NONE = 0,
  MOVE = 1,
  ROTATE = 2,
  MODIFY_BEHAVIOUR = 3,
  ADD = 4,
  REMOVE = 5
};
```

```cpp
// command.h
struct AddCommand : Command {
  int x;
  int y;
  ID id;

  AddCommand(int x, int y, ID id) {
    this->x = x;
    this->y = y;
    this->id = id;
    type = CMD_TYPE::ADD;
  }
};
```

Our `AddCommand` is simpler than the `RemoveCommand` as we only need to store the ID of the entity we want to spawn. So the `AddCommand` does not store an Entity itself.

```cpp
// command.h
struct RemoveCommand : Command {
  int x;
  int y;
  Behaviour storedBehaviour;
  ID storedID;

  RemoveCommand(Entity* entity) {
    x = entity->x;
    y = entity->y;
    storedBehaviour = entity->behaviour;
    storedID = entity->id;
    type = CMD_TYPE::REMOVE;
  }
};
```

Here we need to save info about our Entity as we remove it. Lets say that we have petrified an Entity before removing it. To perserve our history we need to store this Behaviour so we can add it back.

As usual we add constructors to both `Add` and `Remove` then add them as variables and constructor parameters to `AnyCommand` .

```cpp
// command.h
union AnyCommand {
  Command command;
  MoveCommand move;
  RotateCommand rotate;
  ModifyBehaviourCommand modify;
  AddCommand add;
  RemoveCommand remove;

  AnyCommand(MoveCommand mov) {
    move = mov;
  };
  AnyCommand(RotateCommand rot) {
    rotate = rot;
  };
  AnyCommand(ModifyBehaviourCommand mod) {
    modify = mod;
  }
  AnyCommand(AddCommand add) {
    this->add = add;
  }
  AnyCommand(RemoveCommand rem) {
    remove = rem;
  }
};
```

Note, due to my honestly pretty substandard naming convention of my parameters I ended up with the same variable name for my `addCommand` and the parameter. forcing me to use `this->` to disambiguate. This is no issue really, but the syntax has a certain smell to it.
Next we'll first refactor a silly mistake in `Command.cpp` before we add our Add/Remove logic to `Execute()` and `Undo()` .

In our switch cases we get the relevant command by writing `CommandType theCommand = cmd.specifiComm` This should always have been a pointer so we don't create any new data. So instead we write `CommandType* theCommand = &cmd.specifiCommand` . We can easily fix this issue that covers a lot of our lines inside `Execute()` and `Undo` by first pressing `v` to enter selection mode then we select the entire code block by moving the caret down over each line. Then we press `s` type `cmd.` and press `enter` . This will put a caret on each of the lines at the exact position where it found the text `cmd.` we use `\` so that the `.` is not escaped and is actually evaluated as text.
Once we have all our cloned Carets we enter insert mode with `i` and delete the `.` and replace it with `->` . With this we've modified 10+ places with just one command. Learning this select and multi-edit command will drastically improve your speed when refactoring.
Now we can add the switch cases to our functions

```cpp
// command.cpp
// inside Execute()
case CMD_TYPE::ADD: {
  AddCommand* add = &cmd.add;
  AddEntity(add->id, add->x, add->y, level);
  break;
}
case CMD_TYPE::REMOVE: {
  RemoveCommand* remove = &cmd.remove;
  RemoveEntity(remove->x, remove->y, level);
  break;
}
```

We encapsulate each case with `{}` then we call our old `AddEntity` and `RemoveEntity` using the parameters we stored in the commands.

```cpp
// command.cpp
// inside Undo()
case CMD_TYPE::ADD: {
  AddCommand* add = &cmd.add;
  RemoveEntity(add->x, add->y, level);
  break;
}
case CMD_TYPE::REMOVE: {
  RemoveCommand* remove = &cmd.remove;
  AddEntity(remove->storedID, remove->x, remove->y, level);
  Entity* entity = GetEntity(level, remove->x, remove->y);
  SetBehaviour(entity, remove->storedBehaviour);
  break;
}
```

As both `RemoveEntity()` `AddEntity()` and `GetEntity()` require that we pass along `LevelData*` we need to change the parameter list of our `Undo()` function to recieve a `LevelData*` this will require us to modify our `command.h` file to add this parameter as well as updating all of our callsites to pass along this variable.

```cpp
// command.h
void Undo(CommandBuffer* buffer, LevelData* level);
```

we call `Undo` from `game.cpp` `dev_gui.cpp` `command.cpp` so those three callsites are where you will need to add and pass along the `LevelData*` parameter.
Now in our `levelEditor.h/.cpp` We will be changing from adding/removing our Entities by calling those functions directly and instead creating then pushing our new commands to do the same actions. To push our commands we need to pass along our `commandBuffer` . To do this we need to update our parameters inside `levelEditor.h` to supply it.

```cpp
// levelEditor.h
void PlaceObject(const int x, const int y, Editor* editor, LevelData* level, CommandBuffer* commandbuffer);
void Update(Editor* editor, Input* input, LevelData* level, CommandBuffer* buffer);
```

Then inside `levelEditor.cpp` we can make the necessary changes

```cpp
// levelEditor.cpp
void PlaceObject(const int x, const int y, Editor* editor, LevelData* level, CommandBuffer* commandBuffer) {
  if(editor->object_to_place_id == ID::GROUND || editor->object_to_place_id == ID::WALL) {
    level->cells[y * level->w + x] = (int)editor->object_to_place_id;
  }
  else {
    // AddEntity(editor->object_to_place_id, x, y, level); // old
    AddCommand add(x, y, editor->object_to_place_id);
    Push(commandBuffer, add, level);
  }
}
```

Then we update our callsite for `RemoveEntity()`

```cpp
// levelEditor.cpp
void Update(Editor* editor, Input* input, LevelData* level, CommandBuffer* buffer) {
  if(MousePressed(input, MouseButtons::LEFT)) {
    // code hidden for clarity
  }
  else if(MousePressed(input, MouseButtons::RIGHT)) {
    if(camera::GetIsPointInsideGrid(input->mouse_x, input->mouse_y, level)) {
      int x;
      int y;
      camera::WorldToGrid(input->mouse_x, input->mouse_y, &x, &y, level);
      Entity* entity = GetEntity(level, x, y);
      if(entity == nullptr) {
        return;
      }
      RemoveCommand remove(entity);
      Push(buffer, remove, level);
    }
  }
}
```

Now our history works as intended with our add/remove. To clarify why this was important to do now. We already were and will continue to test our game by making temporary levels using our `levelEditor`. It will be extremely bothersome to have our history malfunction and cause issues that we might confuse with mistakes in newly written code. That's why we make sure to squash this bug right away.
