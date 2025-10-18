# TODO.md — ImGui as submodule, clean vendorization

## Goal

Integrate `imgui` via submodule without editing its tree. Stage needed files for build.

---

## Plan (choose one)

- [ ] **Preferred**: Handle layout in build system (CMake).  
- [ ] Custom `submodule.<name>.update` command that copies files out.  
- [ ] Wrapper fork that preserves layout.  
- [ ] Git subtree and rearrange in-tree.

---

## Track submodule

- [ ] Add submodule

  ```sh
  git submodule add https://github.com/ocornut/imgui.git 3rdparty/imgui
  git submodule update --init --recursive
  ```

- [ ] Pin a commit (recorded by Git).

---

## Option A — Build-only integration (preferred)

### CMake

- [ ] Add library with required sources

  ```cmake
  # CMakeLists.txt
  set(IMGUI_DIR ${CMAKE_SOURCE_DIR}/3rdparty/imgui)

  add_library(imgui STATIC
    ${IMGUI_DIR}/imgui.cpp
    ${IMGUI_DIR}/imgui_draw.cpp
    ${IMGUI_DIR}/imgui_tables.cpp
    ${IMGUI_DIR}/imgui_widgets.cpp
    ${IMGUI_DIR}/backends/imgui_impl_win32.cpp
    ${IMGUI_DIR}/backends/imgui_impl_dx11.cpp
  )
  target_include_directories(imgui PUBLIC
    ${IMGUI_DIR}
    ${IMGUI_DIR}/backends
  )
  ```

- [ ] If a mirrored layout is required, copy at configure/build time (never commit):

  ```cmake
  file(MAKE_DIRECTORY ${CMAKE_BINARY_DIR}/vendor/imgui)
  file(COPY
    ${IMGUI_DIR}/imgui.h
    ${IMGUI_DIR}/imconfig.h
    DESTINATION ${CMAKE_BINARY_DIR}/vendor/imgui
  )
  ```

### Project includes

- [ ] Include paths point to `3rdparty/imgui` or `${CMAKE_BINARY_DIR}/vendor/imgui`.

---

## Option B — Custom submodule update command

### Repo layout

- [ ] Keep submodule clean. Copy into `vendor/imgui/` outside the submodule.

### `.gitmodules`

- [ ] Add custom update command

  ```ini
  [submodule "3rdparty/imgui"]
      path = 3rdparty/imgui
      url  = https://github.com/ocornut/imgui.git
      update = !sh -c 'git -C "$1" checkout "$2" && ./scripts/vendorize-imgui.sh "$1"' -
  ```

### Script

- [ ] Create `scripts/vendorize-imgui.sh`

  ```sh
  #!/usr/bin/env sh
  set -euo pipefail
  sub="$1"
  mkdir -p vendor/imgui
  rsync -a --delete     "$sub/imgui.h" "$sub/imconfig.h"     "$sub/imgui.cpp" "$sub/imgui_draw.cpp" "$sub/imgui_tables.cpp" "$sub/imgui_widgets.cpp"     "$sub/backends/imgui_impl_win32."* "$sub/backends/imgui_impl_dx11."*     vendor/imgui/
  ```

- [ ] Make executable: `chmod +x scripts/vendorize-imgui.sh`
- [ ] Run: `git submodule update --init --recursive`

---

## Option C — Wrapper fork

- [ ] Fork `imgui`.
- [ ] Add repo-level CMakeLists or layout you prefer.
- [ ] Point submodule URL to your fork.
- [ ] Periodically merge upstream.

---

## Option D — Git subtree

- [ ] Add:

  ```sh
  git subtree add --prefix 3rdparty/imgui https://github.com/ocornut/imgui.git master --squash
  ```

- [ ] Rearrange files as needed in-tree.
- [ ] Update:

  ```sh
  git subtree pull --prefix 3rdparty/imgui https://github.com/ocornut/imgui.git master --squash
  ```

---

## CI

- [ ] Cache submodules:

  ```sh
  git submodule sync --recursive
  git submodule update --init --recursive --depth 1
  ```

- [ ] Verify build of `imgui` target.
- [ ] If using Option B, run `git submodule update` step so vendor files exist.

---

## Tests

- [ ] Compile a minimal app that creates an ImGui context and a frame.
- [ ] Link against `imgui`.
- [ ] Smoke-test backends (Win32 + DX11 here).

---

## Maintenance

- [ ] Document the chosen option in `CONTRIBUTING.md`.
- [ ] Lock the submodule commit in release branches.
- [ ] Re-run vendorize script or reconfigure build after updates.

---

## Do not

- [ ] Do not rename or move files inside the submodule.
- [ ] Do not rely on Git hooks for portability.
- [ ] Do not commit build-output vendor copies. Add to `.gitignore`.

---

## Minimal setup checklist

- [ ] Add submodule at `3rdparty/imgui`.
- [ ] Wire it via CMake (Option A).  
- [ ] If you need a `vendor/` mirror, copy during configure/build only.
