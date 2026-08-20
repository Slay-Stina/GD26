> **Linux:** This chapter is adapted for Linux.

# Sokoban Programming III

As we often do, it's time to refactor our code. We're going to break the movement logic out into its own function then learn about something called **recursive functions** which we will need to help us push boxes around on the level.

A recursive function is a function that calls itself. It needs one or more exit conditions or else the function might call itself forever.

An example of a small recursive function:

```cpp
int Factorial(int nmbr){
  if(nmbr == 1){
    return 1;
  }
  return nmbr * factorial(nmbr - 1);
}
```

This function will multiply all numbers from `nmbr` to 1 together. If we start with `nmbr = 5` we get the following output: `5 x 4 x 3 x 2 x 1`.

We are going to use a recursive function to help us push boxes around. We do this because before the player character can move into the cell of the box she pushed, we need to know if the box could move.

Here's what we'll need to do:

1. Add a sprite for the box both to our game and Tiled
2. Remove the old movement logic and put it in a new `TryMove` recursive function inside `game.h/cpp`
3. Set up the behaviour for the box so it behaves correctly

Add the box to our list of IDs inside `entity.h`:

```cpp
NONE = 0,
GROUND = 1,
WALL = 2,
PLAYER = 3,
BOX = 4 // <- new
```

Add the `ID::BOX` case to the switch case inside `InitializeBaseBehaviour()`:

```cpp
void InitializeBaseBehaviour(){
  assert(id != ID::NONE);
  switch (id) {
    default:
      SetBehaviour(NONE);
      break;
    case ID::PLAYER:
      SetBehaviour((Behaviour)(CAN_MOVE | IS_PLAYER | RESPOND_TO_INPUT));
      break;
    case ID::BOX: // <- new
      SetBehaviour((Behaviour)CAN_MOVE);
      break;
  }
}
```

In `gameState.h`:

```cpp
Image* player;
Image* box; // <- new
```

In `Initialize()` inside `game.cpp`:

```cpp
data->player = AssetManagement::LoadSprite(data->arena_images, renderer, "player.png");
data->box = AssetManagement::LoadSprite(data->arena_images, renderer, "box.png"); // <- new
```

Load the new level:

```cpp
data->currentLevelIndex = 1; // updated to `1` from `0`
CreateLevel(data->arena_levels, &data->levels[0], "assets/levels/testLevel.tmj");
CreateLevel(data->arena_levels, &data->levels[1], "assets/levels/testLevel_box.tmj"); // <- new
CreateEntities(&data->levels[data->currentLevelIndex], data->arena_entities);
```

In `levelRenderer.cpp`:

```cpp
switch(entity.id){
  case ID::PLAYER:
    img = data->player;
    break;
  case ID::BOX: // <- new
    img = data->box;
    break;
  default:
    img = data->fallback;
    break;
}
```

Add a new function to `game.h`:

```cpp
bool TryMove(Entity* mover, LevelData* level, int xDir, int yDir);
```

In `Update` inside `game.cpp`:

```cpp
for (int i = 0; i < data->GetCurrentLevel()->entityCount; i++) {
  Entity* entity = &data->GetCurrentLevel()->entityBuffer[i];
  if(entity->HasBehaviour((Behaviour)(Behaviour::RESPOND_TO_INPUT | Behaviour::CAN_MOVE))){
    int xChange = 0;
    int yChange = 0;
    if(KeyPressed(SDL_SCANCODE_RIGHT, keys, data->keys_previous)){
      xChange = 1;
    }
    else if(KeyPressed(SDL_SCANCODE_LEFT, keys, data->keys_previous)){
      xChange = -1;
    }
    else if(KeyPressed(SDL_SCANCODE_UP, keys, data->keys_previous)){
      yChange = -1;
    }
    else if(KeyPressed(SDL_SCANCODE_DOWN, keys, data->keys_previous)){
      yChange = 1;
    }
    if(xChange != 0 || yChange != 0){
      TryMove(entity, data->GetCurrentLevel(), xChange, yChange); // <- new
    }
  }
}
```

The `TryMove()` function:

```cpp
bool TryMove(Entity* mover, LevelData* level, int xDir, int yDir){
  if(mover->HasBehaviour(CAN_MOVE) == false){
    return false;
  }
  int test_x = mover->x + xDir;
  int test_y = mover->y + yDir;
  Entity* stepInto_entity = level->GetEntity(test_x, test_y);
  ID stepInto_tile_id = (ID)level->GetCellID(test_x, test_y);
  if(stepInto_entity == nullptr){
    if(stepInto_tile_id == ID::GROUND){
      mover->x = test_x;
      mover->y = test_y;
      return true;
    }
    return false;
  }
  if(stepInto_entity->HasBehaviour(CAN_MOVE)){
    if(TryMove(stepInto_entity, level, xDir, yDir)){
      mover->x = test_x;
      mover->y = test_y;
      return true;
    }
  }
  return false;
}
```

And with this we can now push a box around the level recursively! We're making steady progress towards the basic logic of a Sokoban game!
