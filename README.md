# sdl

SDL2 bindings for [Milo](https://github.com/milo-language/milo). Video, input, gamepad,
and audio.

```bash
milo add github.com/milo-language/milo-sdl            # latest release
milo add github.com/milo-language/milo-sdl@v0.2.0     # or pin a tag
```

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

## Bindings, not a framework

This package is the `extern` declarations and the constants — nothing else. No `Window`
type, no `Drop` impls, no frame-loop helper, no opinion about how you poll input. Those
choices differ per project and are cheap to write; the declarations are what everybody
was retyping.

## Modules

| Import | Contents |
|---|---|
| `"sdl"` | init/quit, window, renderer, texture, events, keyboard, mouse, timing |
| `"sdl/gl"` | OpenGL context: attributes, creation, drawable size, buffer swap |
| `"sdl/keys"` | USB HID scancodes — what `SDL_GetKeyboardState` is indexed by |
| `"sdl/gamepad"` | GameController API, buttons, hotplug events |
| `"sdl/audio"` | queue-API audio device and `SDL_AudioSpec` |

## Getting an OpenGL context

`"sdl/gl"` is how you get a context, not how you draw — for the drawing, use a GL
binding such as [`gl`](https://github.com/milo-language/gl). The order matters and is
the part that costs people an afternoon:

```milo
from "sdl" import { SDL_CreateWindow, SDL_WINDOWPOS_CENTERED, SDL_WINDOW_SHOWN }
from "sdl/gl" import {
    SDL_GL_CreateContext, SDL_GL_SetAttribute, SDL_GL_SetSwapInterval,
    SDL_GL_CONTEXT_MAJOR_VERSION, SDL_GL_CONTEXT_MINOR_VERSION,
    SDL_GL_CONTEXT_PROFILE_CORE, SDL_GL_CONTEXT_PROFILE_MASK, SDL_WINDOW_OPENGL,
}

SDL_GL_SetAttribute(SDL_GL_CONTEXT_MAJOR_VERSION, 3)      // 1. before the window
SDL_GL_SetAttribute(SDL_GL_CONTEXT_MINOR_VERSION, 3)
SDL_GL_SetAttribute(SDL_GL_CONTEXT_PROFILE_MASK, SDL_GL_CONTEXT_PROFILE_CORE)
let win = SDL_CreateWindow(title.cstr(), SDL_WINDOWPOS_CENTERED, SDL_WINDOWPOS_CENTERED,
                           w, h, SDL_WINDOW_SHOWN | SDL_WINDOW_OPENGL)   // 2. the flag
let ctx = SDL_GL_CreateContext(win)                                      // 3.
SDL_GL_SetSwapInterval(1)                                                // 4.
```

SDL reads the attributes when it picks the window's pixel format and ignores them
afterwards. Set them *after* the window exists and they are accepted, return 0, and
change nothing — you ask for 3.3 core, silently get whatever the driver felt like, and
find out much later from a shader compile error that does not mention the version.

Two more that are invisible in the signatures. **A window cannot be both**: calling
`SDL_CreateRenderer` on a GL window takes the context over, so pick one before the
window exists. And **`SDL_GL_GetDrawableSize` is not `SDL_GetWindowSize`** — on a Retina
display the drawable is twice the window in each axis, and setting the viewport from the
wrong one renders the frame into a quarter of the window.

## Everything is checked against the real headers

Each `extern fn` carries `@cSig` and each constant carries `@cValue`, so the whole
surface is verified against SDL's own headers at build time:

```
error[c-decl]: a declaration does not match the C header it claims to describe
  sdl$SC_W: Milo says 27, SDL2/SDL.h defines SDL_SCANCODE_W as something else
```

This is not hypothetical. Writing this package turned up a real mismatch on the first
build: `SDL_IsGameController` returns `SDL_bool`, an enum whose enumerators are both
non-negative — so C gives it an *unsigned* underlying type. Every hand-written copy of
that declaration across our own projects said `i32`.

**You do not need SDL's headers to build.** Milo links SDL by name. If the headers are
absent — a machine with `libSDL2` but not `libsdl2-dev` — the guards announce that they
are skipping and the build proceeds:

```
warning: @cLayout/@cSig/@cValue guards skipped — 'SDL2/SDL.h' is not installed,
so there is no header to check these declarations against
```

Requires Milo 0.1.0 or newer for `@cValue`.

## Installing SDL2

```bash
brew install sdl2                 # macOS
apt install libsdl2-dev           # Debian/Ubuntu
```

Homebrew's `sdl2` formula now installs **sdl2-compat**, an SDL2 API layer over SDL3. The
bindings here are ABI-compatible with it and the constants verify identically, so nothing
in this package cares — but it explains why `sdl2-config --version` may report an SDL3
build underneath.

Header discovery for the build-time guards goes through `pkg-config --cflags sdl2`.

## Two things the signatures do not tell you

**Ask for the subsystems you will use.** Leaving `SDL_INIT_AUDIO` out of `SDL_Init` does
not error — `SDL_OpenAudioDevice` just returns 0, so every sound is a silent no-op and
the program otherwise runs perfectly. This cost one of our games its entire soundtrack.

**`SDL_QueueAudio` appends, it does not mix.** Music and effects on one device serialise:
an effect waits behind everything already queued before it is audible. Open a second
device and let the OS mixer do the mixing.

## Pointers

`SDL_Window*`, `SDL_Renderer*` and `SDL_Texture*` are all `*u8` here — Milo has no opaque
handle type in an `extern` position, so nothing stops you passing one where another is
wanted. Keep them in named variables and the mistake is hard to make.

## License

MIT
