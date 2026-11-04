# 26.60 source-port candidate — NOT a verified DLL

## Status and scope

This checkout is a **source candidate** targeting **LeviLamina v26.60.0 client**.
The game-facing plugin is **uncompiled, unlinked and untested in Minecraft**.
No DLL was produced, installed or validated by this port. Do not reuse a 26.51
DLL, publish a release, or treat the dependency metadata as binary compatibility
proof. The mod version remains unchanged; this is not a release announcement.

Source evidence: `../LaminaSort-port-evidence/REPORT.md` and the local upstream
checkout `../LaminaSort-port-upstream-2660`, tag `v26.60.0`, commit
`072b9ffe0e9d9e0ec846f11d294880e892fe7770`. All paths below are relative to
that upstream checkout. The audit did not reproduce the reported registry crash;
an old-SDK/new-runtime ABI mismatch remains a hypothesis, not a confirmed cause.

## Changes and exact header checks

- `xmake.lua` pins `levilamina v26.60.0`, retaining client-only configuration.
  `tooth.json` declares client dependency `26.60.*`.
- The key callback passes its `IClientInstance&` into `SortSession::run`.
  The removed `_isCursorSelectedActive()` is replaced, not deleted without a
  guard. Checks happen before the inventory snapshot or any transfers.
- `src-client/mc/client/game/IClientInstance.h:275` declares virtual
  `LocalPlayer* getLocalPlayer() const`. A missing player aborts the sort.
- `src-client/mc/client/player/LocalPlayer.h:88` derives from `Player`;
  `src/mc/world/actor/player/Player.h:399` declares
  `ll::TypedStorage<8, 440, PlayerUIContainer> mPlayerUIContainer`.
  This is an **embedded object, not a nullable container pointer**.
  `src/ll/api/base/Alias.h:84-85` confirms `.get()` returns a reference.
- `src/mc/world/PlayerUIContainer.h:15` derives from `SimpleContainer`.
  `src/mc/world/SimpleContainer.h:25,29` declares virtual
  `ItemStack const& getItem(int) const` and `int getContainerSize() const`.
  If the UI container has no cursor slot (size <= slot index), sorting aborts
  without reading the item. Calls use virtual dispatch, not unavailable `$`
  direct thunks. There is no guessed offset, pointer cast, or old declaration.
- `src/mc/world/actor/player/PlayerUISlot.h:6` defines `CursorSelected = 0`.
  The guard uses that enum via an explicit integer cast, then refuses a stack
  with `!isNull() && mCount > 0`, matching the existing snapshot emptiness rule.
  `src/mc/world/item/ItemStackBase.h:84,267` declares `uchar mCount` and
  `bool isNull() const`; `ItemStack.h` supplies the derived stack definition.
- `SortSession.cpp` explicitly includes IClientInstance, LocalPlayer, Player,
  PlayerUIContainer, PlayerUISlot and ItemStack definitions. The existing
  `mc/client/multiplayer/ClientLevel.h` include and registry lookup are retained;
  the audit found no mandatory rename for those calls.

The cursor-slot check is header-backed, **not proven semantically equivalent**
to the removed helper. Drag distribution, progressive split, touch controls and
creative cursor/group state need runtime validation. In particular, the old
helper may have considered state beyond the UI cursor slot. Do not remove or
bypass this protection to get a build working.

## Clean Windows x64 build to perform (not performed here)

Use a separate Windows test checkout, Visual Studio 2022 C++ build tools and
Windows SDK, xmake, and **LLVM 22 clang-cl**. CI already requests LLVM major 22;
the README records LLVM 22.1.8 for the previous SDK, not a verified 26.60 build.
Open an x64 VS developer PowerShell; put your LLVM 22 `bin` first on PATH:

```powershell
# Replace this with your actual LLVM 22 installation directory.
$env:PATH = "C:\Program Files\LLVM\bin;$env:PATH"
where.exe clang-cl
clang-cl --version
xmake --version

# Run from the candidate checkout, not the old 26.51 checkout.
xmake repo -u
xmake f -c -y -p windows -a x64 -m release -o build-26.60 --target_type=client --trace=y
xmake clean -a
xmake -v -y
xmake build -y LaminaSortTests
xmake run LaminaSortTests
```

Confirm `clang-cl --version` actually reports LLVM 22 before configuring. Keep
verbose configure/build output and resolved package paths. Confirm the resolved
LeviLamina headers are **v26.60.0 client**, not cached 26.51 or server headers.
Upstream's Windows client branch requires `bedrockdata v26.60.5-client.5`;
verify that exact client data resolution and a matching game/runtime. If old
package resolution persists, use an isolated clean checkout/package cache and
resolve again; do not hand-edit cached SDK headers or reuse old objects/DLLs.
The separate build directory and cleaned configuration alone do not prove the
package cache is correct. Build or link failures are blockers, not permission
to drop safety checks.

The existing packer expects `bin/LaminaSort/` with `LaminaSort.dll` and
`manifest.json`. Inspect fresh timestamps and record the new DLL hash and build
provenance before testing. Check that the loaded DLL is this build, not another
copy elsewhere. These are future verification steps, **not artifacts supplied
by this source port**. Do not run release/upload/install commands as part of
source validation.

## Safe disposable-world acceptance checklist (all pending)

Use a separate test game/profile with a matching LeviLamina client and a **new
disposable local world**. Back up anything valuable first; do not test in a
production world or multiplayer inventory. Only use expendable items. Record
inventory/container item identities, counts, names, enchantments, durability,
locks and shulker contents before and after. Save logs/crash dumps; stop on any
loss, duplication, desync or unexpected operation.

- [ ] Verify SDK/client/game versions and loaded DLL path/hash from the clean
      Windows build; launch, load/unload/enable/disable without crashes.
- [ ] With an empty cursor, sort inventory, chest, barrel and shulker; confirm
      region selection, partial-stack consolidation, stable ordering and repeat
      sort idempotence. Hotbar/equipment/locked or out-of-region slots stay put.
- [ ] Hold a full stack and then a split/single item on the cursor. Press sort
      over inventory and each storage screen. It must skip, with no transfers,
      item movement or count changes; then place the item and retry normally.
- [ ] Repeat during left/right drag distribution and progressive split; test
      supported touch/controller interactions and creative picked-up/group
      cursor states. Any nonempty or active held-item state that sorts is a
      failed safety acceptance check, even if the cursor-slot guard passed.
- [ ] Type the sort key in creative search and anvil naming fields; no sort.
      Unsupported crafting/furnace/brewing/semantic slots remain excluded.
- [ ] Close/reopen screens repeatedly, change worlds, disconnect and return to
      menus; stale controllers must not be used. No player, no manager, or an
      unavailable UI cursor slot must fail closed without item reads/transfers.
- [ ] Exercise creative registry absent/empty/not-yet-received scenarios in a
      controlled test/debug environment; identifier fallback must not crash.
      Do not force this by corrupting live pointers or patching layouts.
- [ ] Inspect transfer/response and trace logs, verify final counts and metadata
      after reopening containers/reloading the disposable world, and capture a
      fresh stack/dump if the registry lookup or any controller path crashes.

## Verification performed on Linux

Using Debian g++ 14.2.0, the existing game-independent tests were rebuilt into a
scratch binary (not the pre-existing untracked `planner-tests`):

```sh
g++ -std=c++20 -Wall -Wextra -Werror -pedantic -Isrc \
  src/mod/sort/SortKey.cpp src/mod/sort/SortPlanner.cpp \
  tests/SortKeyTest.cpp tests/SortPlannerTest.cpp \
  -o ../LaminaSort-2660-planner-tests
../LaminaSort-2660-planner-tests
```

Result: `SortPlannerTest: all checks passed` and
`SortKeyTest: all checks passed` (exit 0).
A separate scratch-only test extracted the real pre-snapshot guard statements
and exercised nine manager/player/container/cursor cases with C++ test doubles;
it failed on the old removed API before the port and passed after the change.
That checks branch logic only: **neither it nor planner tests compile the SDK or
validate game ABI, link symbols, cursor semantics or inventory safety**.
This Linux environment has no xmake or clang-cl; no Windows plugin build or
runtime test was attempted. The uncompiled candidate status remains unchanged.
