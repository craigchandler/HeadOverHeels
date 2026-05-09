# SDL2 Port Plan

Purpose
-------
Track the work needed to replace Allegro with an SDL2 backend while keeping the
game playable at each step.

The practical goal is not to redesign the engine first. The first SDL2 pass
should preserve the current wrapper API, get a working build, and then remove
Allegro-facing names after the port is stable.

Current code shape
------------------
- `source/WrappersAllegro.hpp` is the public graphics/audio/input wrapper used
  by the rest of the game.
- `source/WrappersAllegro.cpp` is the Allegro implementation of that wrapper.
- Most gameplay code calls `allegro::Pict`, `AllegroColor`,
  `allegro::Sample`, and `allegro::ogg::OggPlayer`; it does not call Allegro C
  APIs directly.
- `source/CMakeLists.txt` currently globs all `.cpp` files, then conditionally
  adds backend-specific C sources for Allegro 4 or Allegro 5.
- `source/algif`, `source/alogg`, and `source/loadpng` are Allegro-oriented
  helper code. An SDL2 build should not compile those by default.
- The in-game GUI mainly uses the project's bitmap font in `gamedata/font.png`.
  `allegro::textOut` exists, but it is not the main text rendering path.

Important correction from code review: `WrappersAllegro.hpp` cannot remain
literally unchanged. It currently emits `#error I need either Allegro 4 or
Allegro 5` when neither Allegro backend is selected. The first SDL2 patch must
add an SDL2-compatible branch to this header, or introduce a new neutral wrapper
header and update includes across the codebase.

Recommended migration strategy
------------------------------
Use two phases.

Phase 1: compatibility backend

- Add `USE_SDL2`.
- Keep the current public wrapper names for now:
  - `WrappersAllegro.hpp`
  - `AllegroColor`
  - `allegro::Pict`
  - `allegro::Sample`
  - `allegro::ogg::OggPlayer`
- Add `source/WrappersSDL.cpp`.
- Add an SDL2 branch to `WrappersAllegro.hpp`.
- Ensure an SDL2 build links no Allegro libraries.

Phase 2: naming cleanup

- Rename the wrapper to neutral names after the SDL2 backend is green.
- Candidate names:
  - `Wrappers.hpp` / `WrappersSDL.cpp`
  - `backend::Pict`
  - `backend::Sample`
  - `ColorValue`
- This phase should be mechanical and covered by the SDL2 build.

Do not combine the port and the rename in one step. The wrapper names are ugly,
but keeping them briefly will make the first working SDL2 build much easier to
review.

Header changes
--------------
Add a `USE_SDL2` branch to `source/WrappersAllegro.hpp`.

Prefer opaque project-owned types instead of leaking SDL headers through the
public wrapper header:

```cpp
#elif defined( USE_SDL2 ) && USE_SDL2

typedef unsigned int type_of_allegro_color;

struct HohSdlBitmap;
typedef HohSdlBitmap AllegroBitmap;

struct HohSdlSample;
typedef HohSdlSample AllegroSample;

#else
```

Then define the real structs privately in `WrappersSDL.cpp`, for example:

```cpp
struct HohSdlBitmap {
        SDL_Surface* surface;
        bool isScreen;
};

struct HohSdlSample {
        Mix_Chunk* chunk;
};
```

This keeps SDL out of most translation units and preserves the existing
`Pict::ptr()` and `Sample::ptr()` signatures.

CMake changes
-------------
Top-level `CMakeLists.txt` should gain an SDL2 option and skip all Allegro
discovery when SDL2 is selected:

```cmake
option(USE_SDL2 "Build with SDL2 instead of Allegro" OFF)

if(USE_SDL2 AND USE_ALLEGRO5)
  message(FATAL_ERROR "USE_SDL2 and USE_ALLEGRO5 are mutually exclusive")
endif()

if(USE_SDL2 AND USE_BUNDLED_ALLEGRO)
  message(FATAL_ERROR "USE_SDL2 and USE_BUNDLED_ALLEGRO are mutually exclusive")
endif()

if(USE_SDL2)
  pkg_check_modules(SDL2 REQUIRED IMPORTED_TARGET sdl2)
  pkg_check_modules(SDL2_IMAGE REQUIRED IMPORTED_TARGET SDL2_image)
  pkg_check_modules(SDL2_MIXER REQUIRED IMPORTED_TARGET SDL2_mixer)
  pkg_check_modules(SDL2_TTF QUIET IMPORTED_TARGET SDL2_ttf)
else()
  # existing Allegro discovery and bundled Allegro ExternalProject live here
endif()
```

`source/CMakeLists.txt` should explicitly exclude wrapper implementations from
the initial glob, then add exactly one backend implementation:

```cmake
list(FILTER HOH_CPP_SOURCES EXCLUDE REGEX ".*/WrappersAllegro\\.cpp$")
list(FILTER HOH_CPP_SOURCES EXCLUDE REGEX ".*/WrappersSDL\\.cpp$")

if(USE_SDL2)
  target_sources(headoverheels PRIVATE
    "${CMAKE_CURRENT_SOURCE_DIR}/WrappersSDL.cpp")

  target_compile_definitions(headoverheels PRIVATE USE_SDL2=1)

  target_link_libraries(headoverheels PRIVATE
    PkgConfig::SDL2
    PkgConfig::SDL2_IMAGE
    PkgConfig::SDL2_MIXER)

  if(TARGET PkgConfig::SDL2_TTF)
    target_link_libraries(headoverheels PRIVATE PkgConfig::SDL2_TTF)
  endif()
elseif(USE_ALLEGRO5)
  # existing Allegro 5 path
elseif(TARGET PkgConfig::ALLEGRO4 OR HOH_BUNDLED_ALLEGRO4 OR ALLEGRO4_FOUND)
  # existing Allegro 4 path
endif()
```

For SDL2, do not add `HOH_ALLEGRO4_SOURCES` or `HOH_ALLEGRO5_SOURCES`. Those
lists contain Allegro-specific GIF, Ogg, and PNG helper code.

Dependencies
------------
Required for the SDL2 backend:

- SDL2
- SDL2_image
- SDL2_mixer

Optional:

- SDL2_ttf, only if `allegro::textOut` is implemented with TTF rendering.
- libogg/libvorbis, only if bypassing SDL_mixer for Ogg playback.

Suggested distro packages:

```sh
sudo apt install libsdl2-dev libsdl2-image-dev libsdl2-mixer-dev libsdl2-ttf-dev
sudo dnf install SDL2-devel SDL2_image-devel SDL2_mixer-devel SDL2_ttf-devel
```

Rendering plan
--------------
Start with software surfaces, not an SDL renderer-first design.

Reason: the game performs many direct pixel operations through `Picture` and
`allegro::Pict`: `getPixelAt`, `putPixelAt`, image cloning, color replacement,
masking, sprite construction, floor tile generation, screenshots, and GIF frame
processing. `SDL_Surface` is a closer match to the existing code than
`SDL_Texture`.

Recommended initial design:

- Store every `Pict` as a canonical `SDL_Surface`.
- Use `SDL_PIXELFORMAT_RGBA32` for loaded and created surfaces.
- Use `SDL_MapRGBA` and `SDL_GetRGBA`; do not depend on byte order manually.
- Make `theScreen()` wrap the window surface returned by
  `SDL_GetWindowSurface`.
- Implement `update()` with `SDL_UpdateWindowSurface` after the existing redraw
  timer fires.
- Add a renderer/texture path later only if profiling proves it is needed.

Blit semantics to preserve:

- `bitBlit` is a raw copy. It should not treat transparent pixels specially.
- `drawSprite` is a masked/alpha sprite draw.
- `drawSpriteWithTransparency` applies an extra alpha/transparency value.
- `setWhereToDraw` changes the current target surface for draw calls.
- `mendIntoPict` converts magenta `(255, 0, 255)` in loaded assets into fully
  transparent pixels.

Implementation hints:

```cpp
static SDL_Surface* convertToGameFormat(SDL_Surface* source)
{
        SDL_Surface* converted =
                SDL_ConvertSurfaceFormat(source, SDL_PIXELFORMAT_RGBA32, 0);
        SDL_FreeSurface(source);
        return converted;
}
```

For `bitBlit`, set blend mode to `SDL_BLENDMODE_NONE` before
`SDL_BlitSurface`. For `drawSprite`, set blend mode to `SDL_BLENDMODE_BLEND`.
Restore or set the expected mode every call, because SDL surface blend mode is
stored on the source surface.

Input plan
----------
The game stores key preferences as strings and uses the wrapper to map those
strings to backend scancodes. SDL2 must preserve the current names.

Minimum expected names include:

- Letters `a` through `z`
- Digits `0` through `9`
- `Escape`, `Enter`, `Space`, `Tab`, `Backspace`
- `Left`, `Right`, `Up`, `Down`
- `Insert`, `Delete`, `Home`, `End`, `PageUp`, `PageDown`
- `F1` through `F12`
- Punctuation names and aliases currently accepted by `nameOfKeyToScancode`
- Keypad names such as `Pad 0`, `Pad 8`, `Pad +`, `Pad -`, `Pad Enter`

Implement SDL input with:

- `SDL_PollEvent` to update key state and key queue.
- `SDL_GetKeyboardState` or an internal `std::map<int, bool>` for
  `isKeyPushed`.
- A queued scancode list for `areKeypushesWaiting` and `nextKey`.
- `SDL_GetModState` for Shift, Control, and Alt.
- `releaseKey(name)` clearing the internal key state for that mapped scancode.

Audio plan
----------
Use SDL_mixer first.

`Sample` mapping:

- Store `Mix_Chunk*` in the SDL sample wrapper.
- `Sample::loadFromFile` uses `Mix_LoadWAV`.
- `Sample::play` uses `Mix_PlayChannel(-1, chunk, 0)`.
- `Sample::loop` uses `Mix_PlayChannel(-1, chunk, -1)`.
- Store the returned channel as the current `voice`.
- `Sample::stop` uses `Mix_HaltChannel(voice)`.
- `Sample::isPlaying` uses `Mix_Playing(voice)`.
- Map volume `0..255` to `0..MIX_MAX_VOLUME`.
- Map pan `0..255` to `Mix_SetPanning(channel, left, right)`.

`initAudio("no")` should still be a supported no-audio mode. In that mode, the
wrapper should return success and make sample/music operations no-ops. This
matches the current behavior of falling back to an inert audio driver.

Music/Ogg plan:

- The game currently owns a single `SoundManager::oggPlayer`.
- `Mix_Music*` is acceptable for the first SDL2 implementation because the game
  plays one music stream at a time.
- `OggPlayer::play(file, loop)` should stop any previous music, load
  `Mix_LoadMUS`, and call `Mix_PlayMusic`.
- `OggPlayer::stop()` should halt and free the music.
- `OggPlayer::syncPlayersWithDigitalVolume()` should call `Mix_VolumeMusic`.

If future code needs multiple simultaneous Ogg streams, revisit this and use
`Mix_Chunk` or a custom streaming callback. Do not take on that complexity for
the first port.

GIF animation
-------------
The game uses animated GIFs for `head.gif` and `heels.gif`.

Do not compile `source/algif` or `source/algif5` in SDL2 mode as-is; those
paths depend on Allegro types. Options:

- Use `IMG_LoadAnimation` when the available SDL2_image version supports it.
- Port the existing `algif5` parser to output `SDL_Surface` frames instead of
  `ALLEGRO_BITMAP`.
- As a temporary milestone only, load the first frame and leave animation
  disabled, but mark that as incomplete.

Text rendering
--------------
Do not make SDL2_ttf a hard dependency unless needed.

The main game UI uses the bitmap font code in `source/gui/Font.cpp`, which is
built from `gamedata/font.png` and wrapper blits. Once surfaces and blits work,
most UI text should already work.

`allegro::textOut` can be implemented later with either:

- a small built-in bitmap/debug font, or
- SDL2_ttf if a real TTF font is added or packaged.

Validation checklist
--------------------
The SDL2 backend is not ready until these pass:

- Configure with `-DUSE_SDL2=ON` without finding or linking Allegro.
- Build with `-DUSE_SDL2=ON`.
- `readelf -d build/source/headoverheels` shows SDL libraries and no Allegro
  libraries.
- `cmake --build build --target install-gamedata` stages data.
- Main window opens in windowed mode.
- `gamedata/loading-screen.png` or another known PNG loads and draws.
- Menus render text and sprites.
- Keyboard navigation works, including configured keys from saved preferences.
- `head.gif` and `heels.gif` animate, or the known temporary first-frame
  limitation is documented.
- Sound effects play and stop.
- Music starts, loops, stops, and follows the music volume setting.
- Screenshots/captures still save PNG files.
- Fullscreen/windowed toggles still work.

Developer build command
-----------------------
The local data path still works the same way as the Allegro build:

```sh
cmake -S . -B build \
  -DCMAKE_BUILD_TYPE=Debug \
  -DCMAKE_INSTALL_PREFIX="$PWD/build" \
  -DUSE_SDL2=ON \
  -DUSE_BUNDLED_TINYXML2=ON

cmake --build build --parallel
cmake --build build --target install-gamedata
./build/source/headoverheels
```

First implementation milestones
-------------------------------
1. Add `USE_SDL2` option and CMake source selection.
2. Add the `USE_SDL2` branch in `WrappersAllegro.hpp`.
3. Add `WrappersSDL.cpp` with initialization, teardown, window creation, and
   no-op audio/input stubs so it compiles.
4. Implement `Pict`, PNG loading, raw blits, sprite blits, and `update`.
5. Implement keyboard state and queued key reads.
6. Implement samples with SDL_mixer.
7. Implement Ogg music with `Mix_Music`.
8. Implement GIF animation.
9. Decide whether `textOut` needs SDL2_ttf or a tiny bitmap fallback.
10. Only after the SDL2 build is green, remove Allegro CMake and rename the
    wrapper API.

Known cleanup after SDL2 is working
-----------------------------------
- Rename `WrappersAllegro.hpp` and `setTitleOfAllegroWindow`.
- Replace `AllegroColor` with a backend-neutral color name.
- Remove Allegro tarballs and bundled Allegro CMake.
- Remove Allegro-specific helper folders from the default source tree if they
  are no longer needed.
- Update `README.md` so SDL2 is the default path.
