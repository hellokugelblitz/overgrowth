# Architecture overview

Key subsystems (map to Source/*):
- Graphics: Source/Graphics — renderer, shaders, camera, VBO helpers.
- Game: Source/Game — entities, level, scripts.
- Asset: Source/Asset — loaders, manager, database.
- Physics: Source/Physics — collision, dynamics.
- Input: Source/UserInput — mouse/keyboard/controller.
- Main: Source/Main — engine bootstrap loop.
- Internal/Utility: timers, profiling, platform compatibility.

Guidelines:
- Public API lives in headers under Source/* — these are the Doxygen inputs.
- Keep examples under docs/tutorials/ and small runnable projects under Samples/.
