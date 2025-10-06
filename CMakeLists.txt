cmake_minimum_required(VERSION 3.15)
project(imgui_test_engine VERSION 1.0.0 LANGUAGES CXX C)

set(CMAKE_CXX_STANDARD 14)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_EXPORT_COMPILE_COMMANDS ON)

# Options
option(IMGUI_TEST_ENGINE_ENABLE_IMPLOT "Enable ImPlot support" ON)
option(IMGUI_TEST_ENGINE_ENABLE_STD_FUNCTION "Enable std::function support" ON)
option(BUILD_WITH_SDL2 "Build with SDL2 backend" ON)
option(BUILD_WITH_GLFW3 "Build with GLFW3 backend" ON)
option(BUILD_CAPTURE_TOOL "Build imgui_capture_tool" ON)
option(BUILD_TEST_SUITE "Build imgui_test_suite" ON)
option(ENABLE_TESTS "Enable test execution" ON)
option(ENABLE_VIEWPORT_TESTS "Enable viewport tests" ON)

# Set paths - imgui is expected to be at ../imgui relative to this project
set(IMGUI_DIR "${CMAKE_CURRENT_SOURCE_DIR}/../imgui" CACHE PATH "Path to imgui directory")
set(IMGUI_TEST_ENGINE_DIR "${CMAKE_CURRENT_SOURCE_DIR}/imgui_test_engine")
set(IMPLOT_DIR "${CMAKE_CURRENT_SOURCE_DIR}/imgui_test_suite/thirdparty/implot")
set(SHARED_DIR "${CMAKE_CURRENT_SOURCE_DIR}/shared")

# Set IMGUI_USER_CONFIG (matches Makefile behavior)
set(IMGUI_USER_CONFIG "imgui_test_suite/imgui_test_suite_imconfig.h" CACHE STRING "Path to custom imgui config file")

# Compiler flags
if(CMAKE_CXX_COMPILER_ID MATCHES "GNU|Clang")
    add_compile_options(-Wall -Wextra -Werror)
endif()

# Find required packages
find_package(OpenGL REQUIRED)
find_package(Threads REQUIRED)

if(BUILD_WITH_SDL2)
    find_package(SDL2 REQUIRED)
endif()

if(BUILD_WITH_GLFW3)
    find_package(glfw3 REQUIRED)
    if(UNIX AND NOT APPLE)
        find_package(X11 REQUIRED)
    endif()
endif()

# Check if imgui directory exists
if(NOT EXISTS "${IMGUI_DIR}/imgui.cpp")
    message(FATAL_ERROR "ImGui not found at ${IMGUI_DIR}. Please set IMGUI_DIR to the correct path or place imgui at ../imgui")
endif()

# ImGui library
add_library(imgui STATIC
    ${IMGUI_DIR}/imgui.cpp
    ${IMGUI_DIR}/imgui_demo.cpp
    ${IMGUI_DIR}/imgui_draw.cpp
    ${IMGUI_DIR}/imgui_tables.cpp
    ${IMGUI_DIR}/imgui_widgets.cpp
)

target_include_directories(imgui PUBLIC
    ${CMAKE_CURRENT_SOURCE_DIR}
    ${IMGUI_DIR}
    ${IMGUI_DIR}/backends
)

target_link_libraries(imgui PUBLIC
    OpenGL::GL
    ${CMAKE_DL_LIBS}
)

# Add IMGUI_USER_CONFIG to imgui
if(IMGUI_USER_CONFIG)
    target_compile_definitions(imgui PUBLIC IMGUI_USER_CONFIG="${IMGUI_USER_CONFIG}")
endif()

# Add backend implementations based on what's being built
if(BUILD_WITH_SDL2)
    target_sources(imgui PRIVATE
        ${IMGUI_DIR}/backends/imgui_impl_sdl2.cpp
        ${IMGUI_DIR}/backends/imgui_impl_opengl3.cpp
    )
    target_link_libraries(imgui PUBLIC SDL2::SDL2)
endif()

if(BUILD_WITH_GLFW3)
    target_sources(imgui PRIVATE
        ${IMGUI_DIR}/backends/imgui_impl_glfw.cpp
        ${IMGUI_DIR}/backends/imgui_impl_opengl3.cpp
    )
    target_link_libraries(imgui PUBLIC glfw)
endif()

# ImPlot library (optional)
if(IMGUI_TEST_ENGINE_ENABLE_IMPLOT)
    if(EXISTS ${IMPLOT_DIR}/implot.cpp)
        add_library(implot STATIC
            ${IMPLOT_DIR}/implot.cpp
            ${IMPLOT_DIR}/implot_demo.cpp
            ${IMPLOT_DIR}/implot_items.cpp
        )

        target_include_directories(implot PUBLIC
            ${IMPLOT_DIR}
            ${IMGUI_DIR}
        )

        target_link_libraries(implot PUBLIC imgui)
    else()
        message(WARNING "ImPlot not found at ${IMPLOT_DIR}, disabling ImPlot support")
        set(IMGUI_TEST_ENGINE_ENABLE_IMPLOT OFF)
    endif()
endif()

# ImGui Test Engine library
add_library(imgui_test_engine STATIC
    ${IMGUI_TEST_ENGINE_DIR}/imgui_capture_tool.cpp
    ${IMGUI_TEST_ENGINE_DIR}/imgui_te_context.cpp
    ${IMGUI_TEST_ENGINE_DIR}/imgui_te_coroutine.cpp
    ${IMGUI_TEST_ENGINE_DIR}/imgui_te_engine.cpp
    ${IMGUI_TEST_ENGINE_DIR}/imgui_te_exporters.cpp
    ${IMGUI_TEST_ENGINE_DIR}/imgui_te_perftool.cpp
    ${IMGUI_TEST_ENGINE_DIR}/imgui_te_ui.cpp
    ${IMGUI_TEST_ENGINE_DIR}/imgui_te_utils.cpp
)

target_include_directories(imgui_test_engine PUBLIC
    ${IMGUI_TEST_ENGINE_DIR}
    ${IMGUI_TEST_ENGINE_DIR}/thirdparty
    ${IMGUI_DIR}
)

target_compile_definitions(imgui_test_engine PUBLIC
    IMGUI_TEST_ENGINE_ENABLE_COROUTINE_STDTHREAD_IMPL=1
)

# Add IMGUI_USER_CONFIG to imgui_test_engine
if(IMGUI_USER_CONFIG)
    target_compile_definitions(imgui_test_engine PUBLIC IMGUI_USER_CONFIG="${IMGUI_USER_CONFIG}")
endif()

target_link_libraries(imgui_test_engine PUBLIC
    imgui
    Threads::Threads
)

# Configure compile definitions
if(IMGUI_TEST_ENGINE_ENABLE_IMPLOT)
    target_compile_definitions(imgui_test_engine PUBLIC IMGUI_TEST_ENGINE_ENABLE_IMPLOT=1)
    if(TARGET implot)
        target_link_libraries(imgui_test_engine PUBLIC implot)
    endif()
else()
    target_compile_definitions(imgui_test_engine PUBLIC IMGUI_TEST_ENGINE_ENABLE_IMPLOT=0)
endif()

if(NOT IMGUI_TEST_ENGINE_ENABLE_STD_FUNCTION)
    target_compile_definitions(imgui_test_engine PUBLIC IMGUI_TEST_ENGINE_DISABLE_STD_FUNCTION)
endif()

# Shared library for test applications
add_library(shared_helpers INTERFACE)
target_include_directories(shared_helpers INTERFACE ${SHARED_DIR})

# Function to create test suite executable
function(add_test_suite_executable TARGET_NAME BACKEND_LIB EXTRA_DEFS)
    set(TEST_SUITE_DIR "${CMAKE_CURRENT_SOURCE_DIR}/imgui_test_suite")

    # Find all test source files
    set(TEST_SOURCES
        ${TEST_SUITE_DIR}/imgui_test_suite.cpp
        ${TEST_SUITE_DIR}/imgui_tests_core.cpp
        ${TEST_SUITE_DIR}/imgui_tests_widgets.cpp
        ${TEST_SUITE_DIR}/imgui_tests_widgets_inputtext.cpp
        ${TEST_SUITE_DIR}/imgui_tests_inputs.cpp
        ${TEST_SUITE_DIR}/imgui_tests_nav.cpp
        ${TEST_SUITE_DIR}/imgui_tests_tables.cpp
        ${TEST_SUITE_DIR}/imgui_tests_docking.cpp
        ${TEST_SUITE_DIR}/imgui_tests_viewports.cpp
        ${TEST_SUITE_DIR}/imgui_tests_perf.cpp
        ${SHARED_DIR}/imgui_app.cpp
    )

    add_executable(${TARGET_NAME} ${TEST_SOURCES})

    target_include_directories(${TARGET_NAME} PRIVATE
        ${CMAKE_SOURCE_DIR}
        ${TEST_SUITE_DIR}
        ${SHARED_DIR}
        ${IMGUI_DIR}
        ${IMGUI_DIR}/backends
        ${IMGUI_TEST_ENGINE_DIR}
    )

    target_link_libraries(${TARGET_NAME} PRIVATE
        imgui
        imgui_test_engine
        shared_helpers
        Threads::Threads
        OpenGL::GL
    )

    # Backend-specific configuration
    if(BACKEND_LIB STREQUAL "sdl2")
        target_compile_definitions(${TARGET_NAME} PRIVATE IMGUI_APP_SDL_GL3)
        target_link_libraries(${TARGET_NAME} PRIVATE SDL2::SDL2)
    elseif(BACKEND_LIB STREQUAL "glfw3")
        target_compile_definitions(${TARGET_NAME} PRIVATE IMGUI_APP_GLFW_GL3)
        target_link_libraries(${TARGET_NAME} PRIVATE glfw)
        if(UNIX AND NOT APPLE)
            target_link_libraries(${TARGET_NAME} PRIVATE X11::X11)
        endif()
    endif()

    # Add IMGUI_USER_CONFIG to test suite
    if(IMGUI_USER_CONFIG)
        target_compile_definitions(${TARGET_NAME} PRIVATE IMGUI_USER_CONFIG="${IMGUI_USER_CONFIG}")
    endif()

    if(IMGUI_TEST_ENGINE_ENABLE_IMPLOT AND TARGET implot)
        target_link_libraries(${TARGET_NAME} PRIVATE implot)
    endif()

    # Add extra definitions
    if(EXTRA_DEFS)
        target_compile_definitions(${TARGET_NAME} PRIVATE ${EXTRA_DEFS})
    endif()
endfunction()

# Build imgui_test_suite with different configurations
if(BUILD_TEST_SUITE)
    if(BUILD_WITH_SDL2)
        add_test_suite_executable(imgui_test_suite_sdl2 "sdl2" "")

        # Add test for SDL2 build
        if(ENABLE_TESTS)
            add_test(NAME RunTests_SDL2
                COMMAND imgui_test_suite_sdl2 -nogui -nopause -nothrottle
                WORKING_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}
            )
        endif()

        # Add viewport test for SDL2 build
        if(ENABLE_VIEWPORT_TESTS)
            add_test(NAME RunViewportTests_SDL2
                COMMAND imgui_test_suite_sdl2 -nogui -nopause -nothrottle -viewport
                WORKING_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}
            )
        endif()
    endif()

    if(BUILD_WITH_GLFW3)
        add_test_suite_executable(imgui_test_suite_glfw3 "glfw3" "")

        # Add test for GLFW3 build
        if(ENABLE_TESTS)
            add_test(NAME RunTests_GLFW3
                COMMAND imgui_test_suite_glfw3 -nogui -nopause -nothrottle
                WORKING_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}
            )
        endif()

        # Add viewport test for GLFW3 build
        if(ENABLE_VIEWPORT_TESTS)
            add_test(NAME RunViewportTests_GLFW3
                COMMAND imgui_test_suite_glfw3 -nogui -nopause -nothrottle -viewport
                WORKING_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}
            )
        endif()
    endif()

    # Build with IMPLOT=0 and STD_FUNCTION=0 (using SDL2 as default backend)
    if(BUILD_WITH_SDL2)
        set(MINIMAL_DEFS
            "IMGUI_TEST_ENGINE_ENABLE_IMPLOT=0"
            "IMGUI_TEST_ENGINE_DISABLE_STD_FUNCTION"
        )
        add_test_suite_executable(imgui_test_suite_minimal "sdl2" "${MINIMAL_DEFS}")
    endif()
endif()

# Build imgui_capture_tool (matches CI workflow single-file build approach)
if(BUILD_CAPTURE_TOOL)
    # Check if example_null exists
    if(NOT EXISTS "${IMGUI_DIR}/examples/example_null/main.cpp")
        message(WARNING "imgui example_null not found at ${IMGUI_DIR}/examples/example_null/main.cpp")
        message(STATUS "Note: imgui_capture_tool build skipped - requires imgui/examples/example_null/main.cpp")
    else()
        # Build as single-file executable (matches CI workflow approach)
        set(CAPTURE_TOOL_SOURCES
            ${IMGUI_DIR}/imgui.cpp
            ${IMGUI_DIR}/imgui_demo.cpp
            ${IMGUI_DIR}/imgui_draw.cpp
            ${IMGUI_DIR}/imgui_tables.cpp
            ${IMGUI_DIR}/imgui_widgets.cpp
            ${IMGUI_DIR}/examples/example_null/main.cpp
            ${IMGUI_TEST_ENGINE_DIR}/imgui_capture_tool.cpp
            ${IMGUI_TEST_ENGINE_DIR}/imgui_te_utils.cpp
        )

        add_executable(imgui_capture_tool ${CAPTURE_TOOL_SOURCES})

        target_include_directories(imgui_capture_tool PRIVATE
            ${CMAKE_CURRENT_SOURCE_DIR}
            ${IMGUI_DIR}
            ${IMGUI_DIR}/backends
            ${IMGUI_DIR}/examples
            ${IMGUI_TEST_ENGINE_DIR}
        )

        target_compile_definitions(imgui_capture_tool PRIVATE
            IMGUI_TEST_ENGINE_ENABLE_COROUTINE_STDTHREAD_IMPL=1
        )

        target_link_libraries(imgui_capture_tool PRIVATE
            Threads::Threads
            ${CMAKE_DL_LIBS}
        )

        # Link math library on Unix
        if(UNIX)
            target_link_libraries(imgui_capture_tool PRIVATE m)
        endif()

        message(STATUS "imgui_capture_tool will be built")
    endif()
endif()

# Enable testing
if(ENABLE_TESTS OR ENABLE_VIEWPORT_TESTS)
    enable_testing()
endif()

# Print configuration summary
message(STATUS "")
message(STATUS "ImGui Test Engine Configuration:")
message(STATUS "  IMGUI_DIR: ${IMGUI_DIR}")
message(STATUS "  IMGUI_USER_CONFIG: ${IMGUI_USER_CONFIG}")
message(STATUS "  IMGUI_TEST_ENGINE_ENABLE_IMPLOT: ${IMGUI_TEST_ENGINE_ENABLE_IMPLOT}")
message(STATUS "  IMGUI_TEST_ENGINE_ENABLE_STD_FUNCTION: ${IMGUI_TEST_ENGINE_ENABLE_STD_FUNCTION}")
message(STATUS "  BUILD_WITH_SDL2: ${BUILD_WITH_SDL2}")
message(STATUS "  BUILD_WITH_GLFW3: ${BUILD_WITH_GLFW3}")
message(STATUS "  BUILD_CAPTURE_TOOL: ${BUILD_CAPTURE_TOOL}")
message(STATUS "  BUILD_TEST_SUITE: ${BUILD_TEST_SUITE}")
message(STATUS "  ENABLE_TESTS: ${ENABLE_TESTS}")
message(STATUS "  ENABLE_VIEWPORT_TESTS: ${ENABLE_VIEWPORT_TESTS}")
message(STATUS "")

# Custom targets for convenience
if(BUILD_TEST_SUITE)
    add_custom_target(build_all_tests
        DEPENDS
            $<$<BOOL:${BUILD_WITH_SDL2}>:imgui_test_suite_sdl2>
            $<$<BOOL:${BUILD_WITH_GLFW3}>:imgui_test_suite_glfw3>
            $<$<BOOL:${BUILD_WITH_SDL2}>:imgui_test_suite_minimal>
    )

    # Custom target to run all tests (mimicking CI workflow)
    add_custom_target(run_all_ci_tests
        COMMAND ${CMAKE_COMMAND} --build . --target build_all_tests
        COMMAND ${CMAKE_CTEST_COMMAND} --output-on-failure
        WORKING_DIRECTORY ${CMAKE_BINARY_DIR}
        COMMENT "Running all CI tests"
    )
endif()
