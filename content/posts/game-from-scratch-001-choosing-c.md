---
title: Game from Scratch 001 - Choosing C
description: Building a game from scratch with only C. Getting set up with a build script.
tags:
    - game
    - game dev
    - c
    - crt
    - avoiding c runtime
date: 2025-06-16
---

## Language Choice
In my [previous post]({{< relref "posts/game-from-scratch-000-dream-big.md" >}}) I talked about having to make a decision on what language to pick between C++ and Odin. In the end, I went with something else entirely: C. The main thing I don't like with the language I use at work, C#, is that it's grown bloated. It's starting to feel messy and it seems like Microsoft no longer knows where they want to take it. That's also one of the main arguments against C++ as a language, that it's too messy and has too many features.

What I like about C even though I haven't used it since university is that it's a really small language. It's easy to read because it doesn't have ten different ways of iterating over a collection for example. It's low-level so you have a lot of control. It also has lots of documentation, examples and tutorials that will help me when I get stuck. It might seem archaic to choose C over modern alternatives like Odin and Zig but I have a good feeling about this!

## Escaping the C Runtime
One cool thing about C is the control it gives you. I found out while reading that you can actually avoid most or all of the C Runtime (CRT). The CRT is what runs before your main function is even called, more on that later. It's also a bunch of functions that each platform must provide. Things like malloc, printf and standard math functions are included too.

So why skip it? Why ditch those handy cross-platform utilities? Why go through all this extra work? I've seen some really experienced developers complain about the quality of the CRT but I can't really speak to that because I don't have enough experience with it. Honestly, I see it as a challenge and a way to learn. I also hope to gain three specific things by not using the CRT:

### Smaller binaries
By avoiding the C Runtime I can reduce the size of my final binary by a lot (yes, I know about dynamic linking but play along). CRT version: 101KB. My version: 4KB. That's a huge difference! These days, with fast internet a smaller binary size is not that important but I like to believe that people appreciate it. It's also a fun challenge to keep the binary size small.

### Startup speed
I wrote earlier that your main function is not the first thing that gets called when your program starts. The CRT runs first to set things up and parse arguments before it even calls your main function. By avoiding all of that I should, in theory, get a much faster startup which will hopefully be noticeable.

### Full control
What I lose in convenience I gain in control. I won't be able to use things such as malloc to get a platform independent way of allocating memory. That's no issue for me though since I intend to write platform specific code and utilize the power of each platform. A good replacement for malloc would be VirtualAlloc on Windows. I'm hoping to separate all game code from the platform specific implementations so this should not be a problem for me.

## Build.bat
I'm not using an IDE, as can be seen in my [previous post]({{< relref "posts/game-from-scratch-000-dream-big.md" >}}) so I need a way to build my code. I didn't want anything overly complex like [CMake](https://cmake.org/). Eventually, I found the idea of a [single script file](https://github.com/EpicGamesExt/raddebugger/blob/master/build.bat) used for building an entire project. I wrote my own version and it has honestly been great. Just 20 lines of batch script, building both debug and release versions every time. Compile times are under a second, so there's no reason not to build both versions.

Here's my entire build script. Finding out what each flag does is left as an exercise to the reader.
```batch
set input_files=         ..\..\src\game.c
set output_file=         game.exe
set output_dir_debug=    build\debug
set output_dir_release=  build\release

set cl_flags=            /nologo /W3 /WX /FC /Z7 /GS- /Gs999999
set cl_link=             /link /incremental:no /opt:icf /opt:ref /subsystem:windows

if not exist %output_dir_debug%  mkdir %output_dir_debug%
if not exist %output_dir_release% mkdir %output_dir_release%

echo [building debug...]	
pushd %output_dir_debug%
call cl /Od %cl_flags% /Fe: %output_file% %input_files% %cl_link%
popd

echo [building release...]
pushd %output_dir_release%
call cl /O2 %cl_flags% /Fe: %output_file% %input_files% %cl_link%
popd
```

## Next up
Next, I'll create a window and handle the Windows message loop. With that, we'll move into some more exciting C code.

Should be fun!
