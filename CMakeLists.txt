cmake_minimum_required(VERSION 3.30) # FetchContent is available in 3.11+
project(SquareMarchingMiningGame)

# For sanitizers
# set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -g -fsanitize=address -fno-omit-frame-pointer")

if(CMAKE_BUILD_TYPE STREQUAL "Debug")
    # Debug-specific flags
    if(UNIX AND NOT APPLE)
        # set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -g -fsanitize=address -fno-omit-frame-pointer")
    endif ()
elseif(CMAKE_BUILD_TYPE STREQUAL "Release")
    # Release-specific flags
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -o3")
endif()

set(CMAKE_CXX_STANDARD 23)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)

set(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/bin)
set(CMAKE_LIBRARY_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/lib)
set(CMAKE_ARCHIVE_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/lib)

include(cmake/Options.cmake)
include(cmake/CompilerFlags.cmake)
include(cmake/CopyHelper.cmake)

set(CMAKE_MSVC_RUNTIME_LIBRARY "MultiThreaded")

# Generate compile_commands.json
set(CMAKE_EXPORT_COMPILE_COMMANDS ON)

# Dependencies
set(RAYLIB_VERSION 5.5)
set(BUILD_SHARED_LIBS OFF) # Force static linking
find_package(raylib ${RAYLIB_VERSION} QUIET) # QUIET or REQUIRED
if (NOT raylib_FOUND) # If there's none, fetch and build raylib
    include(FetchContent)
    FetchContent_Declare(
            raylib
            DOWNLOAD_EXTRACT_TIMESTAMP OFF
            URL https://github.com/raysan5/raylib/archive/refs/tags/${RAYLIB_VERSION}.tar.gz
    )
    FetchContent_GetProperties(raylib)
    if (NOT raylib_POPULATED) # Have we downloaded raylib yet?
        set(FETCHCONTENT_QUIET NO)
        FetchContent_MakeAvailable(raylib)
        set(BUILD_EXAMPLES OFF CACHE BOOL "" FORCE) # don't build the supplied examples
    endif()
endif()

# Our Project
add_executable(${PROJECT_NAME} main.cpp
        config.h
        Utilities/Vector2Utils.cpp
        Utilities/Vector2Utils.h
        CoreGameLogic/GameManager.cpp
        CoreGameLogic/GameManager.h
        CoreGameLogic/GameObject.cpp
        CoreGameLogic/GameObject.h
        CoreGameLogic/Component.cpp
        CoreGameLogic/Component.h
        GameLogic/Player/PlayerRenderer.cpp
        GameLogic/Player/PlayerRenderer.h
        GameLogic/Player/PlayerMovement.cpp
        GameLogic/Player/PlayerMovement.h
        GameLogic/Dynamics/V2SecondOrderDynamics.cpp
        GameLogic/Dynamics/V2SecondOrderDynamics.h
        Utilities/GameUtilities.cpp
        Utilities/GameUtilities.h
        Libraries/stb_image.h
        GameLogic/Map/MapFileReader.cpp
        GameLogic/Map/MapFileReader.h
        GameLogic/Map/Terrain.cpp
        GameLogic/Map/Terrain.h
        GameLogic/Map/TerrainData.cpp
        GameLogic/Map/TerrainData.h
        GameLogic/Map/TerrainChunk.cpp
        GameLogic/Map/TerrainChunk.h
        CoreGameLogic/RenderPipeline.cpp
        CoreGameLogic/RenderPipeline.h
        EventSystem/ActionEvent.h
        EventSystem/FunctionEvent.h
        CoreGameLogic/RenderPass.cpp
        CoreGameLogic/RenderPass.h
        GameLogic/Testing/TerrainMinerTest.cpp
        GameLogic/Testing/TerrainMinerTest.h
        CoreGameLogic/GameCamera.cpp
        CoreGameLogic/GameCamera.h
        GameLogic/Collisions/CollisionInfo.h
        GameLogic/Testing/PulsingOrb.cpp
        GameLogic/Testing/PulsingOrb.h
        GameLogic/Testing/OrbDeployer.cpp
        GameLogic/Testing/OrbDeployer.h
        GameLogic/Map/TerrainPoint.h
        GameLogic/Collisions/RaycastHitInfo.h
        GameLogic/Testing/TerrainRaycasterTest.cpp
        GameLogic/Testing/TerrainRaycasterTest.h
        GameLogic/UI/UIManager.cpp
        GameLogic/UI/UIManager.h
        GameLogic/UI/UIElement.cpp
        GameLogic/UI/UIElement.h
        CoreGameLogic/InputManager.cpp
        CoreGameLogic/InputManager.h
        CoreGameLogic/InputManager.cpp
        CoreGameLogic/InputManager.h
        GameLogic/UI/UIOreMeter.cpp
        GameLogic/UI/UIOreMeter.h
        GameLogic/UI/Inventory/UIInventory.cpp
        GameLogic/UI/Inventory/UIInventory.h
        GameLogic/Player/Inventory/InventoryManager.cpp
        GameLogic/Player/Inventory/InventoryManager.h
        GameLogic/Player/Inventory/InventorySlot.h
        GameLogic/Player/Inventory/IItem.cpp
        GameLogic/Player/Inventory/IItem.h
        GameLogic/Player/Inventory/ItemInfo.h
        GameLogic/Player/Tools/MiningTool.cpp
        GameLogic/Player/Tools/MiningTool.h
        GameLogic/Player/Inventory/PlayerInventory.cpp
        GameLogic/Player/Inventory/PlayerInventory.h
        GameLogic/UI/Inventory/UIInventorySlot.h
        CoreGameLogic/AssetManager.cpp
        CoreGameLogic/AssetManager.h
        GameLogic/Player/Tools/BallSpawnerTool.cpp
        GameLogic/Player/Tools/BallSpawnerTool.h
        GameLogic/Player/Items/SingleOreTankItem.cpp
        GameLogic/Player/Items/SingleOreTankItem.h
        GameLogic/Ores/OreInfo.cpp
        GameLogic/Ores/OreInfo.h
        GameLogic/Ores/IOreContainer.h
)
target_link_libraries(${PROJECT_NAME} raylib)

# Web Configurations
if (${PLATFORM} STREQUAL "Web")
    set_target_properties(${PROJECT_NAME} PROPERTIES SUFFIX ".html") # Tell Emscripten to build an example.html file.
    set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} -s USE_GLFW=3 -s ASSERTIONS=1 -s WASM=1 -s ASYNCIFY -s GL_ENABLE_GET_PROC_ADDRESS=1")
endif()

# Checks if OSX and links appropriate frameworks (Only required on MacOS)
if (APPLE)
    target_link_libraries(${PROJECT_NAME} "-framework IOKit")
    target_link_libraries(${PROJECT_NAME} "-framework Cocoa")
    target_link_libraries(${PROJECT_NAME} "-framework OpenGL")
endif()

set_compiler_flags(RUN_SANITIZERS TRUE TARGET_NAMES ${MAIN_EXECUTABLE_NAME})

copy_files(DIRECTORY GeneratedMaps COPY_TO_DESTINATION TARGET_NAME ${PROJECT_NAME})
copy_files(DIRECTORY Shaders COPY_TO_DESTINATION TARGET_NAME ${PROJECT_NAME})
copy_files(DIRECTORY Sounds COPY_TO_DESTINATION TARGET_NAME ${PROJECT_NAME})
copy_files(DIRECTORY Textures COPY_TO_DESTINATION TARGET_NAME ${PROJECT_NAME})