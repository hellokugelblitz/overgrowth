# main — program entrypoint and bootstrap

Location
- Source: Source/Main/main.cpp

Purpose
- Implements program startup, command-line parsing, global initialization, main loop, and clean shutdown.
- Provides platform-specific bootstrap (desktop vs PS4 stub).
- Entrypoints:
  - main(...) — parses CLI, selects mode (normal, DDSConvert, unit tests).
  - GameMain(...) — performs full engine init, runs main loop, and shuts down.

Audience
- Integrators, build engineers, and contributors who need to understand program startup, CLI options, or where to hook early initialization/teardown behavior.

High-level flow
1. main(argc, argv)
   - Enable FPU exceptions (optional).
   - Parse TCLAP command-line options.
   - Configure global flags (debug output, write/working dir overrides, special modes).
   - Choose execution path:
     - DDSConvertMain (asset conversion tool)
     - RunUnitTests (if compiled with UNIT_TESTS)
     - RunWithCrashReport -> GameMain (normal execution)
2. GameMain(argc, argv)
   - Register main thread id.
   - Platform-specific DPI initialization (Windows).
   - Seed random generator and initialize allocation helpers.
   - Substitute allocation callbacks for external libs (Recast/Detour).
   - Initialize profiler context.
   - Call SetUpEnvironment (config, paths).
   - Register log handlers (console, file, ram).
   - Construct Engine instance and call Engine::Initialize().
   - Enter main loop:
     - Per-frame: Engine::Update(), optionally Engine::Draw(), buffer swap, state cleanup.
     - Frame rate limiting (if vsync disabled and config limits FPS).
     - SDL_Delay(0) to yield to other threads and PROFILER_TICK.
   - Flush/save on exit:
     - Write queued savefile data, dispose engine, free resources, finalize profiler and logs.

Main loop details
- Update: engine->Update() runs input handling, state machine, background loading and other subsystems.
- Draw: engine->Draw() called when rendering enabled; Graphics::SwapToScreen() and ClearGLState() are used each frame.
- Frame limiting: when vSync is off, config_.limit_fps_in_game() and max_frame_rate restrict rendering frequency; sleeps and small delays are used to cap FPS.
- Profiler integration: STIMING_* macros and g_profiler_ctx label major phases (SetUpEnvironment, Engine initialize, STUpdate, STDraw, STDrawSwap).

Command-line options (selected from main.cpp)
- -c, --config <string> — configuration string (overrides/merges into config).
- -l, --level <string> — debug: load level on startup.
- --write-dir <string> — override write directory.
- --working-dir <string> — override working directory.
- --ogda-manifest <string> — use Ogda manifest.
- --ddsconvert — run DDSConvert asset tool and exit.
- -d, --debug-output — enable debug log level.
- -s, --spam-output — enable spam log level.
- --quit-after-load — quit after loading the specified level.
- --no-dialogues — automatic dialogue responses (skip UI dialogues).
- --clear-log — truncate logfile on start.
- --disable-rendering — skip drawing (headless mode).
- --load-all-levels — load all levels from manifest (test mode).
- --clear-cache, --clear-cache-dry-run — clear known cache files (write folder).
- --level-load-stress — loop-load levels for stress testing.
- --run-unit-tests (build-time only) — execute unit tests.

Global flags set by main.cpp
- overloadedWriteDir / overloadedWorkingDir — external overrides for file paths.
- debug_output, spam_output, clear_log, quit_after_load, no_dialogues, disable_rendering, load_all_levels, clear_cache, clear_cache_dry_run, level_load_stress

Platform notes
- Desktop: full bootstrap, SDL, profiler, logging, allocation overrides, Recast/Detour allocators.
- Windows: attempts to set per-monitor DPI awareness. On debug builds SymInitialize/SymCleanup are used for better stack traces.
- PS4: minimal stub path in #else branch (file open test). PS4-specific startup not implemented in this source.

Logging & crash handling
- Uses LogSystem with multiple handlers: ConsoleHandler, FileHandler, RamHandler.
- FileHandler created once SetUpEnvironment provides write path.
- Crash reporting wrapper: RunWithCrashReport(GameMain) is used to capture crashes and ensure diagnostics are written.
- ProfilerContext is used to group and emit profiling ranges.

Memory & allocator behavior
- Allocation alloc; alloc.Init()/alloc.Dispose() wrap engine lifetime and profiling of allocations.
- Custom allocator callbacks provided for Recast (rcAlloc) and Detour (dtAlloc) via dtAllocSetCustom / rcAllocSetCustom.
- RamHandler records recent log entries in memory to aid post-mortem analysis.

Error and exit flow
- main catches TCLAP exceptions and prints brief CLI errors.
- On shutdown GameMain ensures save_file_.ExecuteQueuedWrite() is called to flush queued save writes.
- Engine::Dispose() and delete engine ensure resources are released; profiler_context.Dispose() finalizes profiler.

Build/run examples
- Development run (reads config defaults):
  - ./Build/overgrowth
- Load a level on startup:
  - ./Build/overgrowth --level levels/test_level
- Headless (no rendering) run:
  - ./Build/overgrowth --disable-rendering
- Clear caches dry-run:
  - ./Build/overgrowth --clear-cache-dry-run

Troubleshooting & debugging tips
- If the program exits early with logging only: inspect the file log (GetLogfilePath()) and console handler output.
- Enable debug output: -d to get additional log messages.
- To reproduce crashes with symbolized stack traces on Windows, build with OG_DEBUG and ensure dbghelp is available.
- For high CPU usage: check config_.limit_fps_in_game() and vSync settings — when vSync is off the loop may spin if not capped.

Where to look next in the codebase
- Engine lifecycle and state machine: Source/Main/engine.h / engine.cpp
- Graphics bootstrap / SwapToScreen: Source/Graphics/graphics.cpp
- Logging: Source/Logging/*
- Profiling/STIMING macros: Source/Timing/*
- Config and environment setup: SetUpEnvironment (search in Source/Internal or Main)

Documentation & API
- main.cpp is an implementation file; there is no header to generate API for. Document usage and CLI here in docs/modules/main.md and link to it from docs/index.md and docs/getting_started.md.

Maintenance notes
- Add any new CLI options in this doc when you change main.cpp.
- Keep examples in docs/getting_started.md in sync with available flags.