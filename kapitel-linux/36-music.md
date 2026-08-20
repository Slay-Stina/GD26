# 36 Music

With FMOD core implemented it's easy to set up music playback. But because a music track can be very large and a game might have a lot of tracks playing throughout the game we can't really pre-load all music as we do for our sound effects. Instead we'll be streaming the music a few bytes at a time.

```cpp
// audioSystem.h
enum class SONG_ID {
  NONE,
  THEME
};

struct AudioSystem {
  bool initialized;
  void* fmod_memory;
  FMOD_SYSTEM* sound_system;
  static const int CHANNEL_COUNT = 32;
  FMOD_CHANNEL* channels[CHANNEL_COUNT];
  FMOD_SOUND* soundEffects[(int)SFX_ID::COUNT];
  SONG_ID song_id;
  FMOD_SOUND* song;
  FMOD_CHANNEL* song_channel;
};

void PlaySong(SONG_ID id);
```

We create a new enum to hold all of our songs, then we create a separate `FMOD_SOUND` and `FMOD_CHANNEL` that we'll use exclusively for music. a new function `PlaySong()` will be the only function call we'll need.

```cpp
// audioSystem.cpp
void PlaySong(SONG_ID id) {
  g_audioSystem->song_id = id;
  if(g_audioSystem->song != nullptr) {
    FMOD_Channel_Stop(g_audioSystem->song_channel);
    FMOD_Sound_Release(g_audioSystem->song);
  }
  FMOD_SYSTEM* system = g_audioSystem->sound_system;
  const char* song_name;
  switch (id) {
    case SONG_ID::THEME:
      song_name = "assets/audio/music/hellofatime.mp3";
      break;
    case SONG_ID::NONE:
      break;
  }
  FMOD_System_CreateStream(system, song_name, FMOD_LOOP_NORMAL, nullptr, &g_audioSystem->song);
  int FOREVER = -1;
  FMOD_Sound_SetLoopCount(g_audioSystem->song, FOREVER);
  FMOD_System_PlaySound(system, g_audioSystem->song, nullptr, false, &g_audioSystem->song_channel);
}
```

We check if we have something playing, in that case we stop the channel and call `_Release()` . We grab our `FMOD_SYSTEM` from `g_audioSystem` and instead of our more elaborate `SoundDataEntry` we just pick the `song_name` directly from a switch case. We do this because even a huge game rarely has more than 30-50 songs. and we currently have 1...
We start a Stream that will pull in our .mp3 piece by piece as needed. Then we set the Loop Count to `-1` which FMOD interprets as keep looping forever. we also use `FMOD_LOOP_NORMAL` instead of `FMOD_DEFAULT` when creating the stream to let FMOD know that we want this stream to loop once finished. `SetLoopCount` just controls how many times it's allowed to loop.
Then we call `PlaySound()` as normal.
In `Game.cpp` we call our play function

```cpp
// game.cpp
void Initialize(GameData* data, SDL_Window* window, SDL_Renderer* renderer) {
  *data->ticks_total = 0;
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
  PlaySong(SONG_ID::THEME);
  ChangeScene(data, SCENE_TYPES::MAINMENU);
}
```

That's it! Now we can play music by streaming our audio!