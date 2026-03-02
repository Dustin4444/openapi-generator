cmake_minimum_required (VERSION 3.2)

project(api-server)

set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=c++17 -pg -g3" )

add_library(warn_interface INTERFACE)
if(MSVC)
    target_compile_options(warn_interface INTERFACE
        /W3
        /wd4018
        /wd4244
        /wd4267
        /wd4715
        /wd4805
    )
    target_compile_definitions(warn_interface INTERFACE
        _CRT_SECURE_NO_WARNINGS
        _SILENCE_CXX17_CODECVT_HEADER_DEPRECATION_WARNING
    )
else()
    include(CheckCXXCompilerFlag)
    foreach(_flag -Wall -Wextra -Wno-unused-variable)
        string(MAKE_C_IDENTIFIER "HAS${_flag}" _flag_var)
        check_cxx_compiler_flag("${_flag}" ${_flag_var})
        if(${_flag_var})
            target_compile_options(warn_interface INTERFACE ${_flag})
        endif()
    endforeach()
    # Some compilers will ignore -Wformat-security without -Wformat, so combine them here.
    check_cxx_compiler_flag("-Wformat -Wformat-security" HAS_WFORMAT_SECURITY)
    if(HAS_WFORMAT_SECURITY)
        target_compile_options(warn_interface INTERFACE -Wformat -Wformat-security)
    endif()
    check_cxx_compiler_flag("-Wunused-parameter" HAS_WUNUSED_PARAMETER)
    if(HAS_WUNUSED_PARAMETER)
        target_compile_options(warn_interface INTERFACE -Wno-unused-parameter)
    endif()
endif()

include(ExternalProject)

set(EXTERNAL_INSTALL_LOCATION ${CMAKE_CURRENT_SOURCE_DIR}/external)

ExternalProject_Add(PISTACHE
    GIT_REPOSITORY https://github.com/pistacheio/pistache.git
    BUILD_IN_SOURCE true
    INSTALL_COMMAND meson setup build --prefix=${EXTERNAL_INSTALL_LOCATION} --libdir=lib && meson install -C build
)

ExternalProject_Add(NLOHMANN
    GIT_REPOSITORY https://github.com/nlohmann/json.git
    CMAKE_ARGS -DCMAKE_INSTALL_PREFIX=${EXTERNAL_INSTALL_LOCATION} -DJSON_BuildTests=OFF
)

include_directories(${EXTERNAL_INSTALL_LOCATION}/include)
link_directories(${EXTERNAL_INSTALL_LOCATION}/lib)

include_directories(model)
include_directories(api)
include_directories(impl)

file(GLOB SRCS
    ${CMAKE_CURRENT_SOURCE_DIR}/api/*.cpp
    ${CMAKE_CURRENT_SOURCE_DIR}/impl/*.cpp
    ${CMAKE_CURRENT_SOURCE_DIR}/model/*.cpp
    ${CMAKE_CURRENT_SOURCE_DIR}/*.cpp
)

add_executable(${PROJECT_NAME} ${SRCS} )
add_dependencies(${PROJECT_NAME} PISTACHE NLOHMANN)
target_link_libraries(${PROJECT_NAME} pistache pthread warn_interface)
