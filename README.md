# Head over Heels

The free and open source remake of the Jon Ritman and Bernie Drummond’s game

## The Story

This repository originated with version 1.0.1 by *Jorge Rodríguez Santos* aka *helmantika*
which I found on http://www.headoverheels2.com/

*Jorge Rodríguez Santos* wrote :

> In summer 2001 I decided to achieve the goal of programming a remake of Head over Heels. The first stage arrived two years later, when I published a beta version of it. That experience was a really extraordinary one; above all, because of all the support I received from so many people that, in one way or another, collaborated to make possible that the project saw finally the light. Although the game credits are a good proof of it, I would specifically mention in these pages the participation, as the graphic designer, of Davit Masiá. Without his work, I could never complete my own one.

> Unfortunately, I couldn’t develop that beta into an stable version because, for almost three years, I went through personal circumstances I prefer to forget. However, once I was able to recover my free time and feeling, as I felt, that I had a thorn in my side for not having finished the «remake», I decided to return to the project. Ignacio Pérez Gil had released the 0.3 version of his isometric engine Isomot, and I took it as the basis from which I reconsidered the work. In 2007, with most of the code already finished, I decided to reopen the forum.

> These last two years have been ones of hard work, but a very gratifying one for the same reason the first part was: the unconditional support from several persons that have always paid attention to the evolution of the game. I’d like to thank, from my heart, to Santiago Acha, Xavi Colomé, Paco Santoyo and Hendrik Bezuidenhout their support and the good moments they have given me.

I found it in the fall of 2016 and saw that the sources lack a (sup)port for Mac OS X at all, and in particular for OS X on PowerPC,
therefore I began this project on GitHub which eventually evolved into what it is now.

However, till the end of 2023 this project went almost unnoticed.
Neither Jorge nor anyone else from the original team was ever participated in any further development and never wrote a single line about what’s goin’ on or was done here.
https://osgameclones.com/ was perhaps the only lone site on which a link to this repo ever appeared.
In the spring of 2019, I’ve lost all my interest and almost forgot about this project for 4.5 years...

...Until the fall of 2023, when François @kiwifb Bissy’s contribution awakened my care.
And then, *finalmente*, Jorge came to this project.
At the end of 2023 we were sharing various stories via email.
His new site features [the article about this project](https://hoh.helmantika.com/2024/04/02/el-remake-de-douglas/)

## The Present & The Future

I reworked almost every piece of the Jorge’s code, dealed around many gotchas and added many new features like

* animated movin’n‘binkin dudes in the menus
* transitions between the user interface menus
* the new black & white set of graphics
* ... with the speccy-like coloring of rooms
* the room entry tunes
* room miniatures
* the camera optionally follows the active character
* accelerated falling
* support for various screen sizes not only 640 x 480
* support for both the allegro 5 and allegro 4 libraries
* the many easy-to-enable cheats :O

But yep, all of this is still quite “raw” for now and really needs some more love.
Any playing, testing, issues found, spreading a word about this project somewhere, and other contributions are welcome.
Especially patches, these are **very much** welcome ;)

## Building

Prerequisites
-------------

Debian/Ubuntu:

```sh
sudo apt update
sudo apt install build-essential cmake pkg-config \
  zlib1g-dev libpng-dev libogg-dev libvorbis-dev libtinyxml2-dev \
  libx11-dev libxext-dev libxcursor-dev libxpm-dev libxxf86vm-dev
```

Arch Linux:

```sh
sudo pacman -Syu base-devel cmake pkgconf zlib libpng libogg libvorbis tinyxml2 \
  libx11 libxext libxcursor libxpm libxxf86vm
```

Allegro 4 packages vary by distribution. Prefer a system development package
when one is available. If not, use `-DUSE_BUNDLED_ALLEGRO=ON`.

Developer Build
---------------

This keeps everything under `build/`. The key is configuring the install prefix
to the build directory before building the executable, because that prefix is
compiled into the game as its data path:

```sh
cmake -S . -B build \
  -DCMAKE_BUILD_TYPE=Debug \
  -DCMAKE_INSTALL_PREFIX="$PWD/build" \
  -DUSE_BUNDLED_TINYXML2=ON \
  -DUSE_BUNDLED_ALLEGRO=ON

cmake --build build --parallel
cmake --build build --target install-gamedata
./build/source/headoverheels
```

That produces this local layout:

```text
build/source/headoverheels
build/share/headoverheels/...
```

If you omit `-DCMAKE_INSTALL_PREFIX`, CMake keeps its default prefix of
`/usr/local`, and the compiled data path is `/usr/local/share/headoverheels`.
The `install-gamedata` target will still stage files in `build/share` by
default, but the executable will not look there unless the install prefix was
set to `build` before it was compiled.

Normal Install
--------------

Choose the install prefix at configure time:

```sh
cmake -S . -B build \
  -DCMAKE_BUILD_TYPE=Release \
  -DCMAKE_INSTALL_PREFIX="$HOME/.local"

cmake --build build --parallel
cmake --install build
```

The installed layout is:

```text
<prefix>/bin/headoverheels
<prefix>/share/headoverheels/...
```

At runtime the game uses the configured data directory. For a normal install,
that is `<prefix>/share/headoverheels`.

Useful Targets
--------------

Install only gamedata into the build tree:

```sh
cmake --build build --target install-gamedata
```

For this staged data to be used by `build/source/headoverheels`, configure with
`-DCMAKE_INSTALL_PREFIX="$PWD/build"` and rebuild the executable.

Override the data-only staging prefix:

```sh
cmake -S . -B build -DHOH_GAMEDATA_STAGE_PREFIX="$PWD/some-prefix"
cmake --build build --target install-gamedata
```

CMake Options
-------------

- `-DUSE_SYSTEM_ZLIB=ON|OFF`: prefer system zlib, default `ON`.
- `-DUSE_SYSTEM_LIBPNG=ON|OFF`: prefer system libpng, default `ON`.
- `-DUSE_SYSTEM_OGG=ON|OFF`: prefer system libogg, default `ON`.
- `-DUSE_SYSTEM_VORBIS=ON|OFF`: prefer system libvorbis, default `ON`.
- `-DUSE_SYSTEM_TINYXML2=ON|OFF`: prefer system tinyxml2, default `ON`.
- `-DUSE_BUNDLED_TINYXML2=ON`: fetch and build tinyxml2 11.0.0 privately if
  system tinyxml2 is not found.
- `-DUSE_BUNDLED_ALLEGRO=ON`: force the bundled Allegro 4 path, extract
  `external/allegro/allegro-4.4.3.1.tar.gz`, and link it as a static library.

Notes
-----

- The bundled Allegro path is mainly for developer convenience and unsupported
  distributions. System Allegro packages are still preferred for distribution
  packaging.
- Bundled tinyxml2 is linked into the game build but is not installed as a
  separate library by this project.
- If CMake reports a missing dependency, either install the corresponding
  development package or enable the bundled option when one exists.
