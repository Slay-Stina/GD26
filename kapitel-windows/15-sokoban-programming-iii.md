# Sokoban Programming III

As we often do, it's time to refactor our code. We're going to break the movement logic out into its own function then learn about something called **recursive functions** which we will need to help us push boxes around on the level.

A recursive function is a function that calls itself. It needs one or more exit conditions or else the function might call itself forever. This would completely stall our program and eventually it would crash.

A recursive function is really just a looping function where we take some state and pass it along to the latest iteration of the function. We can always turn any recursive function into a for-loop (and vice-versa) but once the concept of a recursive function is understood it has a lot more visual clarity.

An example of a small recursive function:

```cpp
int Factorial(int nmbr){
  if(nmbr == 1){
    return 1;
  }
  return nmbr * factorial(nmbr - 1);
}
```

This function will multiply all numbers from `nmbr` to 1 together. If we start with `nmbr = 5` we get the following output: `5 x 4 x 3 x 2 x 1`. What is important to understand is that as long as we don't reach the end of the recursive chain (`nmbr == 1`) then we will keep calling `factorial` until our base case is reached. This means that the first iteration of this function to get resolved is the fifth iteration, where we passed in `nmbr - 1` and number was 2. Then we calculate `nmbr - 1` to be 1 and instead of calling `factorial()` again we just return 1. Now in the earlier iteration we get `return 2 * factorial(2 - 1)` and with the knowledge that we returned 1 we get `return 2 * 1` aka 2. The earlier iteration now becomes `return 3 * 2`, the next `return 4 * 6` and finally we return to the very first iteration with `return 5 * 24`. With no earlier iteration to return back to, we can leave our recursive loop having climbed back up from the deepest iteration, each time taking the result back with us to the previous iteration.

So this function computes 5! (5 factorial) and correctly gets the result 120.

We are going to use a recursive function to help us push boxes around. We do this because before the player character can move into the cell of the box she pushed, we need to know if the box could move. We could check the tile 1 extra space away, but what if we decide we should be able to push a box that in turn pushes another box? Does this not start to sound a bit recursive?

Here's what we'll need to do:

1. Add a sprite for the box both to our game and Tiled
2. Remove the old movement logic and put it in a new `TryMove` recursive function inside `game.h/cpp`
3. Set up the behaviour for the box so it behaves correctly

We can start by adding the box to our list of IDs inside `entity.h`:

```cpp
NONE = 0,
GROUND = 1,
WALL = 2,
PLAYER = 3,
BOX = 4 // <- new
```

Then we add the `ID::BOX` case to the switch case inside the `InitializeBaseBehaviour()` function:

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

We want to have the box be able to move, but should not get moved by us pressing the arrow keys. So we just give it the `CAN_MOVE` behaviour.

In `gameState.h` we add an `Image*` pointer to a box:

```cpp
Image* player;
Image* box; // <- new
```

Then we can load a texture and store the result in our `box` pointer. In our `Initialize()` function inside `game.cpp` we can load it:

```cpp
data->player = AssetManagement::LoadSprite(data->arena_images, renderer, "player.png");
data->box = AssetManagement::LoadSprite(data->arena_images, renderer, "box.png"); // <- new
```

I decided to create a new .TMJ file with a level containing more space to move around, a player and a box. I saved this new .TMJ inside my `assets/levels` folder. Then inside the same `Initialize()` function we load this new level as well as updating `currentLevelIndex` to refer to this new level:

```cpp
data->currentLevelIndex = 1; // updated to `1` from `0`
CreateLevel(data->arena_levels, &data->levels[0], "assets/levels/testLevel.tmj");
CreateLevel(data->arena_levels, &data->levels[1], "assets/levels/testLevel_box.tmj"); // <- new
CreateEntities(&data->levels[data->currentLevelIndex], data->arena_entities);
```

In `levelRenderer.cpp` we select our box `Image*` to be the texture rendered from `RenderEntities()`:

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

Finally we'll add a new function to `game.h`:

```cpp
bool TryMove(Entity* mover, LevelData* level, int xDir, int yDir);
```

This uses the info we can get from `LevelData` as well as the direction we are hoping to move the entity. It returns a `bool` so that different cases can return either `true` or `false`, controlling the recursive loop.

In our `Update` inside `game.cpp` we'll strip out the logic that was responsible for moving our player and instead add a call to our new `TryMove()` function. We'll keep the for-loop going over each entity and the keyboard checks:

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

With this the final step is to write the contents of the `TryMove()` function:

```cpp
bool TryMove(Entity* mover, LevelData* level, int xDir, int yDir){
  if(mover->HasBehaviour(CAN_MOVE) == false){
    return false;
  }
```

First we check if the entity we are trying to move actually is allowed to move based on its behaviour flags.

```cpp
// TryMove() continued
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
```

Then we get the x and y positions that the entity would move into (if they are allowed to move). We check if there is an entity on that spot as well as retrieving the type of tile it is, storing this in `stepInto_entity` and `stepInto_tile_id`. We store it as `ID` requiring us to cast the `uint8_t` that we get from `GetCellID` into `ID`.

Then if there was no entity blocking us (`== nullptr`) and the tile was `GROUND` then we perform the move, updating x and y. With this we have successfully performed a move and can return `true`. Otherwise if the tile was not `GROUND` we can now return `false`.

```cpp
// TryMove() continued
  if(stepInto_entity->HasBehaviour(CAN_MOVE)){
    if(TryMove(stepInto_entity, level, xDir, yDir)){
      mover->x = test_x;
      mover->y = test_y;
      return true;
    }
  }
  return false;
```

Now at this point we know that we are walking into another entity. We can then check if that entity is allowed to move — if not we can return `false`. If it is allowed to move then we recursively call `TryMove()` again, using that entity instead. This will then return either `true` or `false` letting us know if the cell is available to get moved into by our original mover.

And with this we can now push a box around the level recursively! We're making steady progress towards the basic logic of a Sokoban game!
