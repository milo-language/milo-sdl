# sdl notes

The surface is SDL2's own, so [SDL's documentation](https://wiki.libsdl.org/SDL2/)
is the reference for what each function does. What follows is what is specific
to these bindings.

## Modules

| Import | Contents |
|---|---|
| `"sdl"` | init/quit, window, renderer, texture, events, keyboard, mouse, timing |
| `"sdl/glcontext"` | OpenGL context: attributes, creation, drawable size, buffer swap |
| `"sdl/keys"` | USB HID scancodes — what `SDL_GetKeyboardState` is indexed by |
| `"sdl/gamepad"` | GameController API, buttons, hotplug events |
| `"sdl/audio"` | queue-API audio device and `SDL_AudioSpec` |

## Everything is checked against the real headers

Each `extern fn` carries `@cSig` and each constant carries `@cValue`, so the
whole surface is verified against SDL's own headers at build time:

```
error[c-decl]: a declaration does not match the C header it claims to describe
  sdl$SC_W: Milo says 27, SDL2/SDL.h defines SDL_SCANCODE_W as something else
```

This is not hypothetical. Writing this package turned up a real mismatch on the
first build: `SDL_IsGameController` returns `SDL_bool`, an enum whose
enumerators are both non-negative — so C gives it an *unsigned* underlying type.
Every hand-written copy of that declaration across our own projects said `i32`.

**You do not need SDL's headers to build.** Milo links SDL by name. If the
headers are absent — a machine with `libSDL2` but not `libsdl2-dev` — the guards
announce that they are skipping and the build proceeds:

```
warning: @cLayout/@cSig/@cValue guards skipped — 'SDL2/SDL.h' is not installed,
so there is no header to check these declarations against
```

Requires Milo 0.1.0 or newer for `@cValue`. Header discovery for the guards goes
through `pkg-config --cflags sdl2`.

## sdl2-compat

Homebrew's `sdl2` formula now installs **sdl2-compat**, an SDL2 API layer over
SDL3. The bindings here are ABI-compatible with it and the constants verify
identically, so nothing in this package cares — but it explains why
`sdl2-config --version` may report an SDL3 build underneath.

## Things the signatures do not tell you

**Ask for the subsystems you will use.** Leaving `SDL_INIT_AUDIO` out of
`SDL_Init` does not error — `SDL_OpenAudioDevice` just returns 0, so every sound
is a silent no-op and the program otherwise runs perfectly. This cost one of our
games its entire soundtrack.

**`SDL_QueueAudio` appends, it does not mix.** Music and effects on one device
serialise: an effect waits behind everything already queued before it is
audible. Open a second device and let the OS mixer do the mixing.

**A GL window cannot also be a renderer window.** Calling `SDL_CreateRenderer`
on a window created with `SDL_WINDOW_OPENGL` takes the context over, so pick one
before the window exists.

**`SDL_GL_GetDrawableSize` is not `SDL_GetWindowSize`.** On a Retina display the
drawable is twice the window in each axis, and setting the viewport from the
wrong one renders the frame into a quarter of the window.

**GL attributes are read when the window is created.** Set them afterwards and
they are accepted, return 0, and change nothing.

## Pointers

`SDL_Window*`, `SDL_Renderer*` and `SDL_Texture*` are all `*u8` here — Milo has
no opaque handle type in an `extern` position, so nothing stops you passing one
where another is wanted. Keep them in named variables and the mistake is hard to
make.
