## Craft

Minecraft clone for Windows, Mac OS X and Linux. Just a few thousand lines of C using modern OpenGL (shaders). Online multiplayer support is included using a Python-based server.

http://www.michaelfogleman.com/craft/

![Screenshot](https://i.imgur.com/SH7wcas.png)

### Features

* Simple but nice looking terrain generation using perlin / simplex noise.
* More than 10 types of blocks and more can be added easily.
* Supports plants (grass, flowers, trees, etc.) and transparency (glass).
* Simple clouds in the sky (they don't move).
* Day / night cycles and a textured sky dome.
* World changes persisted in a sqlite3 database.
* Multiplayer support!

### Download

Mac and Windows binaries are available on the website.

http://www.michaelfogleman.com/craft/

See below to run from source.

### Install Dependencies

#### Mac OS X

Download and install [CMake](http://www.cmake.org/cmake/resources/software.html)
if you don't already have it. You may use [Homebrew](http://brew.sh) to simplify
the installation:

    brew install cmake

#### Linux (Ubuntu)

    sudo apt-get install cmake libglew-dev xorg-dev libcurl4-openssl-dev
    sudo apt-get build-dep glfw

#### Windows

Download and install [CMake](http://www.cmake.org/cmake/resources/software.html)
and [MinGW](http://www.mingw.org/). Add `C:\MinGW\bin` to your `PATH`.

Download and install [cURL](http://curl.haxx.se/download.html) so that
CURL/lib and CURL/include are in your Program Files directory.

Use the following commands in place of the ones described in the next section.

    cmake -G "MinGW Makefiles"
    mingw32-make

### Compile and Run

Once you have the dependencies (see above), run the following commands in your
terminal.

    git clone https://github.com/fogleman/Craft.git
    cd Craft
    cmake .
    make
    ./craft

### Multiplayer

Register for an account!

https://craft.michaelfogleman.com/

#### Client

You can connect to a server with command line arguments...

    ./craft craft.michaelfogleman.com

Or, with the "/online" command in the game itself.

    /online craft.michaelfogleman.com

#### Server

You can run your own server or connect to mine. The server is written in Python
but requires a compiled DLL so it can perform the terrain generation just like
the client.

    gcc -std=c99 -O3 -fPIC -shared -o world -I src -I deps/noise deps/noise/noise.c src/world.c
    python server.py [HOST [PORT]]

### Controls

- WASD to move forward, left, backward, right.
- Space to jump.
- Left Click to destroy a block.
- Right Click or Cmd + Left Click to create a block.
- Ctrl + Right Click to toggle a block as a light source.
- 1-9 to select the block type to create.
- E to cycle through the block types.
- Tab to toggle between walking and flying.
- ZXCVBN to move in exact directions along the XYZ axes.
- Left shift to zoom.
- F to show the scene in orthographic mode.
- O to observe players in the main view.
- P to observe players in the picture-in-picture view.
- T to type text into chat.
- Forward slash (/) to enter a command.
- Backquote (`) to write text on any block (signs).
- Arrow keys emulate mouse movement.
- Enter emulates mouse click.

### Chat Commands

    /goto [NAME]

Teleport to another user.
If NAME is unspecified, a random user is chosen.

    /list

Display a list of connected users.

    /login NAME

Switch to another registered username.
The login server will be re-contacted. The username is case-sensitive.

    /logout

Unauthenticate and become a guest user.
Automatic logins will not occur again until the /login command is re-issued.

    /offline [FILE]

Switch to offline mode.
FILE specifies the save file to use and defaults to "craft".

    /online HOST [PORT]

Connect to the specified server.

    /pq P Q

Teleport to the specified chunk.

    /spawn

Teleport back to the spawn point.

### Screenshot

![Screenshot](https://i.imgur.com/foYz3aN.png)

### Implementation Details

#### Terrain Generation

The terrain is generated using Simplex noise - a deterministic noise function seeded based on position. So the world will always be generated the same way in a given location.

The world is split up into 32x32 block chunks in the XZ plane (Y is up). This allows the world to be “infinite” (floating point precision is currently a problem at large X or Z values) and also makes it easier to manage the data. Only visible chunks need to be queried from the database.

#### Rendering

Only exposed faces are rendered. This is an important optimization as the vast majority of blocks are either completely hidden or are only exposing one or two faces. Each chunk records a one-block width overlap for each neighboring chunk so it knows which blocks along its perimeter are exposed.

Only visible chunks are rendered. A naive frustum-culling approach is used to test if a chunk is in the camera’s view. If it is not, it is not rendered. This results in a pretty decent performance improvement as well.

Chunk buffers are completely regenerated when a block is changed in that chunk, instead of trying to update the VBO.

Text is rendered using a bitmap atlas. Each character is rendered onto two triangles forming a 2D rectangle.

“Modern” OpenGL is used - no deprecated, fixed-function pipeline functions are used. Vertex buffer objects are used for position, normal and texture coordinates. Vertex and fragment shaders are used for rendering. Matrix manipulation functions are in matrix.c for translation, rotation, perspective, orthographic, etc. matrices. The 3D models are made up of very simple primitives - mostly cubes and rectangles. These models are generated in code in cube.c.

Transparency in glass blocks and plants (plants don’t take up the full rectangular shape of their triangle primitives) is implemented by discarding magenta-colored pixels in the fragment shader.

#### Database

User changes to the world are stored in a sqlite database. Only the delta is stored, so the default world is generated and then the user changes are applied on top when loading.

The main database table is named “block” and has columns p, q, x, y, z, w. (p, q) identifies the chunk, (x, y, z) identifies the block position and (w) identifies the block type. 0 represents an empty block (air).

In game, the chunks store their blocks in a hash map. An (x, y, z) key maps to a (w) value.

The y-position of blocks are limited to 0 <= y < 256. The upper limit is mainly an artificial limitation to prevent users from building unnecessarily tall structures. Users are not allowed to destroy blocks at y = 0 to avoid falling underneath the world.

#### Multiplayer

Multiplayer mode is implemented using plain-old sockets. A simple, ASCII, line-based protocol is used. Each line is made up of a command code and zero or more comma-separated arguments. The client requests chunks from the server with a simple command: C,p,q,key. “C” means “Chunk” and (p, q) identifies the chunk. The key is used for caching - the server will only send block updates that have been performed since the client last asked for that chunk. Block updates (in realtime or as part of a chunk request) are sent to the client in the format: B,p,q,x,y,z,w. After sending all of the blocks for a requested chunk, the server will send an updated cache key in the format: K,p,q,key. The client will store this key and use it the next time it needs to ask for that chunk. Player positions are sent in the format: P,pid,x,y,z,rx,ry. The pid is the player ID and the rx and ry values indicate the player’s rotation in two different axes. The client interpolates player positions from the past two position updates for smoother animation. The client sends its position to the server at most every 0.1 seconds (less if not moving).

Client-side caching to the sqlite database can be performance intensive when connecting to a server for the first time. For this reason, sqlite writes are performed on a background thread. All writes occur in a transaction for performance. The transaction is committed every 5 seconds as opposed to some logical amount of work completed. A ring / circular buffer is used as a queue for what data is to be written to the database.

In multiplayer mode, players can observe one another in the main view or in a picture-in-picture view. Implementation of the PnP was surprisingly simple - just change the viewport and render the scene again from the other player’s point of view.

#### Collision Testing

Hit testing (what block the user is pointing at) is implemented by scanning a ray from the player’s position outward, following their sight vector. This is not a precise method, so the step rate can be made smaller to be more accurate.

Collision testing simply adjusts the player’s position to remain a certain distance away from any adjacent blocks that are obstacles. (Clouds and plants are not marked as obstacles, so you pass right through them.)

#### Sky Dome

A textured sky dome is used for the sky. The X-coordinate of the texture represents time of day. The Y-values map from the bottom of the sky sphere to the top of the sky sphere. The player is always in the center of the sphere. The fragment shaders for the blocks also sample the sky texture to determine the appropriate fog color to blend with based on the block’s position relative to the backing sky.

#### Ambient Occlusion

Ambient occlusion is implemented as described on this page:

http://0fps.wordpress.com/2013/07/03/ambient-occlusion-for-minecraft-like-worlds/

#### Dependencies

* GLEW is used for managing OpenGL extensions across platforms.
* GLFW is used for cross-platform window management.
* CURL is used for HTTPS / SSL POST for the authentication process.
* lodepng is used for loading PNG textures.
* sqlite3 is used for saving the blocks added / removed by the user.
* tinycthread is used for cross-platform threading.

#### build 2021.02.14

编译请在 cmd 敲打

```bash
cmake . -G "MinGW Makefiles"
set PATH=C:\MinGW\bin;%PATH%
mingw32-make # 也可以 cmake --build . --config release
```

加了一些打印，就能看出来有没有找到 CURL 库了

```
-- Found CURL: C:/Program Files (x86)/CURL/lib/libcurl_imp.lib (found version "7.75.0")
====== C:/Program Files (x86)/CURL/include
====== C:/Program Files (x86)/CURL/lib/libcurl_imp.lib
====== opengl32
```

在项目根目录编译，不要像下面建新目录了

```bash
mkdir build && pushd build
...
```

两个原因，原理是 craft.exe 会生成在 build 目录中，这样

1. craft.exe 启动时依赖 libcurl.dll，会找不到
2. craft.exe 启动时需要加载贴图，都在 textures 目录下，也找不到

mingw32-make 会调用 gcc，所以还得看一下 gcc 的版本用的对不对

```bash
gcc -v
Using built-in specs.
COLLECT_GCC=D:\MinGW\bin\gcc.exe
COLLECT_LTO_WRAPPER=d:/mingw/bin/../libexec/gcc/mingw32/9.2.0/lto-wrapper.exe
Target: mingw32
Configured with: ../src/gcc-9.2.0/configure --build=x86_64-pc-linux-gnu --host=mingw32 --target=mingw32 --disable-win32-registry --with-arch=i586 --with-tune=generic --enable-static --enable-shared --enable-threads --enable-languages=c,c++,objc,obj-c++,fortran,ada --with-dwarf2 --disable-sjlj-exceptions --enable-version-specific-runtime-libs --enable-libgomp --disable-libvtv --with-libiconv-prefix=/mingw --with-libintl-prefix=/mingw --enable-libstdcxx-debug --disable-build-format-warnings --prefix=/mingw --with-gmp=/mingw --with-mpfr=/mingw --with-mpc=/mingw --with-isl=/mingw --enable-nls --with-pkgversion='MinGW.org GCC Build-2'
Thread model: win32
gcc version 9.2.0 (MinGW.org GCC Build-2)
```

有个小插曲就是老出现 `unresolved external symbol __imp_curl_easy_init` 的报错

发现这条 error log 前面说的是用的 Strawberry 的 gcc

```
E:\GitHub\MineCraft>D:\Strawberry\c\bin\gcc.exe -v
Using built-in specs.
COLLECT_GCC=D:\Strawberry\c\bin\gcc.exe
COLLECT_LTO_WRAPPER=D:/Strawberry/c/bin/../libexec/gcc/x86_64-w64-mingw32/8.3.0/lto-wrapper.exe
Target: x86_64-w64-mingw32
Configured with: ../../../src/gcc-8.3.0/configure --host=x86_64-w64-mingw32 --build=x86_64-w64-mingw32 --target=x86_64-w64-mingw32 --prefix=/mingw64 --enable-shared --enable-static --disable-multilib --enable-languages=c,c++,fortran,lto --enable-libstdcxx-time=yes --enable-threads=posix --enable-libgomp --enable-libatomic --enable-lto --enable-graphite --enable-checking=release --enable-fully-dynamic-string --enable-version-specific-runtime-libs --enable-libstdcxx-filesystem-ts=yes --disable-libstdcxx-pch --disable-libstdcxx-debug --disable-bootstrap --disable-rpath --disable-win32-registry --disable-nls --disable-werror --disable-symvers --with-gnu-as --with-gnu-ld --with-arch=nocona --with-tune=core2 --with-libiconv --with-system-zlib --with-gmp=/opt/build/prerequisites/x86_64-w64-mingw32-static --with-mpfr=/opt/build/prerequisites/x86_64-w64-mingw32-static --with-mpc=/opt/build/prerequisites/x86_64-w64-mingw32-static --with-isl=/opt/build/prerequisites/x86_64-w64-mingw32-static --with-pkgversion='x86_64-posix-seh, Built by strawberryperl.com project' --with-bugurl=https://sourceforge.net/projects/mingw-w64 CFLAGS='-O2 -pipe -fno-ident -I/opt/build/x86_64-830-posix-seh-rt_v6/mingw64/opt/include -I/opt/build/prerequisites/x86_64-zlib-static/include -I/opt/build/prerequisites/x86_64-w64-mingw32-static/include' CXXFLAGS='-O2 -pipe -fno-ident -I/opt/build/x86_64-830-posix-seh-rt_v6/mingw64/opt/include -I/opt/build/prerequisites/x86_64-zlib-static/include -I/opt/build/prerequisites/x86_64-w64-mingw32-static/include' CPPFLAGS=' -I/opt/build/x86_64-830-posix-seh-rt_v6/mingw64/opt/include -I/opt/build/prerequisites/x86_64-zlib-static/include -I/opt/build/prerequisites/x86_64-w64-mingw32-static/include' LDFLAGS='-pipe -fno-ident -L/opt/build/x86_64-830-posix-seh-rt_v6/mingw64/opt/lib -L/opt/build/prerequisites/x86_64-zlib-static/lib -L/opt/build/prerequisites/x86_64-w64-mingw32-static/lib ' LD_FOR_TARGET=/opt/build/x86_64-830-posix-seh-rt_v6/mingw64/bin/ld.exe
Thread model: posix
gcc version 8.3.0 (x86_64-posix-seh, Built by strawberryperl.com project)
```

这个 gcc 是给 posix 系统用的，一看就不对，无奈才改了 PATH 优先级。

猜想 posix 用的是 *.a 的库，win32 用的才是 *.lib 的库

**注意 mingw32 字样。如果不对，请调整 $PATH 优先级。通常它是找不到某某头文件的罪魁祸首，因为平台弄错了。**

项目依赖的三个库

1. CMake 可以通过 Windows 版的安装文件 msi 安装
2. MinGW 需要下载 MinGW Install Manager 安装，主要是为了得到那个 mingw32-make 命令
3. cURL 需要源码编译

#### 编译 curl

curl 是自己编译的，所以才在 C:/Program Files/CURL/ 目录下

curl 最新版仅提供源码了，没有编译后的binary。所以，你还得知道

curl 编译依赖三个库

1. zlib http://www.zlib.net/
2. openssl https://github.com/openssl/openssl/releases 注意不要用那些 alpha 版本
3. libssh2 这里 libssh2 又依赖上面两个库

##### MinGW 编译

编译过程请参考 https://www.cnblogs.com/wunaozai/p/4495441.html 整个编译过程都是在 msys.bat 控制台下进行的

1. zlib

```bash
make -f win32/Makefile.gcc
```

2. openssl

```bash
./Configure -DHAVE_STRUCT_TIMESPEC -L/mingw/lib -lz -lws2_32 --prefix=/mingw zlib mingw
make && make install
```

--prefix=/mingw 就是 MinGW 的安装目录，在 msys.bat 中相当于根目录 /

如果是 Windows 编译，开启 VS2017控制台，用下列命令

```bash
perl Configure VC-WIN32 --prefix=E:\SourceCode\deps
nmake install
```

3. libssh2

网上说这么编

```bash
./configure --prefix=/mingw --with-libz --with-openssl
```

但是 configure 不是默认就有的，得想个办法生成

CSDN上说要这样

```bash
aclocal
autoconf --force
autoheader
automake --add-missing
```

然后 [issue#101](https://github.com/libssh2/libssh2/issues/101) 就会有人来打脸了。你只需要这样

```bash
./buildconf
make && make install
```

##### MSVC 编译三个库

1. zlib

zlib 是存在 CMakeLists.txt 的，直接用 cmake 方式编译

```bash
mkdir build && pushd build
cmake .. -DCMAKE_INSTALL_PREFIX=E:\SourceCode\deps
cmake --build . --config release
cmake --install .
popd
```

2. openssl

开启 VS2017 控制台，用下列命令

```bash
perl Configure VC-WIN32 no-asm --prefix=E:\SourceCode\deps --openssldir=E:\SourceCode\deps
ms\do_ms.bat
nmake -f ms\ntdll.mak install
nmake -f ms\nt.mak install
```

新版本

```bash
perl Configure VC-WIN32 --prefix=E:\SourceCode\deps --openssldir=E:\SourceCode\deps
nmake
nmake install_sw
```

**在1.0.x之前的版本中,文件为libeay32.dll和ssleay32.dll,在1.1.x之后的版本中，名字是libssl.dll和libcrypto.dll。**

所以对编译结果复制改名好了。

3. libssh2

```bash
mkdir build && pushd build
cmake .. -DCMAKE_INSTALL_PREFIX=E:\SourceCode\deps -DCRYPTO_BACKEND=OpenSSL -DENABLE_ZLIB_COMPRESSION=ON
cmake --build . --target install
```

中间会遇到一些插曲，找不到一些头文件，是因为查到 D:\MinGW 目录去了。MSVC 编译怎么会找到这里呢，发现 cmake .. -G "Visual Studio 15 2017" 都没有用，索性直接把 MinGW 删除了。

实在不放心的可以在 CMakeLists.txt 中加入

```
SET(CMAKE_INCLUDE_PATH "E:/SourceCode/deps/include")
SET(CMAKE_LIBRARY_PATH "E:/SourceCode/deps/lib")
```

**做到这里暂时先忘记之前的 MinGW 编译吧。**

##### 编译 curl

无加密版

```bash
pushd lib && mingw32-make -f Makefile.m32 && popd
pushd src && mingw32-make -f Makefile.m32 && popd
```

得到 .exe .a .lib .dll

有加密版，需要建立一个 deps 目录

```
 If you wish to support zlib, openssl, c-ares, ssh2, you will have to download
 them separately and copy them to the deps directory as shown below:

    somedirectory\
     |_curl-src
     | |_winbuild
     |
     |_deps
       |_ lib
       |_ include
       |_ bin
```

所以 zlib openssl ssh2 的安装目录 --prefix=/mingw 我们应该设成  ../deps

或者直接把根目录的三个目录覆盖过来

控制打开 "适用于 VS 2017 的 x86 本机工具命令提示" 进入 winbuild 建立 bat 文件执行

```bat
@REM @echo off

@IF [%1]==[debug] (

@echo 正在使用debug模式编译libcurl~~~

@nmake /f Makefile.vc mode=static VC=15 WITH_SSL=static WITH_ZLIB=static WITH_SSH2=static ENABLE_IDN=no RTLIBCFG=static DEBUG=yes MACHINE=x86

) ELSE (

@echo 正在使用release模式编译libcurl~~~

@nmake /f Makefile.vc mode=static VC=15 WITH_SSL=static WITH_ZLIB=static WITH_SSH2=static ENABLE_IDN=no RTLIBCFG=static DEBUG=no MACHINE=x86

)

@REM @echo on
```

**但是上面的官方做法很坑，各种找不到符号。** 最后还是用 cmake 的方式完成的

```bash
mkdir build && pushd build
cmake .. -DCMAKE_USE_OPENSSL=ON -DUSE_ZLIB=ON -DCMAKE_USE_LIBSSH2=ON -DOPENSSL_ROOT_DIR=E:\SourceCode\openssl-OpenSSL_1_1_1i
cmake --build . --config release
cmake --install .
```

因为默认都是 x86 编译，会安装到 C:\Program Files (x86)\CURL 中

这时的 curl.exe 还不能正常启动，需要将 deps/bin 下所有的 dll 拷贝过来才行

##### 判断 curl 是否带 openSSL

```bash
> curl.exe -V
curl 7.75.0 (Windows) libcurl/7.75.0 OpenSSL/1.1.1i
Release-Date: 2021-02-03
Protocols: dict file ftp ftps gopher gophers http https imap imaps ldap mqtt pop3 pop3s rtsp smb smbs smtp smtps telnet tftp
Features: AsynchDNS HTTPS-proxy IPv6 Largefile NTLM SSL alt-svc
```

与原装无加密的比较

```bash
> curl.exe -V
curl 7.75.0 (Windows) libcurl/7.75.0
Release-Date: 2021-02-03
Protocols: dict file ftp gopher http imap ldap mqtt pop3 rtsp smb smtp telnet tftp
Features: AsynchDNS IPv6 Largefile NTLM alt-svc
```