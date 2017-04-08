cmake_minimum_required (VERSION 3.3.2)

######### Hunter VERSION - START

string(COMPARE NOTEQUAL "$ENV{TRAVIS_TAG}" "" travis_deploy)
string(COMPARE EQUAL "$ENV{APPVEYOR_REPO_TAG}" "true" appveyor_deploy)

if(travis_deploy)
  set(version "$ENV{TRAVIS_TAG}")
elseif(appveyor_deploy)
  set(version "$ENV{APPVEYOR_REPO_TAG_NAME}")
else()
  set(version "v5.3.5")
endif()

string(REGEX REPLACE "^v" "" version "${version}")
string(REGEX REPLACE "-"  "." version "${version}")
string(STRIP ${version} version)                    # https://cmake.org/cmake/help/v3.7/command/string.html

#################################### Hunter VERSION - END

######### Hunter GATE - START

### Hunter snapshot that will be used ###
include("cmake/HunterGate.cmake")

HunterGate(
  URL "https://github.com/ruslo/hunter/archive/v0.18.39.tar.gz"
  SHA1 "a6fbc056c3d9d7acdaa0a07c575c9352951c2f6c"
)

#################################### Hunter GATE - END

# if(NOT CMAKE_BUILD_TYPE)
#     set(CMAKE_BUILD_TYPE Release
#         CACHE STRING "Set build type to Debug o Release (default Release)" FORCE)
# endif()
# string(TOLOWER "${CMAKE_BUILD_TYPE}" CMAKE_BUILD_TYPE_LOWER)

# get rid of the extra default configurations
# what? why would you get id of other useful build types? - Ellzey
# set(CMAKE_CONFIGURATION_TYPES "Debug;Release" CACHE STRING "Limited configurations" FORCE)

project(Lua VERSION ${version})

list(APPEND CMAKE_MODULE_PATH "${CMAKE_CURRENT_LIST_DIR}/cmake/Modules")

set_property(GLOBAL PROPERTY USE_FOLDERS ON)

set(is_android ${ANDROID}) # syntax compatibility
string(COMPARE EQUAL "${CMAKE_OSX_SYSROOT}" "iphoneos" is_ios)
string(COMPARE EQUAL "${CMAKE_SYSTEM_NAME}" "Linux" is_linux)
string(COMPARE EQUAL "${CMAKE_GENERATOR}" "Xcode" is_xcode)

# See: cmake/Hunter/config.cmake
# First call to 'hunter_add_package' will contains read of Hunter/config.cmake
# Next variables will be used:
#   * is_ios
#   * is_linux
hunter_add_package(Sugar)
include("${SUGAR_ROOT}/cmake/Sugar")
include(sugar_include)

if(is_xcode)
  option(${PROJECT_NAME}_DISABLE_DSYM "Disable generation of dSYM information" ON)
  if(${PROJECT_NAME}_DISABLE_DSYM)
    set(CMAKE_XCODE_ATTRIBUTE_DEBUG_INFORMATION_FORMAT "dwarf")
  endif()
  set(CMAKE_CONFIGURATION_TYPES "Debug;Release;MinSizeRel;RelWithDebInfo" CACHE STRING "Configuration types")
  set(CMAKE_XCODE_ATTRIBUTE_ENABLE_BITCODE NO)
endif()



### Library ###
set(
  LUA_SOURCES
    src/lapi.c
    src/lcode.c
    src/lctype.c
    src/ldebug.c
    src/ldo.c
    src/ldump.c
    src/lfunc.c
    src/lgc.c
    src/llex.c
    src/lmem.c
    src/lobject.c
    src/lopcodes.c
    src/lparser.c
    src/lstate.c
    src/lstring.c
    src/ltable.c
    src/ltm.c
    src/lundump.c
    src/lvm.c
    src/lzio.c
    src/lauxlib.c
    src/lbaselib.c
    src/lbitlib.c
    src/lcorolib.c
    src/ldblib.c
    src/liolib.c
    src/lmathlib.c
    src/loslib.c
    src/lstrlib.c
    src/ltablib.c
    src/lutf8lib.c
    src/loadlib.c
    src/linit.c

    src/lopcodes.h
    src/lparser.h
    src/lprefix.h
    src/lzio.h
    src/lstate.h
    src/lmem.h
    src/lstring.h
    src/llimits.h
    src/llex.h
    src/ltable.h
    src/lgc.h
    src/lfunc.h
    src/ltm.h
    src/ldo.h
    src/lua.h
    src/lapi.h
    src/ldebug.h
    src/luaconf.h
    src/lualib.h
    src/lctype.h
    src/lundump.h
    src/lcode.h
    src/lauxlib.h
    src/lvm.h
    src/lobject.h
    src/lua.hpp
)

add_library(lua_lib STATIC ${LUA_SOURCES})
add_library(lua_lib-shared SHARED ${LUA_SOURCES})
set_target_properties(lua_lib-shared PROPERTIES OUTPUT_NAME lua_lib)

set_property(TARGET lua_lib PROPERTY VERSION ${Lua_VERSION})

target_include_directories(lua_lib-shared
    PUBLIC
    "$<BUILD_INTERFACE:${CMAKE_CURRENT_LIST_DIR}/src>"
)

target_include_directories(lua_lib
    PUBLIC
    "$<BUILD_INTERFACE:${CMAKE_CURRENT_LIST_DIR}/src>"
)

include(CheckLibraryExists) # check_library_exists


# Some platforms (e.g. Linux) need separate math library
check_library_exists(m pow "" LIB_M_REQUIRED)
if(LIB_M_REQUIRED)
  target_link_libraries(lua_lib PRIVATE m)
endif()


### Interpreter ###

add_executable(lua src/lua.c)
target_link_libraries(lua lua_lib)


### Compiler ###

add_executable(luac src/luac.c)

target_link_libraries(luac lua_lib)

string(COMPARE EQUAL "${CMAKE_SYSTEM_NAME}" "Linux" is_linux)
if(is_linux)
  set_target_properties(
      lua luac
      PROPERTIES
      INSTALL_RPATH "\$ORIGIN/../lib"
  )
elseif(APPLE)
  set_target_properties(
      lua luac
      PROPERTIES
      INSTALL_RPATH "@executable_path/../lib"
  )
endif()

### Installation (https://github.com/forexample/package-example) ###

set(config_install_dir "lib/cmake/${PROJECT_NAME}")

set(generated_dir "${CMAKE_CURRENT_BINARY_DIR}/generated")

set(version_config "${generated_dir}/${PROJECT_NAME}ConfigVersion.cmake")
set(project_config "${generated_dir}/${PROJECT_NAME}Config.cmake")
set(targets_export_name "${PROJECT_NAME}Targets")
set(namespace "${PROJECT_NAME}::")

install(TARGETS lua_lib lua_lib-shared lua luac
    EXPORT "${targets_export_name}"
    LIBRARY DESTINATION lib
    RUNTIME DESTINATION bin
    ARCHIVE DESTINATION lib
    INCLUDES DESTINATION include
)

install(
    FILES
    src/lua.h
    src/lua.hpp
    src/lualib.h
    src/lauxlib.h
    src/luaconf.h
    DESTINATION
    include
)

include(CMakePackageConfigHelpers)

write_basic_package_version_file(
    "${version_config}" COMPATIBILITY AnyNewerVersion
)

# Use 'targets_export_name'
configure_package_config_file(
    "cmake/Config.cmake.in"
    "${project_config}"
    INSTALL_DESTINATION "${config_install_dir}"
)

install(
    FILES "${project_config}" "${version_config}"
    DESTINATION "${config_install_dir}"
)

install(
    EXPORT "${targets_export_name}"
    NAMESPACE "${namespace}"
    DESTINATION "${config_install_dir}"
)
