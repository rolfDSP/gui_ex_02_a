# gui_ex_02_a – wxWidgets GUI Example

A small cross-platform C++ desktop application built with **wxWidgets 3.2** and **CMake**.
It is developed under Linux (WSL) and can be built as

- a native **Linux** program (GTK 3), and
- a **Windows** program (cross-compiled from Linux with MinGW-w64).

---

## 1. What the program does

The main window ("GUI Example 01", 400 × 300 pixels) contains:

| Element | Description |
|---|---|
| **Menu bar** | `File` menu with `Test` and `Quit` |
| **"Your name:" input field** | Single-line text field with a hint text |
| **Greet** button (green) | Prints `Hello, <name>!` into the output box |
| **Clear** button (red) | Empties the output box |
| **Output box** | Multi-line, read-only text area; grows with the window |

### Usage

1. Type a name into the input field.
2. Press **Enter** or click **Greet** → `Hello, <name>!` appears in the output box.
   If the field is empty, `Please enter a name.` is shown instead.
   Leading/trailing spaces are removed; the input field is cleared afterwards.
3. Click **Clear** to empty the output box.

### Menu

| Menu entry | Shortcut | Action |
|---|---|---|
| File → Test | Ctrl+T | Calls `MainFrame::onTest()`, which writes `File -> Test was called.` into the output box |
| File → Quit | Ctrl+Q | Closes the window and ends the program |

The menu can also be operated with the keyboard: **Alt+F** opens the File menu, then **T** or **Q**.

---

## 2. Project structure

```
gui_ex_02_a/
├── main.cpp                     # Complete application source code
├── CMakeLists.txt               # Build script (finds or downloads wxWidgets)
├── CMakePresets.json            # Build presets for Linux and Windows
├── cmake/
│   └── mingw-w64-x86_64.cmake   # Toolchain file for the Windows cross-compile
├── build/<preset>/              # Build directories (generated, one per preset)
├── exe/
│   ├── linux/gui_ex_02_a        # Linux executable (generated)
│   └── windows/gui_ex_02_a.exe  # Windows executable (generated)
├── README.md
└── DOC.md                       # This document
```

`build/` and `exe/` are generated and can be deleted at any time; they are recreated by the build.

---

## 3. Prerequisites

Under Linux / WSL (Ubuntu):

```bash
sudo apt install build-essential cmake ninja-build   # compiler, CMake ≥ 3.28, Ninja
sudo apt install mingw-w64                           # only for Windows builds
sudo apt install libwxgtk3.2-dev                     # optional, see below
```

### Where wxWidgets comes from

`CMakeLists.txt` first looks for an **installed** wxWidgets ≥ 3.2:

- **Found** (e.g. `libwxgtk3.2-dev` installed) → the system library is used; linking is dynamic.
- **Not found** → wxWidgets **v3.2.6** is downloaded from GitHub with CMake `FetchContent`
  and compiled as a static library. It is stored per preset in
  `build/<preset>/_deps/wxwidgets-{src,build,subbuild}` (approx. 600 MB).
  The first build takes several minutes; later builds reuse it.

For **Windows builds** the system package can never be used (it is a Linux/GTK build), so
wxWidgets is always fetched and compiled for Windows (MSW port) in `build/windows-*`.

---

## 4. Building

All commands are run in the project directory.

### Available presets

| Preset | Target | Build type | Output |
|---|---|---|---|
| `linux-debug` | Linux | Debug | `exe/linux/gui_ex_02_a` |
| `linux-release` | Linux | Release | `exe/linux/gui_ex_02_a` |
| `windows-debug` | Windows x64 (MinGW) | Debug | `exe/windows/gui_ex_02_a.exe` |
| `windows-release` | Windows x64 (MinGW) | Release | `exe/windows/gui_ex_02_a.exe` |

All presets use the **Ninja** generator and the build directory `build/<preset>`.
Debug and release builds of the same platform write to the same `exe/` folder,
so the last build wins.

### First build (configure + build)

```bash
cmake --preset linux-release
cmake --build --preset linux-release

cmake --preset windows-release
cmake --build --preset windows-release
```

### Rebuild after code changes

```bash
cmake --build --preset linux-release
cmake --build --preset windows-release
```

Only changed files are recompiled. Running `cmake --preset …` again is only needed when
`CMakeLists.txt` was changed (e.g. a new source file added) or the `build/` folder was deleted.

### Running

```bash
./exe/linux/gui_ex_02_a           # Linux (WSLg shows the window on the Windows desktop)
./exe/windows/gui_ex_02_a.exe     # Windows build, can also be started from WSL
```

---

## 5. Build configuration details

### CMakeLists.txt

- C++ standard: **C++20** (required).
- wxWidgets components used: `core`, `base`.
- Executable output goes to `exe/<platform>` (`linux`, `windows`, `macos`)
  for all build types.
- `WIN32_EXECUTABLE TRUE`: on Windows the program starts without a console window.
  Consequence: `std::cout` output is not visible in the Windows build.
- **MinGW only:** linked with `-static -static-libgcc -static-libstdc++`, so the `.exe`
  needs no extra DLLs (libstdc++, libgcc, libwinpthread). Together with the statically
  linked wxWidgets, the `.exe` depends only on DLLs shipped with Windows and can be copied
  to any Windows PC as a single file.

### cmake/mingw-w64-x86_64.cmake (toolchain file)

- Target system: Windows, x86_64.
- Compilers: `x86_64-w64-mingw32-gcc-posix` / `-g++-posix` (POSIX thread model, so
  `std::thread` and `std::mutex` are available), resource compiler `x86_64-w64-mingw32-windres`.
- Libraries, headers and packages are searched **only** in `/usr/x86_64-w64-mingw32`,
  so the Linux wxWidgets is never picked up by mistake.

### Executable size

The Windows release `.exe` is about 14 MB because wxWidgets is linked statically.
It can be reduced considerably with:

```bash
x86_64-w64-mingw32-strip exe/windows/gui_ex_02_a.exe
```

---

## 6. Source code overview (`main.cpp`)

### `class App : public wxApp`

Application entry point. `wxIMPLEMENT_APP(App)` generates `main()` / `WinMain()`.
`OnInit()` creates the main window and shows it.

### `class MainFrame : public wxFrame`

The main window. Everything is set up in the constructor:

1. **Menu bar** – `wxMenuBar` with one `wxMenu` ("File"):
   - `ID_TEST` → "Test", bound to the member function `onTest()`
   - separator
   - `wxID_EXIT` (standard wx ID) → "Quit", bound to a lambda calling `Close()`
2. **Controls** on a `wxPanel`: label (`wxStaticText`), input field and output box
   (`wxTextCtrl`), two `wxButton`s.
3. **Colours** – `SetBackgroundColour` / `SetForegroundColour`:
   Greet = green `(46, 160, 67)`, Clear = red `(207, 34, 46)`, both with white text.
4. **Layout** with `wxBoxSizer`s:
   - row 1: label + input field (input stretches horizontally)
   - row 2: stretch spacer + Greet + Clear (buttons right-aligned)
   - row 3: output box (fills the remaining space)
5. **Events** bound with `Bind()`:

   | Event | Source | Handler |
   |---|---|---|
   | `wxEVT_MENU` | `ID_TEST` | `onTest()` |
   | `wxEVT_MENU` | `wxID_EXIT` | `Close()` |
   | `wxEVT_BUTTON` | Greet | `Greet()` |
   | `wxEVT_TEXT_ENTER` | input field | `Greet()` |
   | `wxEVT_BUTTON` | Clear | `output_->Clear()` |

#### Members

| Member | Type | Purpose |
|---|---|---|
| `ID_TEST` | `enum` (`wxID_HIGHEST + 1`) | Own menu ID; above `wxID_HIGHEST` to avoid clashes with wx IDs |
| `onTest(wxCommandEvent&)` | method | Handler for File → Test |
| `Greet()` | method | Reads and trims the name, writes the greeting, clears the input |
| `input_` | `wxTextCtrl*` | Name input field |
| `output_` | `wxTextCtrl*` | Output box |

Memory management: all controls are created with `new` and are owned by their parent
window; wxWidgets deletes them automatically when the window is closed.

---

## 7. Extending the program

### Add a menu entry

```cpp
// 1. new ID in the enum
enum { ID_TEST = wxID_HIGHEST + 1, ID_FOO };

// 2. menu entry in the constructor
fileMenu->Append(ID_FOO, "&Foo\tCtrl+F", "Status bar help text");

// 3. bind it
Bind(wxEVT_MENU, &MainFrame::onFoo, this, ID_FOO);

// 4. handler
void onFoo(wxCommandEvent &) { /* ... */ }
```

### Show a message box instead of writing to the output box

```cpp
wxMessageBox("Text", "Title", wxOK | wxICON_INFORMATION, this);
```

### Add a source file

Add it to `add_executable(gui_ex_02_a main.cpp other.cpp)` in `CMakeLists.txt`,
then run `cmake --preset …` again before building.

### Use this project as a template

Copy `CMakeLists.txt`, `CMakePresets.json` and the `cmake/` folder, then change the
project/target name `gui_ex_02_a` in `CMakeLists.txt`.

---

## 8. Platform notes

- **Button colours:** Windows (MSW) keeps the native button shape with the colour filled in;
  on Linux (GTK 3) the result depends on the desktop theme (gradients/hover effects possible).
- **Console output:** not visible in the Windows build (GUI subsystem); use the output box
  or `wxMessageBox` / `wxLogMessage` instead.
- **WSL:** Linux GUI programs are displayed on the Windows desktop via WSLg.
