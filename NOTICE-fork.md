# What this fork changes

A fork of [origami-ltd/wasm-revc](https://github.com/origami-ltd/wasm-revc), which is itself a
browser port of [mrxenginner/reVC](https://github.com/mrxenginner/reVC) — a reverse-engineered
reimplementation of the Grand Theft Auto: Vice City engine.

**Essentially all of the work here is theirs.** reVC is the game engine; wasm-revc is the
Emscripten port, the SDL2 skeleton and the build. This fork exists to carry a small number of
changes needed to run the engine on [abhishekstation.pages.dev](https://abhishekstation.pages.dev),
and is published so those changes are inspectable rather than living on one laptop.

No game data is redistributed here or by the site. Vice City's assets belong to Rockstar Games;
a player supplies their own from a copy they own, and the files never leave their device.

## The changes

### `src/core/CdStream_emscripten.cpp` — a failed read no longer wedges the tab

`CStreaming` retries on a non-zero stream status in two places, with no yield and no bail-out:

```c
do   status = CdStreamRead(0, buf, imgOffset+posn, size);
while(CdStreamSync(0) || status == STREAM_NONE);      // Streaming.cpp:2479

while (CdStreamSync(ch) != STREAM_NONE)               // Streaming.cpp:2417
    CdStreamRead(ch, ...);                            // "Try again on error"
```

On POSIX that is safe — a retry re-queues the request to a worker thread, which may succeed.
This backend has no worker: the read already happened, synchronously, on the one thread a page
has. The retry re-runs the identical call, fails identically, and the loop never exits and never
yields. The tab sits at 100% CPU until it is killed, which is what the browser reports as
"Page Unresponsive".

`STREAM_ERROR` is `0xFE`, so one failed read is enough to enter that state permanently. And a
failure is reachable: `CdStreamRemoveImages()` zeroes every entry in `gImgFiles`, and
`LoadPlayerDff()` calls it after adding `gta3.img` on the fly, so `hFile` can be `-1` by the time
a queued request reaches the read — and `read(-1, ...)` returns `-1` every time.

A failed read now zero-fills the sectors and reports the read as done, with a `[DBG]` line naming
the image and offset. A model that comes out wrong beats a page that never comes back. All three
`nStatus` writes in the file are now `STREAM_NONE`, so neither retry loop can spin.

### `src/skel/sdl2/sdl2.cpp`, `src/CMakeLists.txt` — two exports for the host page

`ViceInVehicle()` and `ViceMenuActive()`, so the browser page can lay out its on-screen controls
to match what the player is doing: a stick and action buttons on foot, steering and pedals in a
car, and nothing at all over the frontend menus. The engine has no touch input; the page
synthesises the keyboard events the engine already reads, and these two answers are the only
thing it needs back.

## Licence

Upstream's terms are unchanged and still apply; see `LICENSE.md`. Every file here that came from
reVC or wasm-revc remains theirs.
