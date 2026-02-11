# Engine (Engine::) — Detailed documentation

Location
- Header: Source/Main/engine.h
- Implementation: Source/Main/engine.cpp

Purpose
- Central application manager for the engine process.
- Owns and coordinates major subsystems: AssetManager, SceneGraph, Sound, GUI, FontRenderer, Animation/LipSync systems, ASNetwork, input handling and level loading.
- Manages engine states (menu, level, editor, scriptable UI), loading lifecycle, frame update & draw loops, and some platform/utility helpers.

Audience
- Engine integrators, gameplay programmers, and contributors who need to manipulate engine-level behavior (loading levels, changing states, accessing global subsystems, or modifying the main loop).

Overview and responsibilities
- Initialization/teardown: Initialize(), Dispose()
- Main loop steps: Update(), Draw()
- Subsystems accessors: GetSound(), GetAssetManager(), GetSceneGraph(), etc.
- State stack & queueing: Push/pop states via QueueState, PopQueueStateStack and queued_engine_state_.
- Level load flow: LoadLevel/QueueLevelToLoad/LoadLevelData/PreloadAssets and related helpers.
- Rendering helpers: DrawScene, SetViewportForCamera, LoadScreenLoop, DrawLoadScreen
- User & editor interactions: HandleRabbotToggleControls, InjectWindowResizeEvent, AvatarControlManager wrapper.
- Mod loading callback via ModLoadingCallback override ModActivationChange().

Engine lifecycle
1. Create Engine singleton (Instance()) — generally constructed before Initialize().
2. Initialize() — perform subsystem init, set up scenegraph and asset manager.
3. Per-frame:
   - Update(): input processing, state machine updates, loading progression, background tasks.
   - Draw(): set up viewports, call DrawScene for each view, posteffects handling and HUD.
4. Dispose(): orderly teardown of subsystems and resource release.

Engine states
- EngineState: encapsulates an id, a type (EngineStateType), optional campaign/path reference.
- Engine maintains a state_history and a queued_engine_state_ (deque) for asynchronous state changes.
- Use QueueState(...) to request a state push/pop. The engine consumes queued actions during Update().
- EngineStateAction supports push/pop/until semantics (pop until a type or id).

Common operations (examples)
- Query global subsystems:
  - AssetManager* am = Engine::Instance()->GetAssetManager();
  - SceneGraph* sg = Engine::Instance()->GetSceneGraph();
  - ThreadedSound* s = Engine::Instance()->GetSound();

- Queue a level to load:
  ```cpp
  EngineState st("levels/my_level", kEngineLevelState, Path("levels/my_level"));
  Engine::Instance()->QueueState(st);
  ```

- Push menu state:
  ```cpp
  EngineStateAction action;
  action.type = kEngineStateActionPushState;
  action.state = EngineState("menu/main", kEngineNoState);
  Engine::Instance()->QueueState(action);
  ```

- Pop state (simulate "back" pressed):
  ```cpp
  Engine::Instance()->PopQueueStateStack(true);
  ```

Threading & synchronization
- Loading jobs use a background thread; Engine exposes loading_mutex_ to coordinate finished_loading_time and loading flags.
- Many subsystems (sound, asset manager) are internally threaded (e.g., ThreadedSound, AssetManager) — use their thread-safe APIs.
- Engine methods that are called from loading threads should acquire appropriate locks or use the public queuing APIs to schedule main-thread changes.

Key members and meaning (selected)
- instance_ (static): singleton pointer accessible via Instance().
- queued_engine_state_ (deque<EngineStateAction>): queued state changes; safe to push from other threads via public QueueState wrappers.
- scenegraph_ (SceneGraph*): active scene graph pointer.
- asset_manager (AssetManager): owned AssetManager instance.
- sound (ThreadedSound): owned sound playback subsystem.
- loading_in_progress_, level_loaded_, finished_loading_time: loading lifecycle flags.
- avatar_control_manager: handles player avatar input mapping.
- draw_frame / frame_counter: frame tracking used for profiling & debug text.

Important methods (reference)
- Initialize(): prepare subsystems, load config, prepare for frames.
- Update(): main per-frame update. Processes queued states, input, scripted UI, level load transitions, and updates subsystems.
- Draw(): render the current frame. Calls DrawScene for actual world drawing and manages HUD / overlays.
- Dispose(): cleanup / free resources; called at shutdown.
- QueueState(const EngineState&), QueueState(const EngineStateAction&): request state changes to be applied by the engine.
- PopQueueStateStack(bool allow_game_exit): invoked by input handlers or UI to request a pop; respects allow_game_exit to permit the engine to exit if stack empties.
- DrawScene(DrawingViewport, PostEffectsType, SceneGraph::SceneDrawType): high-level renderer entry, called by Draw() with appropriate viewport/post effect mode.
- GetAvatarIds(vector<ObjectID>&): returns tracked avatar IDs for gameplay logic.
- SetGameSpeed(float val, bool hard): change global time scaling; useful for slow-motion effects.
- InjectWindowResizeEvent(ivec2 size): queue resize; engine coalesces frequent resizes (resize_event_frame_counter).

State queue semantics and best practices
- Always use QueueState or QueueStateAction instead of modifying state_history directly — the engine processes queued actions at deterministic times inside Update().
- For immediate-state needs inside the main thread (e.g., inside scripting callbacks executed synchronously), document and ensure ordering expectations.

Integration points & callbacks
- ModLoadingCallback override: ModActivationChange(const ModInstance*): invoked when a mod is activated/deactivated.
- StaticScriptableUICallback / ScriptableUICallback / NewLevel: hooks for script-driven UI and level creation.

Error handling & user prompts
- QueueErrorMessage(title, message): present an error popup via GUI system (useful for fatal load errors).
- Many functions set flags such as check_save_level_changes_dialog_is_showing — scripts & UI should consult these when deciding to block transitions.

Debugging & profiling tips
- Use Engine::draw_frame / frame_counter to annotate profiler ranges.
- PushGPUProfileRange(const char*) / PopGPUProfileRange() are helpers used across drawing code to label GPU timings.
- Set g_draw_collision to true to view collision debug geometry.
- Use GetShaderNames() to prewarm shader caches via CI or preload scripts to avoid runtime hitches.

Performance & culling
- Rendering uses per-camera viewports; SetViewportForCamera selects viewport geometry for split-screen.
- Frustum culling is delegated to the SceneGraph and Camera helpers; use DrawScene arguments to control culling level.

Pitfalls & gotchas
- Some Engine state modifications assume they run on the main thread. If code runs on loader threads, prefer queuing via QueueState.
- Many engine members are intentionally public or exposed as members to avoid allocations across frame hot paths (e.g., avatar arrays). Avoid re-allocating these each frame.
- Resizing is throttled by resize_event_frame_counter — rapid window resizes will be coalesced; expect a small delay.