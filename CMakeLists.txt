cmake_minimum_required(VERSION 3.2)

# Build options
option(Android "Build for Android" OFF)
option(Emscripten "Build as WASM" OFF)
option(Extern_Config "Do not embed config file" ON)
set(EMSCRIPTEN_PATH "/opt/homebrew/opt/emscripten")  # Updated Emscripten path

project(OpenClaw)

if(NOT CMAKE_BUILD_TYPE)
  set(CMAKE_BUILD_TYPE Release)
endif()

set(CMAKE_CXX_STANDARD 11)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -O2 -g")

# Add subdirectories for dependencies
add_subdirectory(Box2D)
add_subdirectory(libwap)
add_subdirectory(./ThirdParty/Tinyxml)
if (WIN32)
    add_subdirectory(MidiProc)
endif()

# Include directories
include_directories(./Box2D/)
include_directories(./libwap/)
include_directories(./ThirdParty/)
include_directories(./ThirdParty/Tinyxml/)
include_directories(/opt/homebrew/include/SDL2) # Manually add SDL2_gfx include path

if (Emscripten)
    include_directories("${EMSCRIPTEN_PATH}/upstream/emscripten/system/include/emscripten/")
endif()

# Link directories
link_directories(./libwap)
link_directories(./Box2D)
link_directories(/opt/homebrew/lib) # Manually add SDL2_gfx link path

if (WIN32)
    link_directories(./ThirdParty/SDL2/lib/x86)
    link_directories(./Build_Release)
endif()

# Output directories
set(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/../Build_Release)
set(CMAKE_LIBRARY_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/../android/libs/armeabi-v7a)

# Executable target
if(Android)
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -DANDROID")
    add_library(openclaw SHARED "")
else()
    add_executable(openclaw "")
endif()

add_subdirectory(OpenClaw)

# Linker settings
list(APPEND TARGET_LIBS
    libwap
    tinyxml
    Box2D
)

# Platform-specific settings
if (Emscripten)
    if (Extern_Config)
        set(CONFIG_PRELOAD_FILE "")
    else()
        set(CONFIG_PRELOAD_FILE "--preload-file ../Build_Release/config.xml@config.xml")
    endif()
    set_target_properties(openclaw PROPERTIES LINK_FLAGS "-s FETCH -s WASM=1 -s BINARYEN_METHOD='native-wasm' -s EXPORTED_FUNCTIONS='[_main]' -s USE_SDL=2 -s USE_SDL_IMAGE=2 -s SDL2_IMAGE_FORMATS='[\"pcx\",\"png\",\"tga\"]' -s USE_SDL_TTF=2 -s USE_SDL_GFX=2 -s ASYNCIFY=1 -s TOTAL_MEMORY=268435456 ${CONFIG_PRELOAD_FILE} --preload-file ../Build_Release/CLAW.REZ@CLAW.REZ --preload-file ../Build_Release/ASSETS.ZIP@ASSETS.ZIP --preload-file ../Build_Release/console02.tga@console02.tga --preload-file ../Build_Release/clacon.ttf@clacon.ttf")
elseif (APPLE)
    find_package(SDL2 REQUIRED)
    find_package(SDL2_mixer REQUIRED)
    find_package(SDL2_ttf REQUIRED)
    find_package(SDL2_image REQUIRED)

    include_directories(${SDL2_INCLUDE_DIRS})
    list(APPEND TARGET_LIBS
        ${SDL2_LIBRARIES}
        ${SDL2_MIXER_LIBRARIES}
        ${SDL2_TTF_LIBRARIES}
        ${SDL2_IMAGE_LIBRARIES}
        SDL2_gfx # Manually add SDL2_gfx to the linked libraries
        "-framework OpenGL"
        "-framework Cocoa"
    )
else()
    find_package(SDL2 REQUIRED)
    find_package(SDL2_mixer REQUIRED)
    find_package(SDL2_ttf REQUIRED)
    find_package(SDL2_image REQUIRED)

    include_directories(${SDL2_INCLUDE_DIRS})
    list(APPEND TARGET_LIBS
        ${SDL2_LIBRARIES}
        ${SDL2_MIXER_LIBRARIES}
        ${SDL2_TTF_LIBRARIES}
        ${SDL2_IMAGE_LIBRARIES}
        SDL2_gfx # Manually add SDL2_gfx to the linked libraries
    )
    if (WIN32)
        list(APPEND TARGET_LIBS
            Rpcrt4
        )
    else()
        list(APPEND TARGET_LIBS
            stdc++
            m
        )
    endif()
    if (Android)
        list(APPEND TARGET_LIBS
            GLESv2
            GLESv1_CM
            dl
            log
            android
        )
    endif()
endif()

target_link_libraries(openclaw ${TARGET_LIBS})
