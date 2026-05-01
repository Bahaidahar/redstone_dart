# Setup — Redstone.Dart (SERVER_ONLY fork)

This fork patches Norbert515/redstone_dart to run **without the custom Flutter engine** (which is not published in the upstream repo). Mods built with this fork use Dart for **server-side game logic only** — blocks, items, commands, events, recipes, loot tables. Custom Flutter GUI rendering is **not** available.

## Prerequisites

| Tool | Version | Install |
|------|---------|---------|
| Dart SDK | 3.0+ | flutter-bundled or `brew install dart` |
| Java | 21+ | `brew install --cask temurin@21` |
| CMake | 3.21+ | `brew install cmake` |
| Git | any | preinstalled on macOS |

Check with:
```bash
dart --version
java -version
cmake --version
```

If Java is older than 21, switch JAVA_HOME:
```bash
export JAVA_HOME=$(/usr/libexec/java_home -v21)
export PATH="$JAVA_HOME/bin:$PATH"
```

Add to `~/.zshrc` to persist.

## 1. Clone

```bash
git clone https://github.com/Bahaidahar/redstone_dart.git
cd redstone_dart
```

## 2. Build the native bridge (server-only)

The native C++ bridge is what connects Java (Minecraft) to Dart. We build it without Flutter dependencies.

```bash
cd packages/native_mc_bridge
cmake -B build -DSERVER_ONLY=ON .
cmake --build build --config release
cd ../..
```

Output: `packages/native_mc_bridge/build/dart_mc_bridge.dylib` (macOS) / `.so` (Linux) / `.dll` (Windows).

## 3. Activate the patched CLI

```bash
dart pub global activate --source path packages/redstone_cli
```

Add the pub bin to PATH if not already:
```bash
export PATH="$PATH:$HOME/.pub-cache/bin"
```

Verify:
```bash
redstone doctor
```

Expected: all checks green except possibly `dart_dll` warning (auto-downloaded on first run).

## 4. Create a mod

```bash
cd ~/your_workspace
redstone create my_mod
cd my_mod
```

The CLI generates a project with `packages/server`, `packages/client`, `packages/common`. The client package uses Flutter — we will not use it.

### 4a. Configure for server-only mode

Edit `redstone.yaml`:
```yaml
name: my_mod
minecraft_version: "1.21.11"

packages:
  server: packages/server
  client: packages/client
  common: packages/common

entry_points:
  server: packages/server/lib/main.dart
  client: packages/client/lib/main.dart
  datagen: packages/server/lib/main.dart   # required: server entry as datagen target
```

Edit `pubspec.yaml`:
```yaml
redstone:
  minecraft_version: "1.21.11"
```

Edit `minecraft/gradle.properties`:
```properties
minecraft_version=1.21.11
loader_version=0.18.3
fabric_version=0.139.4+1.21.11
```

Edit `minecraft/src/main/resources/fabric.mod.json`:
```json
"depends": {
  "minecraft": "~1.21.11",
  ...
}
```

### 4b. Skip Flutter pub get failure (optional)

The CLI's `redstone create` runs `dart pub get` which fails because the client package depends on Flutter. This is fine for server-only mode — pub get only fails on the **root** workspace. The server package resolves on its own:

```bash
cd packages/server
dart pub get
cd ../..
```

## 5. Run the mod

```bash
REDSTONE_SERVER_ONLY=1 redstone run
```

The first run takes ~10 minutes (Gradle downloads Minecraft, mappings, fabric loader). Subsequent runs are fast.

You should see:
```
🔥 Redstone is running my_mod
REDSTONE_SERVER_ONLY=1 — skipping Flutter / dual-runtime
...
My Mod server mod ready!
  Blocks: N registered
  Items: M registered
```

Minecraft client window opens. Create a world, find your blocks in creative inventory, run `/hello` and `/heal` commands.

## 6. Hot reload

Press `r` in the terminal to push Dart code changes to the running mod.

## What works / doesn't work

**Works:**
- Custom blocks (`CustomBlock`)
- Custom items (`CustomItem`)
- Custom commands (`Commands.register`)
- Events (`Events.onPlayerJoin`, `onPlayerDeath`, `addTickListener`, etc.)
- Recipes (`Recipes.shaped`, `shapeless`, `smelting`)
- Loot tables (`LootTables.modify`)
- Custom entities (server logic; rendering uses vanilla mob models)
- Hot reload of Dart code
- `/hello`-style commands defined in Dart

**Does NOT work (requires custom Flutter engine):**
- Custom Flutter GUI screens
- Custom block-entity 3D Flutter rendering
- Anything in `packages/client/lib/main.dart`

**Known issues:**
- `RecipeManagerMixin: Could not access recipe map` — vanilla MC internals changed; recipes added via Dart are not injected into RecipeManager. Works in creative; survival crafting partially broken.
- `processAllSurfaceTasks UnsatisfiedLinkError` — client tries to call native Flutter surface JNI which isn't in the SERVER_ONLY dylib. Logs error but does not crash.
- `Realms auth failed` — using FabricMC dev account, not real Mojang login. Realms unavailable, single-player works.

## Updating from upstream

```bash
git fetch origin           # fetches Norbert515/redstone_dart
git checkout main
git rebase origin/main     # or git merge origin/main
# resolve conflicts in patched files
git push mine main
```

## Troubleshooting

**`Could not find Flutter SDK`** — env `REDSTONE_SERVER_ONLY=1` not set, or CLI was activated before patches applied. Re-run:
```bash
dart pub global activate --source path packages/redstone_cli
```

**`flutter_embedder.h not found`** — native bridge built without `-DSERVER_ONLY=ON`. Rebuild:
```bash
cd packages/native_mc_bridge
rm -rf build
cmake -B build -DSERVER_ONLY=ON .
cmake --build build
```

**`The language version 3.10 specified for the package 'matcher' is too high`** — test harness pubspec was generated before harness patch took effect. Delete and re-run:
```bash
rm -rf .redstone/test
REDSTONE_SERVER_ONLY=1 redstone run
```

**`Failed to find official mojang mappings for 26.1.2`** — wrong Minecraft version in `gradle.properties` / `redstone.yaml`. Use `1.21.11`.

**`Mod 'My Mod' requires version between 26.1.2 ... 26.2-`** — `fabric.mod.json` still has old version. Edit the `minecraft` dependency to `~1.21.11`.

## Architecture notes

The patches are minimal:

| File | Change |
|------|--------|
| `packages/redstone_cli/lib/src/commands/run_command.dart` | `REDSTONE_SERVER_ONLY=1` env disables Flutter prep + dual-runtime detection |
| `packages/redstone_cli/lib/src/runner/minecraft_runner.dart` | Use server entry + server package_config when env set |
| `packages/redstone_cli/lib/src/project/native_build_sync.dart` | Pass `-DSERVER_ONLY=ON` to cmake when env set |
| `packages/redstone_cli/lib/src/test/test_harness_generator.dart` | Pin `test`, `matcher`, `test_api`, `test_core` to dart_dll-compatible versions |

Why pin test packages? `dart_dll v0.2.0` embeds Dart 3.8. Newer pub.dev releases of `test`/`matcher` declare lang version 3.10 which the embedded VM rejects.

## Reference

- Upstream: https://github.com/Norbert515/redstone_dart
- Docs: https://docs.redstone-dart.dev
- This fork: https://github.com/Bahaidahar/redstone_dart
