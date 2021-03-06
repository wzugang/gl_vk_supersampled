cmake_minimum_required(VERSION 2.8)
set(PROJNAME gl_vk_supersampled)
Project(${PROJNAME})
Message(STATUS "-------------------------------")
Message(STATUS "Processing Project ${PROJNAME}:")

SET(MODELS_DOWNLOAD_FORCE OFF CACHE BOOL "Forces the download operation of 3D models needed for this demo" )
SET(MODELS_DOWNLOAD_DISABLED ON CACHE BOOL "UNCHECK IT to download heavy 3D models needed for this demo" )
#####################################################################################
# look for shared_sources 1) as a sub-folder 2) at some other locations
# this cannot be put anywhere else since we still didn't find CMakeLists_include.txt yet
#
if(NOT BASE_DIRECTORY) # if not defined, it means this cmake file was called as the first entry point and not included
  # check if the external repository is outside or inside the project (as a sub-module)
  # testing the file CMakeLists_include.txt because when sub-modules are not cloned, the folders are still there...
  # we also assume here that if shared_sources is there, shared_external is, too...
  SET(ADD_SUBDIR_BELOW 1)
  if(EXISTS ${CMAKE_CURRENT_SOURCE_DIR}/shared_sources/CMakeLists_include.txt)
    Message(STATUS "found sub-folder shared_sources")
    SET(BASE_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR})
    INCLUDE(${CMAKE_CURRENT_SOURCE_DIR}/shared_sources/CMakeLists_include.txt)
  elseif(EXISTS ${CMAKE_CURRENT_SOURCE_DIR}/../../shared_sources)
    SET(BASE_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}/../..)
    INCLUDE(${CMAKE_CURRENT_SOURCE_DIR}/../../shared_sources/CMakeLists_include.txt)
  else()
    SET(BASE_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}/..)
    INCLUDE(${CMAKE_CURRENT_SOURCE_DIR}/../shared_sources/CMakeLists_include.txt)
  endif()
endif(NOT BASE_DIRECTORY)
_add_project_definitions(${PROJNAME})

#####################################################################################
# additions from packages needed for this sample
# add refs  in LIBRARIES_OPTIMIZED
# add refs  in LIBRARIES_DEBUG
# add files in PACKAGE_SOURCE_FILES
#
_add_package_OpenGL()
_add_package_VulkanSDK()
_add_package_ZLIB()
_add_package_ImGUI()

#####################################################################################
# process the rest of some cmake code that needs to be done *after* the packages add
_add_shared_sources_lib()

#####################################################################################
# Source files for this project
#
file(GLOB SOURCE_FILES *.cpp *.hpp *.inl *.h *.c mt/*.cpp mt/*.h VK_nvidia/*.c VK_nvidia/*.h)

include_directories(${CMAKE_CURRENT_SOURCE_DIR}/VK_nvidia)

#####################################################################################
# GLSL to SPIR-V custom build
#
#more than one file can be given: _compile_GLSL("GLSL_mesh.vert;GLSL_mesh.frag" "GLSL_mesh.spv" GLSL_SOURCES)
# the SpirV validator is fine as long as files are for different pipeline stages (entry points still need to be main())
#_compile_GLSL(<source(s)> <target spv> <LIST where files are appended>)
UNSET(GLSL_SOURCES)
UNSET(SPV_OUTPUT)
_compile_GLSL("GLSL/GLSL_fur.vert" "GLSL/GLSL_fur_vert.spv" GLSL_SOURCES SPV_OUTPUT)
_compile_GLSL("GLSL/GLSL_fur.frag" "GLSL/GLSL_fur_frag.spv" GLSL_SOURCES SPV_OUTPUT)
_compile_GLSL("GLSL/GLSL_passthrough.vert" "GLSL/GLSL_passthrough_vert.spv" GLSL_SOURCES SPV_OUTPUT)
_compile_GLSL("GLSL/GLSL_ds1.frag" "GLSL/GLSL_ds1_frag.spv" GLSL_SOURCES SPV_OUTPUT)
_compile_GLSL("GLSL/GLSL_ds2.frag" "GLSL/GLSL_ds2_frag.spv" GLSL_SOURCES SPV_OUTPUT)
_compile_GLSL("GLSL/GLSL_ds3.frag" "GLSL/GLSL_ds3_frag.spv" GLSL_SOURCES SPV_OUTPUT)
source_group(GLSL_Files FILES ${GLSL_SOURCES})

#####################################################################################
# additional files from helpers
#
#LIST(APPEND COMMON_SOURCE_FILES 
#  ${BASE_DIRECTORY}/shared_sources/nv_helpers_gl/WindowInertiaCamera.h
#  ${BASE_DIRECTORY}/shared_sources/nv_helpers/TimeSampler.h
#  ${BASE_DIRECTORY}/shared_sources/nv_helpers/InertiaCamera.h
#  ${BASE_DIRECTORY}/shared_sources/nv_math/nv_math.inl
#  ${BASE_DIRECTORY}/shared_sources/nv_math/nv_math.h
#  ${BASE_DIRECTORY}/shared_sources/nv_math/nv_math_types.h
#  ${BASE_DIRECTORY}/shared_sources/nv_helpers/profiler.hpp
#  ${BASE_DIRECTORY}/shared_sources/nv_helpers/profiler.cpp
#)
#####################################################################################
# Executable
#
if(WIN32 AND NOT GLUT_FOUND)
  add_definitions(/wd4267) #remove size_t to int warning
  add_definitions(/wd4996) #remove printf warning
  add_definitions(/wd4244) #remove double to float conversion warning
  add_definitions(/wd4305) #remove double to float truncation warning
  add_executable(${PROJNAME} WIN32 ${SOURCE_FILES} ${COMMON_SOURCE_FILES} ${PACKAGE_SOURCE_FILES} ${GLSL_SOURCES})
else()
  # allow gcc to be tolerant on some issues. TODO:should remove this option
  add_definitions(-fpermissive)
  add_executable(${PROJNAME} ${SOURCE_FILES} ${COMMON_SOURCE_FILES} ${PACKAGE_SOURCE_FILES} ${GLSL_SOURCES})
endif()

#####################################################################################
# common source code needed for this sample
#
source_group(common FILES 
  ${COMMON_SOURCE_FILES}
)
#####################################################################################
# Linkage
#
target_link_libraries(${PROJNAME} optimized
    ${LIBRARIES_OPTIMIZED}
    ${PLATFORM_LIBRARIES}
    shared_sources
)
target_link_libraries(${PROJNAME} debug
    ${LIBRARIES_DEBUG}
    ${PLATFORM_LIBRARIES}
    shared_sources
)

#####################################################################################
# copies binaries that need to be put next to the exe files (ZLib, etc.)
#
_copy_binaries_to_target( ${PROJNAME} )
install(FILES ${SPV_OUTPUT} CONFIGURATIONS Release DESTINATION "bin_${ARCH}/SPV_${PROJNAME}")
install(FILES ${SPV_OUTPUT} CONFIGURATIONS Debug DESTINATION "bin_${ARCH}_debug/SPV_${PROJNAME}")
