# 34 FMOD and Audio

We'll be implementing audio. We are just going to dip our toes into audio systems programming which is really a whole discipline in and of itself. And just as with VFX programming and animation systems there are a lot of terms and concepts that someone has to understand in order to fully grasp the boilerplate.
We'll be working with a middleware called FMOD . This is an industry standard tool that many (many) of the largest commercial titles use. There are two ways of using FMOD

1. FMOD core
2. FMOD studio

We'll be using FMOD core even though FMOD studio is far an away the more common way of working with FMOD. We do this because FMOD studio is its own software with a bunch of buttons and menues that do not fit into the scope of this course. FMOD core is just a few .h files and .lib files that we can hook into to produce audio for our games.
To download FMOD core you need to create a free account over at [https://www.fmod.com/download#fmodengine](https://www.fmod.com/download#fmodengine) . Once that is done you should have downloaded the FMOD Engine installer. Once that is loaded and run you'll find that included in the install directory `FMOD SoundSystem\FMOD Studio API Windows\api\core` is the core folder with an `inc` and a `lib` folder. Copy over the contents of each into your project in the project folder with the same name. For my `include` folder I decided to put all fmod .h files into a `FMOD` subdirectory.
In the `lib` folder, look for the `x64` folder inside. In my own projects `lib` folder I've opted for putting the FMOD .lib files into a `FMOD` subdirectory as well.
Audio is a pretty difficult part of game development. and there are many pitfalls and ways to make your life harder. But FMOD honestly does a great job of having minimal boilerplate and handling most of our issues for you.
You can unzip the `chapter 35 assets.zip` and replace the old content inside your `assets` folder with it.
lets set up `audioSystem.h` that will manage FMOD

```cpp
// audioSystem.h
#pragma once
#include "FMOD/fmod_common.h"

namespace Memory {
  struct Arena;
}

enum class SFX_ID {
  JUMP,
  COUNT
};

struct SoundDataEntry {
  SFX_ID id;
  const char* path;
};

struct AudioSystem {
  bool initialized;
  void* fmod_memory;
  FMOD_SYSTEM* sound_system;
  static const int CHANNEL_COUNT = 32;
  FMOD_CHANNEL* channels[CHANNEL_COUNT];
  FMOD_SOUND* soundEffects[(int)SFX_ID::COUNT];
};

extern AudioSystem* g_audioSystem;

void PlaySFX(SFX_ID id, float volume = 1);
void InitializeAudioSystem(AudioSystem* audio, Memory::Arena* arena_main);
void UpdateAudio(AudioSystem* audio);

namespace AssetManagement {
  void LoadAllSFX(AudioSystem* audioSystem);
}
```

We're `#include`ing `fmod_common.h` making sure to have our path to the .h include our subdirectory created earlier. We then forward declare `Memory::Arena` .
`SoundDataEntry` will be used just like `SpriteDataEntry` in our `audioSystem.cpp` later. We'll use it to load our sound files.
`AudioSystem` is our main struct responsible for holding everything we need to play audio. We also do something slighly different with our arrays. As the size of these arrays will not change we can specify their size in advance. This has the effect that when we add `AudioSystem` to `GameData` we do not have to allocate our arrays in our memory arena ourselves, as this is done for us.
We'll be giving FMOD a section of our memory to work with, a pointer to this memory is held in our `void* fmod_memory` .
`FMOD_SYSTEM* sound_system` is the struct that we initialize and always pass in when we work with our audio. This holds all the relevant information that FMOD needs to function.

a channel or voice is an audio thread responsible for streaming and pushing audio to our sound card. We can not have an infinite amount of these as this will throttle our audio thread. Though I am no audio programmer I've found that 16-32 channels works for our needs. When a source of sound is going to be played we must provide it with a channel to take over. This means that we can't have more than 32 things playing at once.
`FMOD_SOUND` holds our actual audio file in memory. FMOD reads bytes from this and pushes that through the relevant channel. We make this array have the size of `SFX_ID::COUNT` as that makes sure that as along as `COUNT` is the last enum in the list we have allocated enough array elements for all sound effects.
Audio tends to be the type of system that easily worms its way throughout our codebase. To avoid having to pass `AudioSystem` everywhere we're making a global pointer to an `AudioSystem` available for every file that includes `audioSystem.h` . The `extern` keyword promises that the actual pointer will be found during compilation from another file. We'll keep this global pointer synced to the `AudioSystem` we'll put inside `GameData` so that any file can point to it and use our audio system. every program has by its nature some global state.
We'll write a pointer with this exact name again inside `AudioSystem.cpp` as this is the actual variable. our `extern` in our .h file is just a way of accessing this by `#include` .
our three functions do exactly what we expect. Plays sound effects, initializes everything and keeps it updated.

in the `AssetManagement` namespace we add a `LoadAllSFX()` .

We'll call our `InitializeAudioSystem` and `LoadAllSFX()` from our `Initialize()` function inside `game.cpp` . But first we must have a `AudioSystem` inside `GameData` to actually work with

```cpp
// gameState.h
struct GameData {
  SCENE_TYPES scene_current;
  SCENE_TYPES scene_previous;
  Scenes scenes;
  Transition transition;
  EditorData editor_data;
  Input input;
  Sprite* spriteBuffer;
  Tileset* tilesetBuffer;
  AudioSystem audio;
  // other variables hidden for clarity
};
```

then inside `game.cpp`

```cpp
// game.cpp
void Initialize(GameData* data, SDL_Window* window, SDL_Renderer* renderer) {
  DEV::Initialize(window, renderer);
  InitializeAudioSystem(&data->audio, data->arena_main);
  AssetManagement::LoadAllSFX(&data->audio);
  AssetManagement::LoadAllSprites(data->spriteBuffer, renderer);
  data->imGui_context = ImGui::GetCurrentContext();
  AssetManagement::LoadAllTilesets(data->tilesetBuffer, data->arena_images);
  SDL_Texture* blackfade = GetSprite(SPRITE_ID::black_1x1, data->spriteBuffer)->texture;
  SDL_SetTextureBlendMode(blackfade, SDL_BLENDMODE_BLEND);
  InitializeGame(&data->scenes.gameplay, data->arena_levels, data->tilesetBuffer);
  InitializeMenu(&data->scenes.mainMenu, data->spriteBuffer, data->arena_main);
  ChangeScene(data, SCENE_TYPES::MAINMENU);
}
```

we'll need to allocate memory for our `AudioSystem`. This starts with deciding on the amount of memory to give it.

```cpp
// common.h
constexpr size_t GAME_MEMORY_ALLOWANCE = MEGABYTES(14);
constexpr size_t AUDIO_MEMORY_ALLOWANCE = MEGABYTES(5);
```

Just make sure the audio segment is smaller than the game memory as we're creating a subarena for the audio from the main memory.
Lets look at our implementation in `audioSystem.cpp`

```cpp
// audioSystem.cpp
// part 1
#include "audioSystem.h"
#include "FMOD/fmod.h"
#include "arena.h"
#include "common.h"
#include <cassert>

AudioSystem* g_audioSystem;

static const SoundDataEntry all_sound_data[] = {
  {SFX_ID::FALLBACK, "assets/audio/sfx/fallback.wav"},
  {SFX_ID::JUMP, "assets/audio/sfx/fallback.wav"},
};
```

here we create our same `g_audioSystem` . Ensure it has the exact same name. the `g_` is called hungarian notation and signifies that the name of the variable tells us about its type. In this case `global_` .
We then set up a compile time known array of `SoundDataEntry` . For now we only have one `::JUMP` and our fallback that both link to `fallback.wav` . But later we'll add more.

```cpp
// audioSystem.cpp
// part 2
void InitializeAudioSystem(AudioSystem* audio, Memory::Arena* arena_main) {
  assert(audio->initialized == false);
  size_t memory_size = AUDIO_MEMORY_ALLOWANCE;
  audio->fmod_memory = Memory::CreateSubArena(arena_main, memory_size);
  FMOD_RESULT memory_init_ok = FMOD_Memory_Initialize(audio->fmod_memory, memory_size, nullptr, nullptr, nullptr, FMOD_MEMORY_ALL);
  assert(memory_init_ok == FMOD_OK);
  FMOD_RESULT system_creation_ok = FMOD_System_Create(&audio->sound_system, FMOD_VERSION);
  assert(system_creation_ok == FMOD_OK);
  FMOD_RESULT system_init_ok = FMOD_System_Init(audio->sound_system, AudioSystem::CHANNEL_COUNT, FMOD_INIT_NORMAL, nullptr);
  assert(system_init_ok == FMOD_OK);
  g_audioSystem = audio;
  audio->initialized = true;
}
```

Every FMOD function returns a `FMOD_RESULT` that tells us what happened during the function call. We can check this against `FMOD_OK` to make sure that it executed succesfully. We use four asserts to help us catch errors with our audio initialization. First we create a SubArena from `arena_main` and assign the first memory address to `audio->fmod_memory` . This is of the type `void*` it is just a place in memory, not a pointer to a variable/struct. We then call `FMOD_Memory_Initialize` and pass along the point in memory and the `memory_size` . If everything went well we can continue.
We then create the `FMOD_SYSTEM` that we point to using `audio->sound_system` .
finally we intiailize the `FMOD_SYSTEM` letting FMOD handle the setup behind the scenes. With all of this done we can assign our global pointer to point at our `audio` and flip our `initialized` flag to `true` .
That is the entire setup to have FMOD in our project. The next steps are about loading our sound effects, finding a free channel and actually playing audio

```cpp
// audioSystem.cpp
// part 3
const int NOT_FOUND = -1;

int GetAvailableChannelIndex(AudioSystem* audio) {
  for (int i = 0; i < AudioSystem::CHANNEL_COUNT; i++) {
    FMOD_CHANNEL* channel = audio->channels[i];
    if(channel == nullptr) {
      return i;
    }
    FMOD_BOOL is_playing = false;
    FMOD_Channel_IsPlaying(channel, &is_playing);
    if(is_playing == false) {
      return i;
    }
  }
  return NOT_FOUND;
}
```

This is a function not found in our `audioSystem.h` as this is just a helper function that only `audioSystem.cpp` needs to be able to call. We can not return a pointer to a `FMOD_CHANNEL` directly because before we have ever used a channel it is actually `nullptr` . Therefore we use a sentinel value of `-1` aka `NOT_FOUND` to represent not having an available channel. This function loops over all channels and checks through `FMOD_Channel_IsPlaying` if the channel is available. If yes it returns that index.
We could make a more advanced system later where audio has different priorities and if no channel is found it just grabs the channel containing the audio with the lowest priority. But for now we're going to ignore this.

```cpp
// audioSystem.cpp
// Part 4
void PlaySFX(SFX_ID id, float volume) {
  assert(id != SFX_ID::COUNT);
  FMOD_SOUND* sfx = g_audioSystem->soundEffects[(int)id];
  if(sfx == nullptr) {
    assert(id != SFX_ID::FALLBACK);
    PlaySFX(SFX_ID::FALLBACK);
  }
  int channel_index = GetAvailableChannelIndex(g_audioSystem);
  if(channel_index == NOT_FOUND) {
    return;
  }
  FMOD_CHANNEL** channel_slot = &g_audioSystem->channels[channel_index];
  FMOD_System_PlaySound(g_audioSystem->sound_system, sfx, nullptr, false, channel_slot);
  FMOD_Channel_SetVolume(*channel_slot, volume);
}
```

Playing sound effects becomes a very simple setup. We find the appropriate `FMOD_SOUND` from our `soundeffects` array, making sure it exists, otherwise we call `FALLBACK` instead. We then look for an available channel . If no such channel is found we can return early. Otherwise we fetch a pointer to the channel pointer from the `channel_index` and call `FMOD_System_PlaySound()` this connects a `FMOD_SOUND` to a `FMOD_CHANNEL` and tells `FMOD_SYSTEM` to begin streaming data from one to the other and into our computers sound card (in simplified terms). We also allow for a volume float to be passed in. calling `SetVolume` on our channel tells it how loud the sfx should be.
FMOD does not automatically keep itself working. We need to make sure we update it each frame.

```cpp
// audioSystem.cpp
// part 5
void Update(AudioSystem* audio) {
  if(g_audioSystem == nullptr || g_audioSystem != audio) {
    g_audioSystem = audio;
  }
  assert(audio->initialized);
  FMOD_System_Update(audio->sound_system);
}
```

this function makes sure that `g_audioSystem` always point to our `audio`. We need this to ensure that nothing breaks after a hot-reload as `g_audioSystem` lives as part of our .DLL .
lastly we call `FMOD_System_Update` and with that our FMOD system will continue to function.
With all of these function calls you should refer to FMODs documentation or ask an LLM or quick-start guide online!
Lets look at loading our SFX now

```cpp
// audioSystem.cpp
// part 6
namespace AssetManagement {
  void LoadAllSFX(AudioSystem* audioSystem) {
    for (const SoundDataEntry& sound_data : all_sound_data) {
      FMOD_RESULT sound_created_ok = FMOD_System_CreateSound(audioSystem->sound_system, sound_data.path, FMOD_DEFAULT, nullptr, &audioSystem->soundEffects[(int)sound_data.id]);
      assert(sound_created_ok == FMOD_OK);
    }
  }
}
```

We loop over all `SoundDataEntry` from `all_sound_data` . to not have to worry about for-looping to a `_amount` variable we can use the other syntax for a for-loop that is "more modern" in c++. this will grab our array and intuit the size itself. And by putting `&` after the variable type we are using it by reference instead of by value or by pointer meaning that the compiler does the heavy lifting and goes and grabs it for us.
as always, knowing why some parameters are `nullptr` and what they do is part of reading documentation, quick-start guides or discussing the implementation with LLMs.
Now to test our audio we can call `PlaySFX()` from our `UpdateGame()` . We'll check the result of `TryMove()` and if true we play `::JUMP`

```cpp
// game.cpp
if(!IsActing(entity)) {
  bool moved = TryMove(entity, level, gameplay->commandBuffer, xDir, yDir, entity->strength);
  if(moved) {
    PlaySFX(SFX_ID::JUMP);
  }
  gameplay->commandBuffer->timestamp += 1;
  gameplay->input_buffer_read_count++;
}
```

Now we can build our game and hear our fallback sound effect each time the player takes a step!