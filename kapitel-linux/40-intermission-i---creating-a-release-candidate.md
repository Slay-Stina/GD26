# 40 Intermission I - Creating a release candidate

Sometime, not very likely for this project, we will want to be able to collect only the files we want to ship to our consumers. Our cache folder, ninja output and cmake files that are generated alongside our shared library and executable are not something we should ship to our consumers.
We can increase the capabilities of our `cmakelists.txt` and our `cmakepresets.json` to give us access to a new parameter `--install` that we can call when compliling our project.
the `--install` parameter tells our compiler to run the `install()` function in our `cmakelists.txt` , and that being responsible for copying over files into a specified directory. The `install()` function is used to tell cmake what files to consider for an install-version of our program as opposed to a debug version.

```cmake
install(TARGETS ${PROJECT_NAME} DESTINATION .)
install(TARGETS ${DLL_NAME} RUNTIME DESTINATION .)
install(FILES ${DLL_FILES} DESTINATION .)
install(DIRECTORY ${CMAKE_BINARY_DIR}/assets DESTINATION .)
```

> [!NOTE]
> On Linux, `install(TARGETS ... RUNTIME DESTINATION .)` will place the executable and shared library (`.so`) files correctly. The `DLL_FILES` variable should reference the `.so` files you need to ship (e.g., `libSDL3.so`, `libfmod.so`). You may need to set `LD_LIBRARY_PATH` or use `rpath` so the executable can find the `.so` files at runtime.

We're adding the following four `install()` function calls at the very bottom of our `cmakelists.txt` . These functions only run if the `--install` parameter has been passed to the compiler. Meaning that if we run `cmake --build` as we usually do, these won't fire.
`TARGETS` are references to things that Cmake builds, in this case our executable and shared library are added. We also add the `RUNTIME` parameter to `${DLL_NAME}` so cmake doesn't also add the `"nameofgame"_game.a` the `.a` files is only used during linking to give access to the contents of the shared library once linking is finished this is not a file that is needed. `DESTINATION .` (note the '.') means that the place where the files will show up is at the same folder that we will specify when calling `--install` from a newly created function inside our `~/.bashrc`
`FILES` are any files found on disk not created during compilation, in this case we fetch `${DLL_FILES}` which are the collection of shared libraries (.so files) we specified earlier in our `cmakelists.txt` .
`DIRECTORY` tells the `install()` function to copy an entire folder. We have to specify the path to it using `CMAKE_BINARY_DIR` a built in path variable that points to our build folder. We then add `/assets` to arrive on the assets folder.
In our `CMakepresets.json` we will be adding a new `configurePresets` and a new `buildPresets` . We do this by putting a comma ',' at the end of the square brackets for the default presets then copying the entire preset block and changing the contents inside.

```json
{
  "version": 10,
  "configurePresets": [
    {
      "name": "default",
      "generator": "Ninja",
      "binaryDir": "${sourceDir}/build",
      "cacheVariables": {
        "CMAKE_BUILD_TYPE": "Debug",
        "CMAKE_CXX_COMPILER": "clang++"
      }
    },
    {
      "name": "release",
      "generator": "Ninja",
      "binaryDir": "${sourceDir}/build",
      "cacheVariables": {
        "CMAKE_BUILD_TYPE": "Release",
        "CMAKE_CXX_COMPILER": "clang++",
        "CMAKE_INTERPROCEDURAL_OPTIMIZATION": "TRUE"
      }
    }
  ],
  "buildPresets": [
    {
      "name": "default",
      "configurePreset": "default"
    },
    {
      "name": "release",
      "configurePreset": "release"
    }
  ]
}
```

Note: previously we had our `version` set to 3 . The install of cmake that we grabbed at the beginning of the course has support for (at least) version 10 . This update is not strictly necessary but small performance increases are likely to have occured between version 3 and 10.
our release preset still uses `/build` as the `binaryDir` . This is on purpose as we want to first build to our build folder then use our `install()` to copy over the relevant stuff.

```json
"CMAKE_BUILD_TYPE": "Release"
```

This strips the debugging logic from our project, meaning that we will lose the ability to meaningfully check its behaviour in for example a debugger. But doing this will decrease the final program in size and make it run faster (just like that)

```json
"CMAKE_INTERPROCEDURAL_OPTIMIZATION": "TRUE"
```

This one liner tells the compiler to perform optimization on our .cpp files, looking at all of them instead of individually. This will help boost performance with a tradeoff of making compilation slower. So for a release build it is perfect. We don't need the performance boost when we are building debug versions but would happily double our compile time if we can get a 5-15% performance boost for the consumer when building our release candidate.

```json
"buildPresets": [
  {
    "name": "default",
    "configurePreset": "default"
  },
  {
    "name": "release",
    "configurePreset": "release"
  }
]
```

We also need a `buildPreset` so we can tell which `configurePreset` should be used when calling cmake from our bash build function
lastly we need to create our `release()` function inside `~/.bashrc`

> [!NOTE]
> Add the following to your `~/.bashrc` or `~/.zshrc` instead of `$profile`. You'll also need the `goto` and `GetConfig` helper functions if you use them.

```bash
function release() {
  local project=$1
  if [ -z "$project" ]; then
    echo "Usage: release <project>"
    return 1
  fi
  cd "$HOME/Projects/$project" || return 1
  local sourceDir="$HOME/Projects/$project"
  local buildDir="$sourceDir/build"
  local releaseDir="$sourceDir/release"
  echo "begin creating release candidate..."
  rm -rf "$buildDir" 2>/dev/null
  rm -rf "$releaseDir" 2>/dev/null
  cmake --preset "release"
  if [ $? -ne 0 ]; then
    echo "fetching cmake preset failed - aborting."
    return 1
  fi
  cmake --build --preset "release"
  if [ $? -ne 0 ]; then
    echo "build failed - aborting."
    return 1
  fi
  cmake --install build --prefix "$releaseDir"
  if [ $? -ne 0 ]; then
    echo "install commands failed - aborting."
    return 1
  fi
  echo "release candidate successfully created."
}
```

we jump to our project directory then store paths into our temporary cache Variables

```bash
local sourceDir="$HOME/Projects/$project"
local buildDir="$sourceDir/build"
local releaseDir="$sourceDir/release"
```

```bash
rm -rf "$buildDir" 2>/dev/null
rm -rf "$releaseDir" 2>/dev/null
```

we remove everything from our build and release folders so we are sure that every file is brand new and nothing is lying around and causing issues

```bash
cmake --preset "release"
if [ $? -ne 0 ]; then
  echo "fetching cmake preset failed - aborting."
  return 1
fi
```

we tell cmake to use our new release `buildPresets` . And if that function exited with an exit code that was not equal `-ne` to 0 we stop the rest of the function as we have failed to load our preset.

```bash
cmake --build --preset "release"
if [ $? -ne 0 ]; then
  echo "build failed - aborting."
  return 1
fi
```

we then build our project putting everything into our `/build` folder. And once again, if we failed to build, then we don't move on to the next part of the function

```bash
cmake --install build --prefix "$releaseDir"
if [ $? -ne 0 ]; then
  echo "install commands failed - aborting."
  return 1
fi
```

then we do the new part, we tell cmake to `--install` from our build folder and the `--prefix` parameter tells cmake to place the installed content inside the specified path.
the `install()` functions in `cmakelists.txt` , our new `buildPresets` and `configurePresets` in `cmakePresets.json` and this new `release` function are all the things we need to create our optimized and stripped down release candidate.