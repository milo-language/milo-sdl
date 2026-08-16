# sdl

This is a package for the [Milo language](https://milo-language.github.io/milo/).

## Overview

SDL2 bindings: video, input, gamepad and audio.

This is the `extern` declarations and the constants, nothing else. No `Window`
type, no `Drop` impls, no frame-loop helper, no opinion about how you poll
input. Those choices differ per project and are cheap to write; the declarations
are what everybody was retyping.

Every function carries `@cSig` and every constant `@cValue`, so the whole
surface is verified against SDL's own headers at build time when they are
installed, and skipped with a named warning when they are not.

The C-header guards, and the gotchas that are invisible in the signatures:
[docs/api.md](docs/api.md).

## Installation

```bash
brew install sdl2                 # macOS
apt install libsdl2-dev           # Debian/Ubuntu

milo add github.com/milo-language/milo-sdl            # latest release
milo add github.com/milo-language/milo-sdl@v0.3.1     # or pin a tag
```

| Import | Contents |
|---|---|
| `"sdl"` | init/quit, window, renderer, texture, events, keyboard, mouse, timing |
| `"sdl/glcontext"` | OpenGL context: attributes, creation, drawable size, buffer swap |
| `"sdl/keys"` | USB HID scancodes — what `SDL_GetKeyboardState` is indexed by |
| `"sdl/gamepad"` | GameController API, buttons, hotplug events |
| `"sdl/audio"` | queue-API audio device and `SDL_AudioSpec` |

## Examples

### A window and a framebuffer

```milo
from "sdl" import {
    SDL_Init, SDL_Quit, SDL_CreateWindow, SDL_CreateRenderer, SDL_CreateTexture,
    SDL_UpdateTexture, SDL_RenderClear, SDL_RenderCopy, SDL_RenderPresent,
    SDL_INIT_VIDEO, SDL_WINDOWPOS_CENTERED, SDL_WINDOW_SHOWN,
    SDL_TEXTUREACCESS_STREAMING, SDL_PIXELFORMAT_ABGR8888,
}

let W: i64 = 320
let H: i64 = 240

fn main(): i32 {
    unsafe {
        SDL_Init(SDL_INIT_VIDEO)
        let win = SDL_CreateWindow("demo", SDL_WINDOWPOS_CENTERED, SDL_WINDOWPOS_CENTERED,
                                   W as i32, H as i32, SDL_WINDOW_SHOWN)
        let ren = SDL_CreateRenderer(win, -1, 0)
        let tex = SDL_CreateTexture(ren, SDL_PIXELFORMAT_ABGR8888,
                                    SDL_TEXTUREACCESS_STREAMING, W as i32, H as i32)

        var fb: Vec<u32> = Vec.new()
        for i in 0..(W * H) { fb.push(0xFF102030) }

        SDL_UpdateTexture(tex, 0 as *u8, fb.ptr() as *u8, (W * 4) as i32)
        SDL_RenderClear(ren)
        SDL_RenderCopy(ren, tex, 0 as *u8, 0 as *u8)
        SDL_RenderPresent(ren)
        SDL_Quit()
    }
    return 0
}
```

A moving version of the same thing: `examples/plasma.milo`.

### Getting an OpenGL context

`"sdl/glcontext"` is how you get a context, not how you draw. For the drawing,
use a GL binding such as [`gl`](https://github.com/milo-language/milo-gl). The
order below is the part that costs people an afternoon:

```milo
from "sdl" import {
    SDL_Init, SDL_CreateWindow, SDL_INIT_VIDEO,
    SDL_WINDOWPOS_CENTERED, SDL_WINDOW_SHOWN
}
from "sdl/glcontext" import {
    SDL_GL_CreateContext, SDL_GL_SetAttribute, SDL_GL_SetSwapInterval,
    SDL_GL_GetDrawableSize,
    SDL_GL_CONTEXT_MAJOR_VERSION, SDL_GL_CONTEXT_MINOR_VERSION,
    SDL_GL_CONTEXT_PROFILE_CORE, SDL_GL_CONTEXT_PROFILE_MASK, SDL_WINDOW_OPENGL,
}

fn main(): i32 {
    unsafe {
        SDL_Init(SDL_INIT_VIDEO)

        // 1. Attributes FIRST: SDL reads them when it picks the pixel format.
        SDL_GL_SetAttribute(SDL_GL_CONTEXT_MAJOR_VERSION, 3)
        SDL_GL_SetAttribute(SDL_GL_CONTEXT_MINOR_VERSION, 3)
        SDL_GL_SetAttribute(SDL_GL_CONTEXT_PROFILE_MASK, SDL_GL_CONTEXT_PROFILE_CORE)

        // 2. Then the window, with the OPENGL flag.
        let win = SDL_CreateWindow("demo", SDL_WINDOWPOS_CENTERED, SDL_WINDOWPOS_CENTERED,
                                   1280, 720, SDL_WINDOW_SHOWN | SDL_WINDOW_OPENGL)

        // 3. Then the context, 4. then vsync.
        let ctx = SDL_GL_CreateContext(win)
        SDL_GL_SetSwapInterval(1)

        // The drawable is not the window: on a Retina display it is twice as
        // large in each axis, and the viewport wants this one.
        var dw: i32 = 0
        var dh: i32 = 0
        SDL_GL_GetDrawableSize(win, dw.addrOf(), dh.addrOf())
        print($"window 1280x720, drawable {dw}x{dh}")
    }
    return 0
}
```

Set those attributes *after* the window exists and they are accepted, return 0,
and change nothing: you ask for 3.3 core, silently get whatever the driver felt
like, and find out much later from a shader compile error that does not mention
the version. A window also cannot be both, so calling `SDL_CreateRenderer` on a
GL window takes the context over. Pick one before the window exists.
