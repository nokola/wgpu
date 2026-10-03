# FanRust patches to wgpu-hal 30.0.1

The branch `fanrust-30.0.1` of `nokola/wgpu` is the wgpu `v30.0.1` release
plus one commit per change below. FanRust uses its `wgpu-hal` through
`[patch.crates-io]`, pinned to a commit of this branch, so every build uses
the same code. All the changes are in the GL backend
(`wgpu-hal/src/gles/`); Vulkan and the other backends are untouched. Every
change is marked `FanRust patch` in the code. Why they exist, with the
numbers: `docs/todo-perf-smudge.md` PS13, PS33, PS34 in FanRust (the
OnePlus A0001, Adreno 330, GLES 3.0).

To see the changes: `git log v30.0.1..fanrust-30.0.1` (one commit each), or
`git diff v30.0.1 -- wgpu-hal`.

On a new wgpu release: branch `fanrust-<version>` from the release tag,
re-apply these commits (drop any wgpu has taken), run the checks at the
end, tag the result and move FanRust's pin to it. Never force-push over a
pinned commit: an old FanRust release must still build.

## The changes

1. **Kept framebuffers** (`mod.rs` `FramebufferCache`, `command.rs`
   `cached_framebuffer_key`, `queue.rs` `C::BindCachedFramebuffer`,
   `device.rs` `destroy_texture`). A render pass with plain colour targets
   binds a framebuffer kept for exactly those targets instead of
   re-attaching them to one shared framebuffer.
   - Assumes: GL reuses a texture name only after `glDeleteTextures`, and
     `destroy_texture` runs before that (it forgets every framebuffer that
     names the texture). A texture wgpu does not own (`drop_guard`) must not
     be deleted by its owner before wgpu destroys its wrapper.
   - Assumes: one GL context per adapter. Framebuffers are not shared
     between contexts; a second context would need its own cache.
   - Breaks if: code outside wgpu-hal deletes textures or framebuffers
     through raw GL.
   - Cap: 256 framebuffers, least recently used goes. A frame drawing to
     more targets than that would create and delete one per pass.

2. **Framebuffer 0 only at a submit's start** (`queue.rs` `reset_state`).
   wgpu-core records every render pass as two command buffers, and binding
   framebuffer 0 before each cost a GPU job on the Adreno 330.
   - Assumes: no command reads or draws through a framebuffer it did not
     bind itself. True in 30.0.1: a pass binds its own; texture copies,
     texture-to-buffer reads and resolves bind `copy_fbo` / `draw_fbo`.
   - Re-check on a wgpu upgrade: every `READ_FRAMEBUFFER` /
     `DRAW_FRAMEBUFFER` user in `queue.rs` binds explicitly.

3. **`glDrawBuffers` only when changed** (`mod.rs`
   `FramebufferCache::draw_buffers_already_set`, `queue.rs`
   `C::SetDrawColorBuffers`). Draw buffers are framebuffer state in GLES 3,
   recorded per kept framebuffer.
   - Assumes: every `glDrawBuffers` outside `SetDrawColorBuffers` clears the
     record (the one other caller, the Mesa shader-clear workaround, does),
     and every bind of a draw framebuffer that is not a kept one clears
     `bound_entry` (`reset_state`, `ResetFramebuffer`, `ResolveAttachment`).
   - Re-check on a wgpu upgrade: `grep -n "draw_buffers(\|bind_framebuffer("`
     in `src/gles/` — each site must be one of the above.

4. **Holding the context through a frame** (`egl.rs`
   `AdapterContext::hold_current` / `release_current`; `wgl.rs` has no-op
   twins). The holding thread's calls skip EGL make-current and release;
   any other thread waits for the hold to end.
   - Assumes: the holder never waits for another thread that uses wgpu
     while it holds. A thread that waits logs a warning after 250 ms and
     panics after 6 s with a message naming `hold_current`. FanRust holds
     only in the shell's frame (`GlFrameHold`) and not while the blit
     pipelines compile on their thread.
   - Assumes: `Instance::create_surface` and adapter enumeration (they make
     the context current on their own) do not run on another thread while
     a hold is on. In FanRust they run on the main thread, outside frames.
   - Self-heals: a `lock` on the holding thread that finds the context no
     longer current (a failed present, a new surface) makes it current
     again.
   - Threads are told apart by a thread-local token, so a GL object dropped
     from a thread-local's destructor counts as "another thread" and waits.
   - Multithreaded rendering: recording command encoders on any thread
     needs no context. A render thread that owns all GL work can hold
     instead of the main thread; with the hold on the render thread, every
     main-thread wgpu call that needs GL (buffer and texture writes,
     creating and dropping resources, submits) waits until the hold ends.

5. **One always-enabled vertex attribute** (`adapter.rs` `open`, `queue.rs`
   `enable_spare_vertex_attribute`, `C::UnsetVertexAttribute`, `mod.rs`
   `Queue::drop`). The last attribute location reads the zero buffer, so
   no draw is attribute-less (the Adreno 330 logs every such draw).
   - A pipeline may use that location: wgpu points it at the pipeline's
     buffer for the pass, and `UnsetVertexAttribute` puts the spare array
     back instead of disabling it. Pinned by the GPU fixture's
     `last_vertex_attribute` stage.
   - Assumes: nothing writes into the zero buffer (wgpu only copies out of
     it), and it is deleted only after the array is disabled (`Queue::drop`).
   - Assumes: one vertex array object for the device's life (true in
     30.0.1: `open` creates and binds the only one).

6. **A replay counter** (`mod.rs` `COMMAND_BUFFERS_REPLAYED`): command
   buffers `Queue::submit` replayed, read by FanRust's `GPU-JOBS` bench.

7. **Present lets go of the context after a failed swap** (`egl.rs`
   `Surface::present`). Upstream returned on a `swap_buffers` error with
   the context still current on the presenting thread (its own `TODO`), so
   the next thread to lock the context panicked on eglMakeCurrent's
   `BadAccess`. Seen on the Redmi Note 13 (Adreno 610) and repeated on the
   OnePlus A0001: leave the app during a cold start and come back while it
   still boots; the first frames go to the destroyed window.
   - Now the context is released (or, when held, returned to its own
     surface) on both paths, and the swap's error is returned after.
   - Re-check on a wgpu upgrade: every `?` / `return Err` in `present`
     after its `make_current(surface)` goes through the same release.

8. **A zero byte after every text a GL debug call gets** (`mod.rs`
   `zero_terminated` / `without_zero`, `command.rs` `add_marker` and
   `begin_encoding`, `queue.rs` `to_debug_str` and the command buffer's
   group in `submit`, `device.rs` `object_label`). `glPushDebugGroup`,
   `glDebugMessageInsert` and `glObjectLabel` get the text's length, as GL
   asks, but the Adreno 630 driver on a Samsung Galaxy S9 (Android 10)
   copies it with `strlcpy`, which reads on to a zero byte whatever the
   length. Rust text has none after it, so the copy ran past the label and
   the app crashed in `strlcpy` under `libGLESv2_adreno.so` when it reached
   unmapped memory (Play, Fantasia Painter 57, from `Queue::submit`). The
   length passed is still the text's own; the zero byte only stops the
   overread.
   - Also: the shader label (`compile_shader`) now asks `DEBUG_FNS` like
     every other label; upstream asked only `supports_debug`, so
     `GlDebugFns::Disabled` still labelled shaders.
   - Pinned by `command.rs` tests `marker_is_followed_by_a_zero_byte` and
     `label_round_trips_with_a_zero_byte` (`cargo test -p wgpu-hal
     --features gles --lib gles::`).
   - Re-check on a wgpu upgrade: `grep -n "object_label\|push_debug_group\|debug_message_insert"`
     in `src/gles/` — every call goes through `object_label` /
     `add_marker` / `zero_terminated`.
   - FanRust's store bundles also turn the debug calls off altogether
     (`GlDebugFns::Disabled`, `imaging_wgpu::headless` STORE_BUILD); this
     change covers the dev builds, which keep the names for GPU captures.

## Checking after any change here or a wgpu upgrade

All run in FanRust, with its `[patch.crates-io]` pointing at the new commit.

- PC: `cargo test -p imaging_wgpu`, and again with `WGPU_BACKEND=gl` (the
  GL backend through WGL; `hold_current` is a no-op there).
- OnePlus A0001: `scripts\build.ps1 -GpuFixture -AllAbi`, deploy, every
  `GPU-FIXTURE` stage OK. Then `python scripts/perf_jobs_bench.py <label>`
  (jobs per call; the numbers to compare against are in
  todo-perf-smudge.md PS13 / PS33) and `perf_stroke_atrace.py` on a smudge.
- The driver's log line: during a stroke, `adb logcat | grep -c
  validate_vertex_attrib_state` stays 0.
