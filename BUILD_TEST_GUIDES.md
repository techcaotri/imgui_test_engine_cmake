# ImGui Test Engine - CMake Build Instructions

This guide explains how to build and test the imgui_test_engine project using CMake on Linux.

## Prerequisites

Install the required dependencies on Ubuntu/Debian:

```bash
sudo apt-get update
sudo apt-get install -y \
    build-essential \
    cmake \
    libsdl2-dev \
    libglfw3-dev \
    libgl1-mesa-dev \
    git
```

## Project Setup

The project expects the following directory structure:

```
parent_directory/
├── imgui/                    # Dear ImGui repository
│   ├── imgui.cpp
│   ├── imgui.h
│   ├── backends/
│   └── ...
└── imgui_test_engine/        # This repository
    ├── CMakeLists.txt        # Place the CMakeLists.txt here
    ├── imgui_test_engine/
    ├── imgui_test_suite/
    ├── shared/
    └── ...
```

### Step 1: Clone both repositories

```bash
# Clone imgui (main repository)
git clone https://github.com/ocornut/imgui.git
cd imgui
git checkout docking  # Recommended: use docking branch for full features
cd ..

# Clone imgui_test_engine
git clone https://github.com/ocornut/imgui_test_engine.git
cd imgui_test_engine

# Initialize submodules (for implot)
git submodule update --init --recursive
```

### Step 2: Place CMakeLists.txt

Place the provided `CMakeLists.txt` in the root of the `imgui_test_engine` directory.

## Build Configurations

### 1. Standard Build (All Features)

Build with both SDL2 and GLFW3 backends, ImPlot enabled:

```bash
cd imgui_test_engine
cmake -B build
cmake --build build -j$(nproc) --verbose
```

### 2. Build with SDL2 Only

```bash
cmake -B build -DBUILD_WITH_GLFW3=OFF
cmake --build build --verbose -j$(nproc)
```

### 3. Build with GLFW3 Only

```bash
cmake -B build -DBUILD_WITH_SDL2=OFF
cmake --build build --verbose -j$(nproc)
```

### 4. Build with IMPLOT=0 and STD_FUNCTION=0 (Minimal)

This matches the CI test case for minimal dependencies:

```bash
cmake -B build \
    -DIMGUI_TEST_ENGINE_ENABLE_IMPLOT=OFF \
    -DIMGUI_TEST_ENGINE_ENABLE_STD_FUNCTION=OFF
cmake --build build --verbose -j$(nproc)
```

### 5. Custom ImGui Path

If your imgui is not at `../imgui`, specify it:

```bash
cmake -B build -DIMGUI_DIR=/path/to/your/imgui
cmake --build build --verbose -j$(nproc)
```

## Running Tests

### Run All Tests (Regular)

```bash
cd build
ctest --output-on-failure
```

Or run tests for specific backend:

```bash
# SDL2 tests
./imgui_test_suite_sdl2 -nogui -nopause -nothrottle

# GLFW3 tests
./imgui_test_suite_glfw3 -nogui -nopause -nothrottle
```

### Run Viewport Tests

```bash
cd build

# SDL2 viewport tests
./imgui_test_suite_sdl2 -nogui -nopause -nothrottle -viewport

# GLFW3 viewport tests
./imgui_test_suite_glfw3 -nogui -nopause -nothrottle -viewport
```

Or use CTest with filter:

```bash
ctest -R ViewportTests --output-on-failure
```

## CI Workflow Tasks

To replicate all the CI workflow tasks:

### Complete CI Workflow Script

```bash
#!/bin/bash
set -e

# Step 1: Clone repositories
git clone https://github.com/ocornut/imgui.git
cd imgui
git checkout docking
cd ..

git clone https://github.com/ocornut/imgui_test_engine.git
cd imgui_test_engine
git submodule update --init --recursive

# Copy CMakeLists.txt to root (assuming you have it saved)
# cp /path/to/CMakeLists.txt .

# Step 2: Build with SDL2
echo "=== Building with SDL2 ==="
rm -rf build && mkdir build && cd build
cmake .. -DBUILD_WITH_GLFW3=OFF
cmake --build . --target imgui_test_suite_sdl2 -j$(nproc)
cd ..

# Step 3: Build with GLFW3
echo "=== Building with GLFW3 ==="
rm -rf build && mkdir build && cd build
cmake .. -DBUILD_WITH_SDL2=OFF -DBUILD_WITH_GLFW3=ON
cmake --build . --target imgui_test_suite_glfw3 -j$(nproc)
cd ..

# Step 4: Build minimal configuration (no ImPlot, no std::function)
echo "=== Building minimal configuration ==="
rm -rf build && mkdir build && cd build
cmake .. \
    -DIMGUI_TEST_ENGINE_ENABLE_IMPLOT=OFF \
    -DIMGUI_TEST_ENGINE_ENABLE_STD_FUNCTION=OFF
cmake --build . --target imgui_test_suite_minimal -j$(nproc)
cd ..

# Step 5: Build all and run tests
echo "=== Building all targets and running tests ==="
rm -rf build && mkdir build && cd build
cmake .. \
    -DBUILD_WITH_SDL2=ON \
    -DBUILD_WITH_GLFW3=ON \
    -DBUILD_TEST_SUITE=ON \
    -DENABLE_TESTS=ON \
    -DENABLE_VIEWPORT_TESTS=ON
cmake --build . --target build_all_tests -j$(nproc)

# Run regular tests
echo "=== Running regular tests ==="
ctest -R RunTests --output-on-failure

# Run viewport tests
echo "=== Running viewport tests ==="
ctest -R ViewportTests --output-on-failure

echo "All CI tasks completed successfully!"
```

## CMake Options Reference

| Option | Default | Description |
|--------|---------|-------------|
| `IMGUI_DIR` | `../imgui` | Path to Dear ImGui directory |
| `IMGUI_TEST_ENGINE_ENABLE_IMPLOT` | ON | Enable ImPlot support for performance tools |
| `IMGUI_TEST_ENGINE_ENABLE_STD_FUNCTION` | ON | Enable std::function support |
| `BUILD_WITH_SDL2` | ON | Build with SDL2 backend |
| `BUILD_WITH_GLFW3` | ON | Build with GLFW3 backend |
| `BUILD_CAPTURE_TOOL` | ON | Build imgui_capture_tool executable (note: currently skipped) |
| `BUILD_TEST_SUITE` | ON | Build imgui_test_suite executable |
| `ENABLE_TESTS` | ON | Enable CTest integration for regular tests |
| `ENABLE_VIEWPORT_TESTS` | ON | Enable CTest integration for viewport tests |

## Build Targets

- `imgui_test_suite_sdl2` - Test suite with SDL2 backend
- `imgui_test_suite_glfw3` - Test suite with GLFW3 backend
- `imgui_test_suite_minimal` - Test suite without ImPlot and std::function
- `build_all_tests` - Build all test executables
- `run_all_ci_tests` - Build and run all tests (CI workflow)

## Troubleshooting

### ImGui Not Found

```bash
# Make sure imgui is at the correct location
ls ../imgui/imgui.cpp

# Or specify custom path
cmake .. -DIMGUI_DIR=/path/to/imgui
```

### Missing Test Source Files

The CMakeLists.txt expects these files in `imgui_test_suite/`:
- imgui_test_suite.cpp
- imgui_tests_core.cpp
- imgui_tests_widgets.cpp
- imgui_tests_widgets_inputtext.cpp
- imgui_tests_nav.cpp
- imgui_tests_tables.cpp
- imgui_tests_docking.cpp
- imgui_tests_viewports.cpp
- imgui_tests_perf.cpp

And in `shared/`:
- imgui_app.cpp
- imgui_app.h

If files are missing, make sure you're using the latest version of the repository.

### SDL2 Not Found

```bash
# Ubuntu/Debian
sudo apt-get install libsdl2-dev

# Fedora/RHEL
sudo dnf install SDL2-devel

# Arch
sudo pacman -S sdl2
```

### GLFW3 Not Found

```bash
# Ubuntu/Debian
sudo apt-get install libglfw3-dev

# Fedora/RHEL
sudo dnf install glfw-devel

# Arch
sudo pacman -S glfw-x11
```

### ImPlot Not Found

ImPlot is a submodule. Initialize it:

```bash
cd imgui_test_engine
git submodule update --init --recursive
```

Or disable ImPlot:

```bash
cmake .. -DIMGUI_TEST_ENGINE_ENABLE_IMPLOT=OFF
```

### Backend Implementation Issues

If you see errors about `imgui_impl_opengl3.cpp` being compiled multiple times:

This is expected when building both SDL2 and GLFW3 backends. The CMakeLists handles this by conditionally adding the implementation based on which backends are enabled.

To avoid this, build only one backend at a time:

```bash
# SDL2 only
cmake .. -DBUILD_WITH_SDL2=ON -DBUILD_WITH_GLFW3=OFF

# GLFW3 only
cmake .. -DBUILD_WITH_SDL2=OFF -DBUILD_WITH_GLFW3=ON
```

## Running Tests Manually

### GUI Mode (Interactive)

```bash
# Run with GUI to see tests execute
./imgui_test_suite_sdl2
```

### Headless Mode (CI)

```bash
# Fast mode without GUI
./imgui_test_suite_sdl2 -nogui -nopause -nothrottle
```

### Run Specific Tests

```bash
# Run only a specific test
./imgui_test_suite_sdl2 -nogui -nopause -nothrottle -test "widgets_*"
```

### Test Options

- `-nogui` - Run without rendering GUI (faster)
- `-nopause` - Don't pause between tests
- `-nothrottle` - Run as fast as possible
- `-viewport` - Enable viewport tests
- `-test <pattern>` - Run only tests matching pattern

## Advanced Usage

### Debug Build

```bash
cmake .. -DCMAKE_BUILD_TYPE=Debug
cmake --build . -j$(nproc)
```

### Release Build with Optimizations

```bash
cmake .. -DCMAKE_BUILD_TYPE=Release
cmake --build . -j$(nproc)
```

### Verbose Build

```bash
cmake --build . --verbose
```

### Clean Build

```bash
rm -rf build
mkdir build && cd build
cmake ..
cmake --build . -j$(nproc)
```

## Notes

1. **ImGui Branch**: It's recommended to use the `docking` branch of ImGui for full feature support, especially for viewport tests.

2. **Capture Tool**: The imgui_capture_tool is currently not built as a standalone executable in this CMakeLists.txt because it requires special application setup. The capture functionality is available through the imgui_test_suite applications.

3. **Test Engine Library**: The test engine is built as a static library that can be linked into your own applications.

4. **Shared Helpers**: The `shared/imgui_app.cpp` provides common application framework code used by the test suite.

5. **Compilation Definitions**: The build automatically adds required definitions like `IMGUI_ENABLE_TEST_ENGINE` and `IMGUI_TEST_ENGINE_ENABLE_COROUTINE_STDTHREAD_IMPL=1`.
