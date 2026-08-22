# Godot Agent API Reference

Status: implemented `GodotClient` API version `v1` for Godot 4.x.

This is the executable capability index for an AI agent. It covers engine
installation first, then every supported public host call, then the native
GDScript contract a generated game builds on.

## Hard API Boundary

The only supported Python entry point is:

```python
from engine_adapters.godot import GodotClient
```

Do not import `engine_adapters.godot._internal`, call adapter-owned GDScript
directly, edit `.a3game` state, construct generated-output paths, or make a
generated game depend on `engine_adapters/godot/examples/` at runtime.

## Execution Authority

The game-generation Agent generates engine-native test source. The Agent MUST
NOT invoke `godot.testing.*` or declare benchmark success.

Engine execution and evaluation code owns builds, native test execution,
runtime evidence, and benchmark results. The generation Agent must not invoke
`godot.testing.*`; the later Execution/Assembly authority may invoke it through
`GodotClient`. A zero process return code alone is not success: a Godot
`--import` or `--export` run that exits zero is still a failure when the
resource does not subsequently load, and a test report must contain matching
passing cases.

## Engine Installation

An agent must establish a verified Godot executable before creating a project,
importing an asset, or launching a game. Installation is the one step that is
not a `GodotClient` call.

`scripts/engine_install/godot/` holds two different kinds of script. Only the
first row installs an engine; the rest are thin wrappers over the adapter CLI
and require an already-installed editor.

| Script | Role | What it actually runs |
| --- | --- | --- |
| `install.sh` / `install.cmd` | **Engine installation** — the AI-automatable installer | `install.py`: resolves, SHA-512 verifies, extracts, probes, and publishes an official Godot 4 release |
| `create_project.sh` / `.cmd` | Project construction | `python -m engine_adapters.godot ... create-project` |
| `import_asset.sh` / `.cmd` | Generated-asset import | `scripts/import_generated_asset.py --engine godot` |
| `run.sh` / `.cmd` | Launch | `python -m engine_adapters.godot ... launch-game` |

### Automated installation sequence

Run this before any other call. It is non-interactive: no prompt, GUI, package
manager, elevated permission, or shell-profile edit.

1. `install.sh --dry-run --json` (Windows: `install.cmd`) — record the resolved
   release asset, architecture, official URL, and target paths without touching
   the network or filesystem.
2. Rerun without `--dry-run`. The installer first probes `--executable`,
   `A3GAME_GODOT_EXECUTABLE`, `A3GAME_GODOT`, legacy `AAAGF_GODOT`, then
   `godot4`/`godot`/`godot-mono` on PATH, and reuses only the pinned version.
3. Parse the JSON and require exit code zero, `ok=true`, and a `verified_version`
   matching the exact requested version. Never infer success from a downloaded
   archive or from a nonzero-length file.
4. Source the emitted `.env` (POSIX) or call the emitted `.cmd` (Windows), or
   set `A3GAME_GODOT_EXECUTABLE` directly to the returned executable path.
5. Set `A3GAME_GODOT_PROJECT`, then create and validate the project through the
   public client.

```bash
scripts/engine_install/godot/install.sh \
  --version 4.5.1 \
  --install-root "$PWD/.tools/godot" \
  --bin-dir "$PWD/.tools/bin" \
  --json
export A3GAME_GODOT_EXECUTABLE="$PWD/.tools/bin/godot4"
export A3GAME_GODOT_PROJECT=/projects/MyGame
scripts/engine_install/godot/create_project.sh --name MyGame
python3 -m engine_adapters.godot --project "$A3GAME_GODOT_PROJECT" validate
```

| Installer property | Behavior the agent may rely on |
| --- | --- |
| Version selection | Exact stable `4.x[.y]`; default `4.5.1-stable`; `latest`, prereleases, and non-4 versions are rejected |
| Architecture | Linux x86-64/x86-32/arm64/arm32, macOS universal, Windows x64/x86/arm64 |
| Source | Official `github.com/godotengine/godot/releases` assets over HTTPS only |
| Integrity | Official `SHA512-SUMS.txt`, exactly one matching entry, full-archive hash, fail closed on absence or mismatch |
| Extraction | Rejects absolute/traversing/duplicate paths, symbolic links, and special nodes before extracting |
| Atomicity | Stages to a sibling tree, probes the staged binary, publishes by rename, restores the prior target if replacement fails |
| Idempotence | Reuse reports `action=reused-managed`; `--version 4.6.0` installs beside `4.5.1`; `--force` replaces only that resolved target |
| Post-install gate | Runs the installed executable and requires the exact requested Godot 4 version |

The installer and adapter use the Python 3.8+ standard library. They require no
pip package, no engine SDK, and **not** Python 3.12. Compatibility is gated by
the full Godot adapter suite on Python 3.8.10. Platform details, all flags, and
failure semantics are in `scripts/engine_install/godot/README.md`.

## Result Contract

Except `api_version` and `runtime.sessions.probe`, every public operation returns
a strict-JSON dictionary with exactly these top-level fields:

| Field | Meaning |
| --- | --- |
| `ok` | Boolean success; never infer success from process exit alone |
| `operation` | Stable operation name |
| `artifacts` | Produced/selected artifact records |
| `diagnostics` | Structured engine diagnostics |
| `warnings` | Non-fatal limitations, including documented local runtime fallback |
| `errors` | Fatal messages; non-empty on failure |
| `payload` | Operation-specific values described below |

`runtime.sessions.probe(timeout)` is the low-level UDP reachability record and
returns `ok`, `reachable`, response/error details, and request matching evidence.

## Client

```python
godot = GodotClient(
    project_path="/projects/MyGame",       # directory or project.godot
    godot_executable="/opt/godot/godot4",  # optional after PATH setup
    api_version="v1",
    runtime_host="127.0.0.1",
    runtime_port=30050,
    editor_timeout=300,
    import_timeout=300,
)
```

Constructor precedence and state:

| Setting | Resolution |
| --- | --- |
| Project | argument -> `A3GAME_GODOT_PROJECT` -> legacy `AAAGF_GODOT_PROJECT` |
| Executable | argument -> `A3GAME_GODOT_EXECUTABLE` -> `A3GAME_GODOT` -> legacy `AAAGF_GODOT` -> PATH discovery |
| Runtime | arguments -> `A3GAME_GODOT_RUNTIME_HOST` / `A3GAME_GODOT_RUNTIME_PORT` -> `127.0.0.1:30050` |
| Timeouts | arguments -> `A3GAME_GODOT_EDITOR_TIMEOUT` / `A3GAME_GODOT_IMPORT_TIMEOUT` -> 300 seconds |
| Private state | `A3GAME_GODOT_DATA_ROOT` -> `<project>/.a3game` |
| Artifact registry | `A3GAME_GODOT_ARTIFACT_REGISTRY` -> `<data-root>/artifacts.json` |
| World registry | `A3GAME_GODOT_WORLD_REGISTRY_ROOT` -> `<data-root>/worlds` |

All managed path components must be ordinary directories and all leaves regular
files. Symbolic links, special nodes, path escapes, malformed strict JSON,
`NaN`, and infinity fail closed.

The signatures below are authoritative for `v1`. `E0`-`E8` refer to runnable
examples under **Runnable Call Patterns**.

| Call | Purpose and successful return | Failure behavior | Example |
| --- | --- | --- | --- |
| `godot.api_version -> str` | Returns literal `v1` | Constructor rejects unsupported API versions | E0 |
| `godot.get_environment_info(*, probe_version=True)` | Project/executable paths, existence, engine version support, runtime endpoint and registry paths | Missing/wrong executable or unsafe state paths are represented in the result | E0 |

## Project

| Call | Purpose and successful return | Failure behavior | Example |
| --- | --- | --- | --- |
| `godot.project.get_info(*, probe_version=True)` | Project marker, main scene, executable and optional `--version` evidence | Probe errors are structured; no state mutation | E0 |
| `godot.project.create(project_path=None, *, project_name="", renderer="gl_compatibility", overwrite=False, dry_run=False)` | Creates a minimal Godot 4 `project.godot`, main scene and import roots; artifacts list written paths | Rejects unsupported renderer, linked/special targets, existing managed files without `overwrite`, partial/unsafe layouts | E1 |
| `godot.project.validate(*, check_engine=True)` | Validates marker/main scene; with engine check, Godot loads and instantiates the resolved `PackedScene` | Missing/unloadable/UID-invalid scene, non-Godot-4 executable, import/load diagnostics or unsafe paths fail | E1 |

## Assets

`source` is a mapping containing `{game_id, run_id, task_kind, task_id,
artifact_key}`. Each identity is one safe path component below repository
`OUTPUT_ROOT`; callers cannot provide an arbitrary generated-output path.
`options` is strict-JSON metadata forwarded to import handling; common options
include `replace_existing` and import hints.

| Call | Purpose and successful return | Failure behavior | Example |
| --- | --- | --- | --- |
| `godot.assets.import_asset(source, asset_type, *, destination="", options=None)` | Resolves, stages, imports, natively inspects and registers any supported type; artifact contains real `res://` path/class/capabilities | Unknown type/identity, source escape, format mismatch, conflict, import/load error or native contract mismatch rolls back files/cache/registry | E2 |
| `godot.assets.import_avatar(source, **kwargs)` | Typed `avatar` helper; accepts `destination` and `options`; returns registered skinned `PackedScene` | Requires mesh, Skeleton3D bones and a skeleton-linked skinned mesh | E2 |
| `godot.assets.import_motion(source, *, skeleton="", destination="", avatar_name="", options=None)` | Typed motion import with optional required live Skeleton3D path/avatar label | Requires a `PackedScene`, animation and bone-targeted track compatible with requested skeleton | E3 |
| `godot.assets.import_scene(source, **kwargs)` | Typed spawnable scene helper; accepts `destination`/`options` | Native resource must load and instantiate as `PackedScene` | E2 |
| `godot.assets.import_prop(source, **kwargs)` | Typed prop helper; accepts `destination`/`options` | Requires instantiable `PackedScene` with `MeshInstance3D`; bare OBJ fails | E2 |
| `godot.assets.import_weapon(source, **kwargs)` | Typed weapon helper; accepts `destination`/`options` | Same spawnable mesh contract as prop | E2 |
| `godot.assets.import_material(source, **kwargs)` | Typed Godot material/image import; accepts `destination`/`options` | Wrong native class, decode/import error or unsafe target fails | E2 |
| `godot.assets.import_texture(source, **kwargs)` | Typed `Texture2D` import; accepts `destination`/`options` | Corrupt/unsupported image, missing imported resource or wrong class fails | E2 |
| `godot.assets.import_effect(source, **kwargs)` | Typed effect resource import; accepts `destination`/`options` | Wrong source/target/native load contract fails | E2 |
| `godot.assets.import_audio(source, **kwargs)` | Typed `AudioStream` import; accepts `destination`/`options` | Decode/load/class mismatch fails | E2 |
| `godot.assets.validate(source, asset_type, *, destination="", options=None)` | Runs source identity, format, destination and conflict preflight without copying | Returns structured path/type/source errors and makes no changes | E2 |
| `godot.assets.resolve_source(source, *, asset_type="")` | Resolves trusted task metadata/artifact path and returns source details | Missing metadata/key/file, identity mismatch, link or OUTPUT_ROOT escape fails | E2 |
| `godot.assets.list(asset_type="", *, root="assets/imported")` | Lists in-project imported resources below a contained root | Traversal, external root, linked/special entries or unreadable project fails | E2 |
| `godot.assets.list_registered(asset_type="")` | Returns registry artifacts and filtered count | Malformed/linked/special/non-strict registry fails without replacement | E2 |
| `godot.assets.get_metadata(artifact_id)` | Returns one registered artifact and metadata | Unknown ID or invalid registry fails | E2 |
| `godot.assets.register_resource(*, resource_path, asset_type, asset_id, source_path="", backend_class="", spawnable=None, metadata=None)` | Natively loads an existing canonical in-project `res://` resource, derives class/spawnability, and registers it; optional claims act as assertions | Traversal/link/missing resource, wrong type/class/spawnability, load/instantiate failure or ambiguous registry upsert fails without writing | E2 |

Supported asset types and the native class each import must produce:

| `asset_type` | Default destination | Required native result |
| --- | --- | --- |
| `avatar` | `assets/imported/avatars` | `PackedScene` with mesh, Skeleton3D bones, and a skeleton-linked skin |
| `motion` | `assets/imported/motions` | `PackedScene` with an animation and a bone-targeted track |
| `scene` | `assets/imported/scenes` | Instantiable `PackedScene` |
| `environment` | `assets/imported/environments` | Instantiable `PackedScene` |
| `prop` | `assets/imported/props` | Instantiable `PackedScene` containing `MeshInstance3D` |
| `static_mesh` | `assets/imported/props` | Instantiable `PackedScene` containing `MeshInstance3D` |
| `weapon` | `assets/imported/weapons` | Instantiable `PackedScene` containing `MeshInstance3D` |
| `material` | `assets/imported/materials` | `Material` (or an image Godot imports to one) |
| `texture` | `assets/imported/textures` | `Texture2D` |
| `effect` | `assets/imported/effects` | Loadable effect resource |
| `audio` | `assets/imported/audio` | `AudioStream` |

glTF main files and local buffers/images are one transaction. Failed native
validation restores replaced source files, adjacent `.import` files, matching
`.godot/imported` cache files, and the prior registry.

## Animation

| Call | Purpose and successful return | Failure behavior | Example |
| --- | --- | --- | --- |
| `godot.animation.import_motion(source, *, skeleton, destination="", avatar_name="", options=None)` | Imports a motion and requires the named live Skeleton3D NodePath | Empty/missing skeleton, absent animation/bone track or mismatched target fails/rolls back | E3 |
| `godot.animation.resolve_skeleton(avatar)` | Reloads registered avatar and returns live Skeleton3D paths/bones | Unknown/non-avatar/unloadable/unskinned resource fails | E3 |
| `godot.animation.validate_compatibility(motion, skeleton)` | Reloads motion and verifies animation plus bone track targeting the requested live skeleton | Structural mismatch fails; does not claim visual retargeting quality | E3 |

## Bindings

| Call | Purpose and successful return | Failure behavior | Example |
| --- | --- | --- | --- |
| `godot.bindings.bind_pbr_material(*, asset_id, source, mesh_assets, destination="assets/imported/materials", options=None)` | Imports/creates material, applies `material_override` through Godot to every target mesh, saves bound scenes, atomically retargets records and returns manifest/artifacts | Missing target/material, zero changed meshes, engine error, unsafe state, duplicate target or any commit failure restores material/scenes/texture/manifest/registry | E3 |

Binding runs an adapter-owned SceneTree script and succeeds only after bound
`PackedScene` files with changed `MeshInstance3D` nodes are reported and
persisted. A binding that changes no mesh is a failure, not a no-op.

## World

| Call | Purpose and successful return | Failure behavior | Example |
| --- | --- | --- | --- |
| `godot.world.build(source, *, options=None)` | Imports a scene then creates, validates and optionally publishes a World according to options | Any asset/draft/native validation failure is returned; no fabricated package | E4 |
| `godot.world.create_draft(spec, *, draft_id="", project_id="", metadata=None)` | Writes a versioned draft pointing to a ready registered scene; explicit IDs override spec and explicit metadata merges over spec | Wrong schema/JSON, unsafe registry, missing/non-spawnable/non-`PackedScene` scene or native load failure makes no valid draft | E4 |
| `godot.world.validate_draft(draft_id)` | Revalidates record/file identity, lifecycle and current registered/native scene | Unknown/damaged/version-mismatched/stale draft fails | E4 |
| `godot.world.publish_draft(draft_id)` | Atomically publishes a validated package and returns package artifact | Only validated drafts publish; linked/special paths, collision or stale scene fail | E4 |
| `godot.world.list_packages(*, project_id="", world_id="")` | Returns strict packages filtered by either/both IDs | One damaged record fails the read; records are never silently skipped/synthesized | E4 |

## Plugin

| Call | Purpose and successful return | Failure behavior | Example |
| --- | --- | --- | --- |
| `godot.plugin.install(source, *, install_dir="", replace_existing=False, enable=True, dry_run=False)` | Installs a registered generated add-on, natively validates `plugin.cfg`/entry, optionally enables it and returns copied files | Source escape/link/special node, unsafe name/script, missing metadata, non-`@tool EditorPlugin`, existing target or ambiguous settings fails atomically | E5 |
| `godot.plugin.install_framework(*, replace_existing=False, enable=True, dry_run=False)` | Installs adapter-owned `A3GamePlayable` v1.1.0, enables plugin and `A3GameRuntime` autoload; payload lists copied implementation/test files | Same atomic validation, target and settings rules; conflicting autoload requires explicit replacement | E5 |
| `godot.plugin.list()` | Lists safe installed add-on descriptors; framework gets distinct artifact type | Unreadable/linked entries are skipped with warnings; unsafe project root fails | E5 |

## Build

| Call | Purpose and successful return | Failure behavior | Example |
| --- | --- | --- | --- |
| `godot.build.project(*, preset, output_path, debug=False, pack_only=False, extra_args=(), allow_external_output=False, timeout=None, dry_run=False)` | Runs named `--export-release`, `--export-debug` or `--export-pack`; returns full committed sibling set and ownership evidence | Missing preset/output, timeout, engine diagnostic, protected/external/unmanaged/tampered output, bad manifest/proof or nested link fails while preserving prior build | E6 |

Builds require a project-owned preset in `export_presets.cfg`.

## Testing

Subject to **Execution Authority** above: the generation Agent writes
`test_*.gd` but does not invoke this namespace.

| Call | Purpose and successful return | Failure behavior | Example |
| --- | --- | --- | --- |
| `godot.testing.run_automation_tests(test_filter="", *, script="", test_root="res://tests", report_path="", timeout=None, dry_run=False)` | Runs adapter/default or explicit SceneTree runner; returns matched/passed/failed/skipped counts and validated cases | Unsafe runner/report/root, timeout, stale/missing/non-strict report, invalid schema/status or any failed case makes operation fail | E6 |

Native `test_*.gd` files under the selected root must extend a constructible
Godot type and return `bool` or a dictionary with boolean `ok` from `run_test()`.
Truthiness is never used for out-of-contract values. Reports publish from a
private sibling staging path only after schema validation.

## Runtime

| Call | Purpose and successful return | Failure behavior | Example |
| --- | --- | --- | --- |
| `godot.runtime.launch_editor(*, scene_path="", extra_args=(), dry_run=False)` | Starts configured editor in its own process group; returns managed process ID | Missing project/executable, unsafe scene or launch error fails | E7 |
| `godot.runtime.stop_editor(process_id)` | Stops only an editor launched by this client | Unknown/wrong process ID or bounded termination failure returns error | E7 |
| `godot.runtime.launch_game(*, scene_path="", headless=False, extra_args=(), dry_run=False)` | Starts project/main or selected scene; returns managed process ID | Same ownership/path/launch checks | E7 |
| `godot.runtime.stop_game(process_id)` | Stops only a managed game process group | Unknown/wrong PID or stop failure returns error | E7 |
| `godot.runtime.launch_player(build_path, *, extra_args=(), dry_run=False)` | Launches a regular exported executable/file and returns managed process ID | Linked/special/missing/non-executable build or launch error fails | E7 |
| `godot.runtime.stop_player(process_id)` | Stops only a managed exported player | Unknown/wrong PID or stop failure returns error | E7 |

## Runtime Sessions

| Call | Purpose and successful return | Failure behavior | Example |
| --- | --- | --- | --- |
| `godot.runtime.sessions.join(*, world_id="", participant_id="", user_id="", avatar_artifact_id="", idle_motion_artifact_id="", move_motion_artifact_id="", controller_kind="human", transform=None, parameters=None, require_runtime=False)` | Resolves optional assets, creates/reconnects controller/entity and returns session plus bridge evidence | Bad mapping/JSON/artifact/type, participant bound to different entity, reachable NACK/protocol error, or required unreachable runtime fails without local registration | E7 |
| `godot.runtime.sessions.leave(*, participant_id="", controller_id="")` | Deactivates current binding and returns session/bridge record | Unknown session or reachable NACK fails without false deactivation | E7 |
| `godot.runtime.sessions.heartbeat(controller_id)` | Updates/returns an active local session | Unknown, inactive or offline controller fails | E7 |
| `godot.runtime.sessions.apply_input(controller_id, *, move_x=0.0, move_y=0.0, run=False, jump=False, yaw=0.0, pitch=0.0, seq=0, require_runtime=False)` | Sends finite normalized input and records it only after acceptable bridge outcome | Non-finite values, unknown/inactive controller, reachable NACK/malformed/mismatched response or required unreachable bridge fails without state mutation | E7 |
| `godot.runtime.sessions.snapshot(*, world_id="")` | Returns defensive session list, count and active count, optionally one World | Invalid World conversion/registry state fails | E7 |
| `godot.runtime.sessions.reset_world(*, world_id="")` | Removes only resolved World (`world_001` default) locally/natively; idempotent counts returned | Reachable native rejection fails without local removal | E7 |
| `godot.runtime.sessions.clear_entity(*, participant_id="", controller_id="", entity_id="", destroy_actor=True)` | Removes all bindings for resolved entity; native result distinguishes matched and destroy-queued nodes | Non-boolean flag, unknown entity or reachable rejection fails without removal | E7 |
| `godot.runtime.sessions.probe(timeout=0.25)` | Raw UDP status/reachability/protocol evidence, not the seven-field result envelope | Timeout/unreachable/malformed/mismatched response is explicit in returned record | E7 |

With `require_runtime=False`, no UDP response allows documented local-only state
with a warning. Once any response arrives, NACK or malformed/mismatched protocol
data is fatal and cannot mutate local state. Reconnect preserves entity ID,
replaces the old controller, and emits `session_reconnected`; `session_left`
means control detached, not entity destruction.

## Reflection

| Call | Purpose and successful return | Failure behavior | Example |
| --- | --- | --- | --- |
| `godot.reflection.inspect_artifact(artifact_id, *, live=True, timeout=60.0)` | Returns registry metadata; live mode asks Godot for resource class, PackedScene nodes, skeletons, skins, animations/tracks and load evidence | Unknown artifact, bad registry, timeout, unloadable/corrupt/wrong resource or malformed native report fails | E8 |

## Observation

| Call | Purpose and successful return | Failure behavior | Example |
| --- | --- | --- | --- |
| `godot.observe.check_status(*, timeout=5.0, check_runtime=False)` | Reports project/executable/version and optional UDP runtime readiness | Invalid environment or requested runtime failure is explicit; no mutation | E8 |

## Runnable Call Patterns

### E0 — environment

```python
godot = GodotClient(project_path="/projects/MyGame")
assert godot.api_version == "v1"
environment = godot.get_environment_info(probe_version=True)
assert environment["ok"], environment["errors"]
```

### E1 — create and validate a project

```python
created = godot.project.create(project_name="MyGame", renderer="gl_compatibility")
assert created["ok"], created["errors"]
validated = godot.project.validate(check_engine=True)
assert validated["ok"], validated["errors"]
```

### E2 — resolve, validate, import, query, or register an asset

```python
source = {
    "game_id": "game101",
    "run_id": "run_001",
    "task_kind": "3d_object",
    "task_id": "crate",
    "artifact_key": "glb_path",
}
assert godot.assets.resolve_source(source, asset_type="prop")["ok"]
assert godot.assets.validate(source, "prop")["ok"]
imported = godot.assets.import_prop(source, destination="assets/imported/props")
assert imported["ok"], imported["errors"]
artifact_id = imported["artifacts"][0]["artifact_id"]
assert godot.assets.get_metadata(artifact_id)["ok"]
assert godot.assets.list_registered("prop")["ok"]

existing = godot.assets.register_resource(
    resource_path="res://scenes/arena.tscn",
    asset_type="scene",
    asset_id="arena",
    backend_class="PackedScene",  # assertion, never trusted inference
    spawnable=True,
)
assert existing["ok"], existing["errors"]
```

### E3 — motion and material binding

```python
motion = godot.animation.import_motion(
    source,
    skeleton="Character/Skeleton3D",
    avatar_name="hero",
)
assert motion["ok"], motion["errors"]
assert godot.animation.resolve_skeleton("hero_avatar")["ok"]
assert godot.animation.validate_compatibility(
    motion["artifacts"][0]["artifact_id"], "Character/Skeleton3D"
)["ok"]

bound = godot.bindings.bind_pbr_material(
    asset_id="hero_material",
    source=source,
    mesh_assets=["hero_avatar"],
)
assert bound["ok"], bound["errors"]
```

### E4 — World lifecycle

```python
draft = godot.world.create_draft({
    "draft_id": "arena_draft",
    "world_id": "arena",
    "scene_artifact_id": "arena",
})
assert draft["ok"], draft["errors"]
assert godot.world.validate_draft("arena_draft")["ok"]
assert godot.world.publish_draft("arena_draft")["ok"]
packages = godot.world.list_packages(world_id="arena")
assert packages["ok"], packages["errors"]
```

### E5 — framework and generated add-ons

```python
framework = godot.plugin.install_framework()
assert framework["ok"], framework["errors"]

generated_addon = godot.plugin.install({
    "game_id": "game101",
    "run_id": "run_001",
    "task_kind": "mechanic",
    "task_id": "gameplay_addon",
})
assert generated_addon["ok"], generated_addon["errors"]
assert godot.plugin.list()["ok"]
```

### E6 — native tests and export

```python
tests = godot.testing.run_automation_tests(
    test_root="res://tests",
    report_path="reports/godot-tests.json",
)
assert tests["ok"], tests["errors"]

build = godot.build.project(
    preset="Linux/X11",
    output_path="builds/game.x86_64",
)
assert build["ok"], build["errors"]
```

### E7 — runtime lifecycle and control

```python
launched = godot.runtime.launch_game(headless=False)
assert launched["ok"], launched["errors"]

joined = godot.runtime.sessions.join(
    world_id="arena",
    participant_id="player_1",
    avatar_artifact_id="hero_avatar",
)
assert joined["ok"], joined["errors"]
controller = joined["payload"]["controller_id"]
assert godot.runtime.sessions.apply_input(
    controller, move_y=1.0, run=True, yaw=0.2, seq=1
)["ok"]
assert godot.runtime.sessions.heartbeat(controller)["ok"]
assert godot.runtime.sessions.snapshot(world_id="arena")["ok"]
assert godot.runtime.sessions.leave(controller_id=controller)["ok"]
assert godot.runtime.stop_game(launched["payload"]["process_id"])["ok"]
```

### E8 — inspect and observe

```python
inspection = godot.reflection.inspect_artifact("hero_avatar", live=True)
assert inspection["ok"], inspection["errors"]
status = godot.observe.check_status(check_runtime=True)
assert status["ok"], status["errors"]
```

## A3GamePlayable Public GDScript Contract

Installed by `godot.plugin.install_framework()` to
`res://addons/a3game_playable/`, version `1.1.0`. The machine-readable
capability matrix is `capabilities.json`; the native smoke test is
`tests/framework_smoke.gd`.

This native GDScript contract is intentionally separate from the host-side
`GodotClient` contract above. `GodotClient` prepares and executes the project;
it is not a dependency inside the generated Godot add-on.

### Enums

`v1.1.0` defines no GDScript enums. Session and input state travel as strict
dictionaries so the UDP bridge and the host `GodotClient` share one wire shape.

### Data Types

| Type | Kind | Contract |
| --- | --- | --- |
| Normalized input | `Dictionary` | `A3GameInputState.normalize(raw)` returns `{ok, input}` or `{ok: false, error}`. `NUMERIC_FIELDS` = `move_x`, `move_y`, `yaw`, `pitch`, `timestamp` must be numeric **and finite**; `run`/`jump` must be booleans; `seq` must be integral. Accepted values are then clamped: `move_x`/`move_y` to `[-1, 1]`, `pitch` to `[-PI/2, PI/2]`. `yaw` is not clamped |
| Session record | `Dictionary` | Returned by `bind_entity` and the `session_*` signals; carries `world_id`, `participant_id`, `entity_id`, `controller_id` |
| Identity snapshot | `Dictionary` | `A3GameIdentity.snapshot()` / `A3GameRuntimeEntity.a3game_snapshot()` |
| Probe result | `Dictionary` | `{ok, hit, position, normal, collider, collider_id, shape, distance}` or `{ok, count, hits}`; `ok=false` carries `error` |

Runtime defaults: `A3GameRuntime.DEFAULT_HOST` = `127.0.0.1`,
`DEFAULT_PORT` = `30050`, `DEFAULT_WORLD_ID` = `world_001`.

### Interfaces

GDScript has no `interface` keyword. The framework expresses the controllable
entity contract structurally: a node is drivable when it provides these members,
which `A3GameRuntimeEntity` already implements.

| Member | Signature | Purpose |
| --- | --- | --- |
| `apply_a3game_input` | `(input_state: Dictionary) -> void` | Receive one normalized input frame |
| `configure_a3game_identity` | `(identity: Dictionary) -> Dictionary` | Adopt world/participant/entity IDs |
| `a3game_snapshot` | `() -> Dictionary` | Report current identity and last input |
| `clear_a3game_entity` | `() -> void` | Detach without destroying the node |

Check with `node.has_method("apply_a3game_input")` before binding. Do not
subclass `A3GameRuntimeEntity` merely to satisfy a type check.

### Components

| Class | Extends | Public surface |
| --- | --- | --- |
| `A3GameIdentity` | `Node` | `@export world_id`/`participant_id`/`entity_id`; `configure(values) -> Dictionary`; `snapshot() -> Dictionary` |
| `A3GameRuntimeEntity` | `Node3D` | `@export a3game_entity_id`/`a3game_world_id`/`a3game_participant_id`; `last_input`; the four interface methods above; signals `runtime_input(input_state)`, `input_rejected(error)` |

### Subsystems

| Class | Extends | Public surface |
| --- | --- | --- |
| `A3GameRuntime` (autoload, `runtime.gd`) | `Node` | `bind_entity(node, entity_id) -> Dictionary`; `find_entity(entity_id) -> Node`; `sessions_snapshot(world_id="") -> Array[Dictionary]`; `last_input_for(entity_id) -> Dictionary` |

`A3GameRuntime` signals: `session_joined(session)`,
`session_reconnected(previous_session, session)`, `session_left(session)`,
`input_received(entity_id, input_state)`, `world_reset(world_id)`.

`session_left` means control detached, not entity destruction.

### Engine Scaffolding

| Class | Kind | Public surface |
| --- | --- | --- |
| `A3GameInputState` | `RefCounted`, static | `normalize(raw_input: Variant) -> Dictionary` |
| `A3GameSceneLoader` | `RefCounted`, static | `instantiate_scene(resource_path: String, parent: Node) -> Dictionary` |
| `A3GameAnimationDirector` | `RefCounted`, static | `find_player(root) -> AnimationPlayer`; `play(root, animation_name, blend_seconds=0.15, speed=1.0) -> Dictionary`; `stop(root, keep_state=false) -> Dictionary` |
| `A3GameCollisionProbe` | `RefCounted`, static | `raycast(...)`, `overlap_sphere(...)` — see below |
| `A3GameVisualKit` | `RefCounted`, static | `pbr_material(...)`, `create_sun(...)`, `create_fill(...)` — see below |
| `A3GameHudLayer` | `CanvasLayer` | `set_title(value: String)`; `set_status(values: Dictionary)` |

### Choosing a Collision Primitive

`A3GameCollisionProbe` provides queries only. It never creates a collider, and
it never moves a body — placing `CollisionShape3D` and choosing a body type stay
gameplay-owned.

| Call | Signature | Returns | Use it for |
| --- | --- | --- | --- |
| `raycast` | `(world: World3D, from: Vector3, to: Vector3, collision_mask: int = 0xFFFFFFFF, exclude: Array[RID] = []) -> Dictionary` | `{ok, hit, position, normal, collider, collider_id, shape, distance}`; a clean miss is `{ok: true, hit: false, distance}` | Hitscan fire, ground/step probes, line of sight, interaction cursor |
| `overlap_sphere` | `(world: World3D, center: Vector3, radius: float, collision_mask: int = 0xFFFFFFFF, max_results: int = 32) -> Dictionary` | `{ok, count, hits}` | Explosion radius, aggro range, pickup proximity, melee arcs |

Both set `collide_with_areas` and `collide_with_bodies` to `true`, so an `Area3D`
trigger and a `StaticBody3D` wall both answer. Narrow with `collision_mask`
rather than filtering results afterwards.

Guard rails: `world` must not be null; `raycast` endpoints must differ;
`overlap_sphere` requires a finite positive `radius` and `1 <= max_results <= 256`.
A violated precondition returns `{ok: false, error: "..."}` — check `ok` before
reading `hit` or `hits`.

Picking the body type is a gameplay decision, not a framework one:

| Body | When |
| --- | --- |
| `CharacterBody3D` | Player and NPC movement you script directly (`move_and_slide`) |
| `RigidBody3D` | Objects the physics engine owns — debris, projectiles with mass, pinball |
| `StaticBody3D` | Level geometry that never moves |
| `Area3D` | Detection volumes that must not push anything |

### Making a Generated Game Look Good

`A3GameVisualKit` builds standard Godot 4 resources and returns them unparented.
The caller adds them to the tree, so the kit never dictates scene layout.

| Call | Signature | Returns |
| --- | --- | --- |
| `pbr_material` | `(color: Color, roughness: float = 0.45, metallic: float = 0.0, emission: Color = Color.TRANSPARENT) -> StandardMaterial3D` | `roughness`/`metallic` are clamped to `[0, 1]`; emission is enabled only when the emission color has a non-zero channel |
| `create_sun` | `(color: Color = Color("fff3d5"), energy: float = 1.0, rotation_degrees: Vector3 = Vector3(-55, -30, 0)) -> DirectionalLight3D` | Key light with `shadow_enabled = true`; `energy` is floored at `0` |
| `create_fill` | `(color: Color = Color("7e8dff"), energy: float = 2.0, radius: float = 12.0) -> OmniLight3D` | Fill light; `omni_range` is floored at `0.01` |

A workable default three-light-ish setup for a generated 3D scene is one
`create_sun()` key plus one or two `create_fill()` lights at contrasting hues —
the defaults are already a warm key against a cool fill. Add a
`WorldEnvironment` with ambient light for anything the two lights do not reach;
the kit deliberately does not create one, because environment and post-processing
choices belong to the game.

For imported assets, prefer `godot.bindings.bind_pbr_material(...)` on the host
side over constructing a material at runtime: the host path persists
`material_override` into the saved `PackedScene` and registers the result, so
the look survives a reload and is visible to World and Runtime.

## Framework Boundaries

`A3GamePlayable` provides runtime contracts only. It does not provide a concrete
character, controller, camera rig, HUD layout, weapon, vehicle, combat rule, or
game-specific input mapping. `capabilities.json` lists `movement`, `combat`,
`camera_rules`, `weapons`, `vehicles`, `scoring`, and `game_specific_ui` as
gameplay-owned.

Native Godot character/rigid-body physics, VFX, and audio also stay
gameplay-owned.

Generated projects own concrete gameplay implementation. The reference projects
under `engine_adapters/godot/examples/` are read-only references, not
dependencies or success criteria.

Current 3D bias, stated so an agent does not plan around capabilities that are
absent: `A3GameRuntimeEntity` extends `Node3D`, and `A3GameCollisionProbe` and
`A3GameVisualKit` take/return 3D types only. A 2D game uses the runtime,
identity, input-normalization, animation, scene-loading, and HUD parts of the
framework and owns its own 2D collision queries and lighting.

## Native Lifecycle Details

### Import

Imports copy only validated task artifacts under `res://assets/imported/`, run
`godot --headless --path <project> --import`, reject non-Godot-4 binaries,
nonzero exits and known corruption/parse/dependency-image errors even at exit
zero, then load the result with Godot. Spawnable mesh types must instantiate as
`PackedScene` with `MeshInstance3D`; avatars additionally need real skeleton
bones and linked skin. Bare OBJ loads as `ArrayMesh`, so use GLB/glTF or wrap it
in a scene.

### Builds and reports

Exports and test reports are staged privately, validated, and atomically
published. Builds record a signed ownership manifest and hashes of the complete
sibling set; the key stays under private adapter state. A later build may replace
only unchanged authenticated members. Project inputs, state keys, altered or
unmanaged outputs, edited manifests, links and special nodes fail closed.

### Examples and proof projects

`engine_adapters/godot/examples/` contains three complete native projects:

- `NeonDodge2D`: arcade survival, input, UI, monitored collisions and state loop;
- `SolarRally3D`: chase-camera racing, physical track, checkpoints, PBR materials,
  directional/omni light, lap/win loop;
- `OrbitPinball2D`: rigid-body pinball, static colliders, animated flippers,
  impulses, combo/lives loop.

Each has a real main `PackedScene`, deterministic unattended driver, manual
keyboard mode, and a Godot smoke script that advances live physics. Their
`mechanic_contract.json` maps to reviewer copies under
`<REPO_PATH>/test_data/outputs/<game>/godot/`.

## Coordinate System

Godot 3D is right-handed, Y-up, with `-Z` forward. glTF shares Y-up and metre
units. Record source facing/scale explicitly; do not apply a blanket conversion
after Godot has already imported the source format.
