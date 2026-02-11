# SceneGraph — world object manager, renderer front-end, and physics bridge

Location
- Implementation: Source/Main/scenegraph.cpp
- Header: Source/Main/scenegraph.h

Purpose
- Central runtime container for all scene Objects (EnvObject, MovementObject, TerrainObject, DecalObject, Hotspot, etc.).
- Coordinates object lifecycle (create, initialize, update, draw, dispose).
- Bridges rendering, physics and gameplay: performs frustum culling, draw batching, clustered light/ decal preparation, physics integration (Bullet), navmesh generation, and object messaging.

Audience
- Gameplay and rendering engineers working on object creation, culling, rendering, physics interactions, or level tooling.

High‑level responsibilities
- Maintain collections of objects by purpose: objects_, visible_objects_, collide_objects_, movement_objects_, visible_static_meshes_, decal_objects_, hotspots_, etc.
- Assign and map object IDs and provide GetObjectFromID()/DoesObjectWithIdExist().
- Provide update loop support: Update(timestep, curr_game_time) — physics, collisions, object updates, hotspot and particle updates, infrequent updates and sanity checks.
- Provide draw support: Draw(scene_draw_type) — static batching, instance draws, transparent pass, particles, flares, sky.
- Provide depth/shadow rendering: DrawDepthMap(...).
- Prepare clustered lighting and decal data for GPU: PrepareLightsAndDecals(...) + BindLights/BindDecals helpers and upload to buffer textures / UBOs.
- Manage navmesh creation, save/load, and integration: CreateNavMesh(), AddSceneToNavmesh(), SaveNavMesh(), LoadNavMesh().
- Manage runtime resources: PreloadShaders(), LoadReflectionCaptureCubemaps(), UnloadReflectionCaptureCubemaps(), Dispose().

Key data structures & concepts
- object_list / visible_static_meshes_: lists of Object* and EnvObject* used for different passes.
- object_from_id_map_: sparse vector mapping int IDs -> Object*.
- update_objects_ array + num_update_objects: hot array of objects that must be updated every frame (fast path).
- decal_tbo and decal_cluster_buffer: CPU-side buffers packed and uploaded to GPU for decals & clusters.
- Cluster grid: screen-space grid (cluster_size, num_z_clusters) for clustered lights & decals.
- ShaderClusterInfo / ShaderDecal / ShaderLight: tightly packed structures that must match shader layouts.
- UniformRingBuffer: transient UBO allocator used for ClusterInfo.

Important methods (what they do)
- SceneGraph::AddObject / LinkObject(Object*)
  - Insert object into scene lists; add to specific per-type arrays (terrain_objects_, movement_objects_, etc).
  - Call Initialize() on object; call GetShaderNames(preload_shaders) for shader preloading.
- SceneGraph::AssignID(Object*)
  - Gives stable IDs and writes to object_from_id_map_. ID 0 reserved for terrain.
- SceneGraph::GetObjectFromID(int)
  - Safe lookup; logs a warning and returns NULL if missing.
- SceneGraph::Update(float timestep, float curr_game_time)
  - Step physics (bullet_world_, plant_bullet_world_, abstract_bullet_world_), handle collisions, message object updates, particle updates, hotspots, navmesh updates and perform infrequent sanity updates.
- SceneGraph::Draw(SceneDrawType)
  - Full rendering pass: pre-sorts visible static meshes, frustum-culls, batches draws by model/shader, draws sky/flares, then non-static objects, transparent pass, decals, and particles.
- SceneGraph::DrawDepthMap(...)
  - Depth/shadow pass; supports different depth draw types and culls objects using provided cull planes.
- SceneGraph::PrepareLightsAndDecals(...)
  - Converts lights and decals into screen-space clusters, builds lookup buffers and uploads per-frame data to textures/UBOs for clustered shading. This is a critical performance hotspot.
- SceneGraph::CreateNavMesh(), AddSceneToNavmesh(), SaveNavMesh(), LoadNavMesh()
  - Build and persist navmesh from collision geometry and explicit navmesh hints and offmesh connections.
- SceneGraph::Dispose()
  - Tear down all objects, physics worlds, navmesh, particle system, sky, map editor and free GPU resources.

Usage patterns & examples
- Add a new object (pseudo):
  - obj = new SomeObject(...);
  - scenegraph->AssignID(obj);
  - scenegraph->LinkObject(obj);
  - if (!obj->Initialize()) { scenegraph->UnlinkObject(obj); delete obj; }
- Query by id:
  - Object* o = scenegraph->GetObjectFromID(id);
- Frustum checks:
  - Use camera helpers (camera->checkSphereInFrustum(...)) to pre-filter objects before expensive work.
- Navmesh:
  - Call CreateNavMesh() after scene is populated; use SaveNavMesh() to persist.

Performance notes & hotspots
- PrepareLightsAndDecals() is heavy (projection, cluster assignment, buffer assembly, GPU uploads). Avoid allocating inside its inner loops; reuse buffers.
- Static batching: visible_static_meshes_ are sorted and drawn in batches by model/shader to minimize state changes.
- update_objects_ is a fixed-size fast-path array used to avoid scanning whole objects_ every frame.
- PreloadShaders() collects shader names and creates programs early to avoid runtime shader compilation stalls.
- Uniform/texture buffer sizes must be checked against GL limits; code already queries GL_MAX_UNIFORM_BLOCK_SIZE and GL_MAX_TEXTURE_BUFFER_SIZE.

Threading & lifetime
- SceneGraph interacts with threaded subsystems (AssetManager, possibly background loaders) but methods that modify the scene (add/unlink objects, navmesh changes) must be called from the main thread or otherwise synchronized.
- Bullet worlds are updated in Update(), assumed to run on the main thread.
- Dispose() must run on main thread during shutdown.

Debugging & profiling tips
- Profiler zones (PROFILER_ZONE / PROFILER_GPU_ZONE) are sprinkled in hotspots — use profiler context (g_profiler_ctx) to find slow areas.
- Flags for runtime debug:
  - g_debug_runtime_disable_scene_graph_draw / _draw_depth_map / _prepare_lights_and_decals to isolate subsystems.
  - g_draw_collision toggles collision mesh drawing.
- SceneGraph::DumpState() writes a file with current object list for post-mortem inspection.
- Keep an eye on destruction_sanity and destruction_memory arrays: used to detect invalid accesses to recently destroyed objects.
- To debug culling/visibility, log or draw object spheres (eo->sphere_center_, eo->sphere_radius_) and camera frustum.

Pitfalls & gotchas
- Shader data structures must match GPU shader memory layout exactly (ShaderDecal, ShaderLight, ShaderClusterInfo). Changing fields requires updating shaders.
- Cluster packing uses float buffers and integer packing macros (SetCount/SetIndex): be careful when modifying sizes or grid component counts.
- Many loops assume visible_static_mesh_indices_ order; modifying that pipeline requires maintaining the sort invariants.
- GetSuffix/use of FindFilePath and config[] lookups are used throughout; missing or malformed level script params can alter rendering paths (e.g., custom shaders, decal toggles).

Maintenance suggestions
- Extract PrepareLightsAndDecals into smaller units and add unit tests for cluster indexing and buffer packing.
- Provide clearer API methods for pushing/removing objects that handle ID assignment and initialization atomically.
- Document shader-side layout expectations next to the C++ structs to avoid divergence.
- Consider migrating heavy per-frame allocations to persistent buffers returned from a memory arena for consistent frame-time behavior.

Related files
- Source/Main/scenegraph.h — API and data members
- Source/Graphics/* — rendering utilities, shaders, textures, models
- Source/Physics/* — Bullet wrappers and collision helpers
- Source/Objects/* — object implementations (EnvObject, MovementObject, TerrainObject, DecalObject, Hotspot, etc.)
- Source/AI/navmesh.* — navmesh creation & parameters
- Source/Main/engine.cpp — high-level engine loop that calls SceneGraph::Update/Draw

Where to read the code
- Start at SceneGraph::Draw and SceneGraph::Update for runtime behavior.
- Read PrepareLightsAndDecals to understand clustered lights/decals and GPU upload flow.
- Check LinkObject/UnlinkObject/AssignID/GetObjectFromID for lifetime and ID management.

If you want, I can:
- Add a short how-to page with examples for adding custom object types and ensuring they appear in the scenegraph and renderer.
- Extract a concise checklist of invariants (shader struct sizes, buffer offsets, GL limits) to include in repository CONTRIBUTING.md.