cmake_minimum_required(VERSION 3.21)
project(coel VERSION 0.1.0)
include(CMakePackageConfigHelpers)
include(GNUInstallDirs)
add_library(coel
    "src/opengl/context.cpp"
    
    "src/vulkan/core.cpp"
    "src/vulkan/instance.cpp"
    "src/vulkan/device.cpp"
    "src/vulkan/surface.cpp"
    "src/vulkan/swapchain.cpp"
    "src/vulkan/pipeline.cpp"
    "src/vulkan/buffer.cpp"
    "src/vulkan/image.cpp"
    "src/vulkan/command.cpp"
)
add_library(coel::coel ALIAS coel)

if(CMAKE_CXX_COMPILER_ID STREQUAL "MSVC")
    target_compile_options(coel PUBLIC /std:c++latest)
else()
    target_compile_features(coel PUBLIC cxx_std_20)
endif()

find_library(SHADERC_LIB shaderc)
find_library(SHADERC_UTIL_LIB shaderc_util)
find_path(SHADERC_INC "shaderc/shaderc.h")
find_library(SPIRV_TOOLS_LIB SPIRV-Tools)
find_library(SPIRV_TOOLS_LINK_LIB SPIRV-Tools-link)
find_library(SPIRV_TOOLS_OPT_LIB SPIRV-Tools-opt)
find_library(SPIRV_TOOLS_REDUCE_LIB SPIRV-Tools-reduce)
find_library(SPIRV_TOOLS_SHARED_LIB SPIRV-Tools-shared)
find_package(glslang CONFIG REQUIRED)
find_package(volk CONFIG REQUIRED)
target_link_libraries(coel
    PUBLIC
        volk::volk
        HLSL SPIRV glslang OGLCompiler
    PRIVATE
        ${SHADERC_LIB} ${SHADERC_UTIL_LIB}
        ${SPIRV_TOOLS_LIB}
        ${SPIRV_TOOLS_LINK_LIB}
        ${SPIRV_TOOLS_OPT_LIB}
        ${SPIRV_TOOLS_REDUCE_LIB}
        ${SPIRV_TOOLS_SHARED_LIB}
)
target_include_directories(coel PRIVATE ${SHADERC_INC})

# API selection
set(COEL_NATIVE_APIS NULL X11 WIN32)
if(CMAKE_SYSTEM_NAME STREQUAL "Linux")
    set(COEL_NATIVE_API COEL_NATIVE_API_X11)
elseif(CMAKE_SYSTEM_NAME STREQUAL "Windows")
    set(COEL_NATIVE_API COEL_NATIVE_API_WIN32)
endif()
foreach(NATIVE_API_NAME ${COEL_NATIVE_APIS})
    if(COEL_NATIVE_API STREQUAL "COEL_NATIVE_API_${NATIVE_API_NAME}")
        set(COEL_USE_${NATIVE_API_NAME} 1)
        target_compile_definitions(coel PUBLIC COEL_USE_${NATIVE_API_NAME}=1)
    else()
        set(COEL_USE_${NATIVE_API_NAME} 0)
        target_compile_definitions(coel PUBLIC COEL_USE_${NATIVE_API_NAME}=0)
    endif()
endforeach()
if(COEL_USE_X11)
    target_compile_definitions(coel PUBLIC VK_USE_PLATFORM_XLIB_KHR)
    target_link_libraries(coel PUBLIC X11 GL ${CMAKE_DL_LIBS})
endif()
if(COEL_USE_WIN32)
    target_compile_definitions(coel PUBLIC VK_USE_PLATFORM_WIN32_KHR WIN32_LEAN_AND_MEAN NOMINMAX)
endif()

# Build options
if(BUILD_SHARED_LIBS)
    if(CMAKE_SYSTEM_NAME STREQUAL "Windows")
        target_link_libraries(coel PRIVATE opengl32)
        target_compile_definitions(coel
            PRIVATE "COEL_EXPORT=__declspec(dllexport)"
            INTERFACE "COEL_EXPORT=__declspec(dllimport)")
    else()
        target_compile_definitions(coel
            PUBLIC "COEL_EXPORT=")
    endif()
else()
    target_compile_definitions(coel
        PUBLIC "COEL_EXPORT=")
endif()

target_include_directories(coel PUBLIC
    $<BUILD_INTERFACE:${CMAKE_CURRENT_LIST_DIR}/include>
    $<INSTALL_INTERFACE:${CMAKE_INSTALL_INCLUDEDIR}>
)

# Packaging
file(WRITE ${CMAKE_BINARY_DIR}/config.cmake.in [=[
@PACKAGE_INIT@
include(${CMAKE_CURRENT_LIST_DIR}/coel-targets.cmake)
check_required_components(coel)
find_package(glslang CONFIG REQUIRED)
find_package(volk CONFIG REQUIRED)
]=])
configure_package_config_file(${CMAKE_BINARY_DIR}/config.cmake.in
    ${CMAKE_CURRENT_BINARY_DIR}/coel-config.cmake
    INSTALL_DESTINATION ${CMAKE_INSTALL_DATADIR}/coel
    NO_SET_AND_CHECK_MACRO)
write_basic_package_version_file(
    ${CMAKE_CURRENT_BINARY_DIR}/coel-config-version.cmake
    VERSION ${PROJECT_VERSION}
    COMPATIBILITY SameMajorVersion)
install(
    FILES
        ${CMAKE_CURRENT_BINARY_DIR}/coel-config.cmake
        ${CMAKE_CURRENT_BINARY_DIR}/coel-config-version.cmake
    DESTINATION
        ${CMAKE_INSTALL_DATADIR}/coel)
install(TARGETS coel EXPORT coel-targets)
install(EXPORT coel-targets DESTINATION ${CMAKE_INSTALL_DATADIR}/coel NAMESPACE coel::)
install(DIRECTORY ${CMAKE_CURRENT_LIST_DIR}/include/ TYPE INCLUDE)
