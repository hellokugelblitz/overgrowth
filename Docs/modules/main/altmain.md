# altmain — auxiliary entry routines, tools and test runners

Location
- Source: Source/Main/altmain.cpp

Purpose
- Collection of alternative entry points, small utility tooling and test helpers used by the project outside the primary game loop.
- Provides helpers for: path compression into ZIPs, simple asset/test runners, a DDS converter CLI tool, and various small utilities used during development.

Primary functions
- CompressPathSet(PathSet, base_folder, dst)
  - Zip a set of files relative to base_folder into a single destination ZIP file.
  - Preserves relative paths inside the archive.
  - Overwrites the destination on the first entry and appends subsequent entries.

- const char* GetSuffix(const char* path)
  - Return the file suffix (characters after the last '.') for a path.
  - Calls FatalError() if no suffix is present.
  - Usage: used to dispatch file handling by extension.

- void LoadSoundGroup(const char* path, const char* data)
  - Currently disabled and intentionally asserts(false).
  - Historical placeholder for loading sound group XML data from in-archive sources.
  - Note: loading-from-zip responsibility belongs to the AssetManager — leave disabled unless reworking asset loading.

- void LoadSound(const char* path, const char* data)
  - Debug helper that constructs a SoundPlayInfo from a path and plays it immediately using the Sound API.
  - Sleeps one second (SDL_Delay(1000)) to allow playback to start; intended for quick manual tests.

- void LoadSoundGroupZipFile(const std::string& path)
  - Unzips the specified archive, enumerates entries and dispatches handlers by suffix:
    - 'xml' -> LoadSoundGroup (disabled)
    - 'wav' -> LoadSound
  - Useful for quick checks of packaged audio assets.

- void CheckCase(const char* path)
  - Validates case-correctness of a filesystem path using IsPathCaseCorrect().
  - Logs the corrected path if mismatch detected.

- int TestMain(int argc, char* argv[], const char* overloaded_write_dir, const char* overloaded_working_dir)
  - Lightweight developer test runner.
  - Calls SetUpEnvironment(...) to initialize paths and configuration.
  - Demonstrates synchronous FZX asset loading via Engine::Instance()->GetAssetManager()->LoadSync<FZXAsset>(...).
  - Iterates over FZX objects and logs metadata (labels, transforms, scales) and simple token checks.
  - Waits for user input before disposing environment; intended for interactive debugging.

- int DDSConvertMain(int argc, char* argv[], const char* overloaded_write_dir, const char* overloaded_working_dir)
  - CLI tool to convert image files to DDS format using Graphics/converttexture utilities.
  - For each path passed on the command line (argv[2..N]):
    - Converts image to <src>_converted.dds via ConvertImage and a temporary path.
  - Interactive: prints status and waits for keypress before exiting.
  - Invoked from main when the --ddsconvert flag is set.

How these are used
- main.cpp selects one of the alternative flows based on CLI options:
  - --ddsconvert => DDSConvertMain
  - Unit test builds may expose TestMain via RunUnitTests or similar helpers.
  - These helpers call SetUpEnvironment / DisposeEnvironment to reuse the runtime initialization used by the main game code.

Integration and dependencies
- Depends on project subsystems:
  - SetUpEnvironment/DisposeEnvironment for config & path setup.
  - Asset manager (Engine::Instance()->GetAssetManager()) for direct FZX loading in TestMain.
  - Graphics conversion utilities (Graphics/converttexture.h) for DDSConvertMain.
  - Zip utilities (Internal/zip_util.h) and ExpandedZipFile for archive handling.
  - SDL for delay/sleep and simple interaction.
- Because these are tooling helpers, they run with fewer safety/performance constraints than core engine code.

Notes and caveats
- LoadSoundGroup is intentionally disabled — do not re-enable without migrating functionality into AssetManager.
- TestMain uses blocking console input (getchar()) — not suitable for automated CI unless modified to skip waits.
- DDSConvertMain creates converted files in-place (appends "_converted.dds"); it uses a temporary path helper to avoid clobbering partial results.
- GetSuffix() will call FatalError() if a filename lacks a dot — callers should ensure inputs are valid or update GetSuffix to return nullptr on failure.

Testing and usage examples
- Convert images:
  - ./overgrowth --ddsconvert path/to/image.png path/to/another.jpg
- Run TestMain (developer build path):
  - Hook TestMain into a test runner or call the function directly from a small harness that sets write/working dir args.
- Inspect ZIP audio assets:
  - Build a small wrapper that calls LoadSoundGroupZipFile("path/to/archive.zip") from a debug binary.

Maintenance suggestions
- Move zip-based resource handling into AssetManager if on-the-fly archive loading is required in production.
- Replace blocking getchar() calls with a flag-driven exit for automated tests.
- Add unit tests for CompressPathSet and GetSuffix behavior (edge cases: empty paths, paths without suffix, base_folder not prefixing full_path).
- Add brief Doxygen-style comments if you want these helpers discovered in generated documentation (they are implementation-level helpers — narrative docs as written here are sufficient).

See also
- SetUpEnvironment / DisposeEnvironment (search in Source/Internal or Main)
- Asset manager: Source/Asset/assetmanager.h / assetmanager.cpp
- Graphics conversion utilities: Source/Graphics/converttexture.h
- Zip utilities: Source/Internal/zip_util.h