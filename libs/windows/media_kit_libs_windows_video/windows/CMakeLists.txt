# This file is a part of media_kit (https://github.com/media-kit/media-kit).
#
# Copyright © 2021 & onwards, Hitesh Kumar Saini <saini123hitesh@gmail.com>.
# All rights reserved.
# Use of this source code is governed by MIT license that can be found in the LICENSE file.

cmake_minimum_required(VERSION 3.14)

# This option is read by the other packages which are part of package:media_kit.
option(MEDIA_KIT_LIBS_AVAILABLE "package:media_kit libraries are available." ON)

set(PROJECT_NAME "media_kit_libs_windows_video")
project(${PROJECT_NAME} LANGUAGES CXX)

# Deal with MSVC incompatiblity
add_compile_definitions(_DISABLE_CONSTEXPR_MUTEX_CONSTRUCTOR)

# ------------------------------------------------------------------------------
function(download_and_verify url md5 locationForArchive)
  # Check if the archive exists.
  if(EXISTS "${locationForArchive}")
    file(MD5 "${locationForArchive}" ARCHIVE_MD5)

    # If MD5 doesn't match, delete the archive to download again.
    if(NOT md5 STREQUAL ARCHIVE_MD5)
      file(REMOVE "${locationForArchive}")
      message(STATUS "MD5 mismatch. File deleted.")
    endif()
  endif()

  # Download the archive if it doesn't exist.
  if(NOT EXISTS "${locationForArchive}")
    message(STATUS "Downloading archive from ${url}...")
    file(DOWNLOAD "${url}" "${locationForArchive}")
    message(STATUS "Downloaded archive to ${locationForArchive}.")

    # Verify MD5 of the newly downloaded file.
    file(MD5 "${locationForArchive}" ARCHIVE_MD5)

    if(md5 STREQUAL ARCHIVE_MD5)
      message(STATUS "${locationForArchive} Verification successful.")
    else()
      message(FATAL_ERROR "${locationForArchive} Integrity check failed, please try to re-build project again.")
    endif()
  endif()
endfunction()

function(check_directory_exists_and_not_empty dir result_var)
  # Check if the directory exists
  if(EXISTS "${dir}")
    # Check if the directory is not empty
    file(GLOB dir_contents "${dir}/*")

    if(dir_contents)
      set(${result_var} TRUE PARENT_SCOPE)
    else()
      set(${result_var} FALSE PARENT_SCOPE)
      message(STATUS "Directory ${dir} exists but is empty!")
    endif()
  else()
    set(${result_var} FALSE PARENT_SCOPE)
    message(STATUS "Directory ${dir} does not exist!")
  endif()
endfunction()

# libmpv archive containing the pre-built shared libraries & headers.
set(LIBMPV "mpv-dev-aarch64-20260607-git-43b14a4.7z")

# Download URL & MD5 hash of the libmpv archive.
set(LIBMPV_URL "https://github.com/bggRGjQaUbCoE/mpv-winbuild-cmake/releases/download/20260607/${LIBMPV}")
set(LIBMPV_MD5 "e4c49712b9cf0d77e1ae8e1a9c8e3ed7")

# Download location of the libmpv archive.
set(LIBMPV_ARCHIVE "${CMAKE_BINARY_DIR}/${LIBMPV}")
set(LIBMPV_SRC "${CMAKE_BINARY_DIR}/libmpv")

download_and_verify(
  ${LIBMPV_URL}
  ${LIBMPV_MD5}
  ${LIBMPV_ARCHIVE}
)

check_directory_exists_and_not_empty(${LIBMPV_SRC} LIBMPV_SRC_VALID)

# Extract the libmpv archive.
if(NOT LIBMPV_SRC_VALID)
  message(STATUS "Extracting ${LIBMPV}...")
  make_directory("${LIBMPV_SRC}")
  add_custom_target("${PROJECT_NAME}_LIBMPV_EXTRACT" ALL)
  
  find_program(SEVEN_ZIP_EXE 7z)
  if(SEVEN_ZIP_EXE)
    message(STATUS "Found 7z: ${SEVEN_ZIP_EXE}")
    set(EXTRACT_COMMAND "${SEVEN_ZIP_EXE}" x "\"${LIBMPV_ARCHIVE}\"" -y)
  else()
    message(STATUS "7z not found, trying to use cmake -E tar (might fail for .7z with specific codecs)")
    set(EXTRACT_COMMAND "${CMAKE_COMMAND}" -E tar xf "\"${LIBMPV_ARCHIVE}\"")
  endif()

  add_custom_command(
    TARGET "${PROJECT_NAME}_LIBMPV_EXTRACT"
    POST_BUILD
    COMMAND ${EXTRACT_COMMAND}
    COMMAND xcopy "\"${LIBMPV_SRC}/include/mpv\"" "\"${LIBMPV_SRC}/mpv\"" /E /H /C /I
    COMMAND rmdir "\"${LIBMPV_SRC}/include\"" /S /Q
    COMMAND ren "\"${LIBMPV_SRC}/mpv\"" "\"include\""
    WORKING_DIRECTORY "${LIBMPV_SRC}"
  )
endif()

# ------------------------------------------------------------------------------

# ANGLE archive containing the pre-built shared libraries & headers.
set(ANGLE "angle-arm64-2026-05-30.zip")

# Download URL & MD5 hash of the ANGLE archive.
set(ANGLE_URL "https://github.com/mmozeiko/build-angle/releases/download/2026-05-30/${ANGLE}")
set(ANGLE_MD5 "782efe73281c2f1de9c5c7f35c99ecfc")

# Download location of the ANGLE archive.
set(ANGLE_ARCHIVE "${CMAKE_BINARY_DIR}/${ANGLE}")
set(ANGLE_SRC "${CMAKE_BINARY_DIR}/ANGLE")

download_and_verify(
  ${ANGLE_URL}
  ${ANGLE_MD5}
  ${ANGLE_ARCHIVE}
)

check_directory_exists_and_not_empty(${ANGLE_SRC} ANGLE_SRC_VALID)

# Extract the ANGLE archive.
if(NOT ANGLE_SRC_VALID)
  message(STATUS "Extracting ${ANGLE}...")
  make_directory("${ANGLE_SRC}")
  add_custom_target("${PROJECT_NAME}_ANGLE_EXTRACT" ALL)
  add_custom_command(
    TARGET "${PROJECT_NAME}_ANGLE_EXTRACT"
    POST_BUILD
    COMMAND "${CMAKE_COMMAND}" -E tar xf "\"${ANGLE_ARCHIVE}\""
    COMMAND powershell -NoProfile -ExecutionPolicy Bypass -Command "if (Test-Path '${ANGLE_SRC}/angle-arm64') { Copy-Item '${ANGLE_SRC}/angle-arm64/*' '${ANGLE_SRC}' -Recurse -Force }"
    COMMAND powershell -NoProfile -ExecutionPolicy Bypass -Command "if (Test-Path '${ANGLE_SRC}/lib/libEGL.lib') { Copy-Item '${ANGLE_SRC}/lib/libEGL.lib' '${ANGLE_SRC}/lib/libEGL.dll.lib' -Force }"
    COMMAND powershell -NoProfile -ExecutionPolicy Bypass -Command "if (Test-Path '${ANGLE_SRC}/lib/libGLESv2.lib') { Copy-Item '${ANGLE_SRC}/lib/libGLESv2.lib' '${ANGLE_SRC}/lib/libGLESv2.dll.lib' -Force }"
    WORKING_DIRECTORY "${ANGLE_SRC}"
  )
endif()

# ------------------------------------------------------------------------------
set(PLUGIN_NAME "media_kit_libs_windows_video_plugin")

add_library(
  ${PLUGIN_NAME} SHARED
  "include/media_kit_libs_windows_video/media_kit_libs_windows_video_plugin_c_api.h"
  "media_kit_libs_windows_video_plugin_c_api.cpp"
)

apply_standard_settings(${PLUGIN_NAME})

set_target_properties(
  ${PLUGIN_NAME}
  PROPERTIES
  CXX_VISIBILITY_PRESET
  hidden
)
target_compile_definitions(
  ${PLUGIN_NAME}
  PRIVATE
  FLUTTER_PLUGIN_IMPL
)

target_include_directories(
  ${PLUGIN_NAME} INTERFACE
  "${CMAKE_CURRENT_SOURCE_DIR}/include"
)
target_link_libraries(
  ${PLUGIN_NAME}
  PRIVATE
  flutter
  flutter_wrapper_plugin
)

# ------------------------------------------------------------------------------
# Local libraries
# ------------------------------------------------------------------------------

set(VK_SWIFTSHADER_DLL "")
# The install-vulkan-sdk-action downloads x64 SwiftShader even on ARM64 runners.
# We must skip it for ARM64 builds to avoid architecture mismatch.
if(CMAKE_GENERATOR_PLATFORM MATCHES "ARM64")
  message(STATUS "Skipping vk_swiftshader.dll for ARM64 build (binary is x64).")
else()
  if(EXISTS "C:/swiftshader/vk_swiftshader.dll")
    set(VK_SWIFTSHADER_DLL "C:/swiftshader/vk_swiftshader.dll")
  endif()
endif()

set(VULKAN_1_DLL "")
# Search recursively because the ARM64 runtime zip extracts to a subdirectory.
file(GLOB_RECURSE VULKAN_1_DLL_CANDIDATES
  LIST_DIRECTORIES false
  "C:/VulkanSDK/*.dll"
)
list(FILTER VULKAN_1_DLL_CANDIDATES INCLUDE REGEX "[/\\\\]vulkan-1\\.dll$")

if(VULKAN_1_DLL_CANDIDATES)
  list(SORT VULKAN_1_DLL_CANDIDATES)
  list(REVERSE VULKAN_1_DLL_CANDIDATES)
  if(CMAKE_GENERATOR_PLATFORM MATCHES "ARM64")
    foreach(CANDIDATE IN LISTS VULKAN_1_DLL_CANDIDATES)
      string(TOLOWER "${CANDIDATE}" CANDIDATE_LOWER)
      if(CANDIDATE_LOWER MATCHES "arm64|aarch64")
        set(VULKAN_1_DLL "${CANDIDATE}")
        break()
      endif()
    endforeach()
  else()
    list(GET VULKAN_1_DLL_CANDIDATES 0 VULKAN_1_DLL)
  endif()
endif()

if(VULKAN_1_DLL)
  message(STATUS "Found vulkan-1.dll at ${VULKAN_1_DLL}")
else()
  message(STATUS "vulkan-1.dll not found in C:/VulkanSDK/; skipping bundled Vulkan loader.")
endif()

# ------------------------------------------------------------------------------
# ZLIB (Chromium fork)
# ------------------------------------------------------------------------------

set(ZLIB_URL "https://chromium.googlesource.com/chromium/src/third_party/zlib")
set(ZLIB_SRC "${CMAKE_BINARY_DIR}/zlib_src")
set(ZLIB_BUILD "${CMAKE_BINARY_DIR}/zlib_build")

if(NOT EXISTS "${ZLIB_SRC}")
  message(STATUS "Cloning zlib from ${ZLIB_URL}...")
  execute_process(
    COMMAND git clone --depth 1 -b main "${ZLIB_URL}" "${ZLIB_SRC}"
    RESULT_VARIABLE GIT_EXIT_CODE
  )
  if(NOT GIT_EXIT_CODE EQUAL 0)
    message(FATAL_ERROR "Failed to clone zlib repository.")
  endif()
endif()

# is_official_build=true is a GN argument. In CMake, we use CMAKE_BUILD_TYPE=Release for optimizations.
if(NOT EXISTS "${ZLIB_BUILD}/zlib.dll" AND NOT EXISTS "${ZLIB_BUILD}/Release/zlib.dll" AND NOT EXISTS "${ZLIB_BUILD}/zlib1.dll" AND NOT EXISTS "${ZLIB_BUILD}/Release/zlib1.dll" AND NOT EXISTS "${ZLIB_BUILD}/z.dll" AND NOT EXISTS "${ZLIB_BUILD}/Release/z.dll")
  message(STATUS "Building zlib...")
  file(MAKE_DIRECTORY "${ZLIB_BUILD}")
  
  execute_process(
    COMMAND "${CMAKE_COMMAND}" -S "${ZLIB_SRC}" -B "${ZLIB_BUILD}"
    WORKING_DIRECTORY "${ZLIB_BUILD}"
    RESULT_VARIABLE CMAKE_CONFIG_EXIT_CODE
  )
  
  if(NOT CMAKE_CONFIG_EXIT_CODE EQUAL 0)
     message(FATAL_ERROR "Failed to configure zlib. Ensure the repository contains a CMakeLists.txt.")
  endif()

  execute_process(
    COMMAND "${CMAKE_COMMAND}" --build "${ZLIB_BUILD}" --config Release
    WORKING_DIRECTORY "${ZLIB_BUILD}"
    RESULT_VARIABLE CMAKE_BUILD_EXIT_CODE
  )
  
  if(NOT CMAKE_BUILD_EXIT_CODE EQUAL 0)
     message(FATAL_ERROR "Failed to build zlib.")
  endif()
endif()

set(ZLIB_DLL "")
foreach(CANDIDATE "${ZLIB_BUILD}/zlib.dll" "${ZLIB_BUILD}/Release/zlib.dll" "${ZLIB_BUILD}/zlib1.dll" "${ZLIB_BUILD}/Release/zlib1.dll" "${ZLIB_BUILD}/z.dll" "${ZLIB_BUILD}/Release/z.dll")
  if(EXISTS "${CANDIDATE}")
    set(ZLIB_DLL "${CANDIDATE}")
    break()
  endif()
endforeach()

if(NOT ZLIB_DLL)
    message(WARNING "zlib.dll, zlib1.dll or z.dll not found after build in ${ZLIB_BUILD}")
else()
    message(STATUS "Found zlib dll at ${ZLIB_DLL}")
endif()

set(
  media_kit_libs_windows_video_bundled_libraries
  "${LIBMPV_SRC}/libmpv-2.dll"
  "${ANGLE_SRC}/bin/d3dcompiler_47.dll"
  "${ANGLE_SRC}/bin/libEGL.dll"
  "${ANGLE_SRC}/bin/libGLESv2.dll"
  "${VK_SWIFTSHADER_DLL}"
  "${VULKAN_1_DLL}"
  "${ZLIB_DLL}"
  PARENT_SCOPE
)
