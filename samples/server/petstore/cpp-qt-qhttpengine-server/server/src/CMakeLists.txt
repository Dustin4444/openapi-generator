cmake_minimum_required(VERSION 3.2 FATAL_ERROR)

set(CMAKE_VERBOSE_MAKEFILE ON)
set(CMAKE_INCLUDE_CURRENT_DIR ON)
set(CMAKE_AUTOMOC ON)

OPTION(NODEBUG  "Deactivate No debugging option" "OFF")

set(CMAKE_C_FLAGS   "${CMAKE_C_FLAGS}   -fPIC")
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fPIC -std=c++14")

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

if(${NODEBUG} STREQUAL "OFF")
set(CMAKE_C_FLAGS   "${CMAKE_C_FLAGS}   -pg -g3")
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -pg -g3")
else (${NODEBUG} STREQUAL "OFF")
set(CMAKE_C_FLAGS   "${CMAKE_C_FLAGS}   -s -O3")
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -s -O3")
endif(${NODEBUG} STREQUAL "OFF")

find_package(Qt5Core REQUIRED)
find_package(Qt5Network REQUIRED)

file(GLOB SRCS
    ${CMAKE_CURRENT_SOURCE_DIR}/models/*.cpp
    ${CMAKE_CURRENT_SOURCE_DIR}/handlers/*.cpp
    ${CMAKE_CURRENT_SOURCE_DIR}/requests/*.cpp
    ${CMAKE_CURRENT_SOURCE_DIR}/main.cpp
)

include_directories(
    ${Qt5Core_INCLUDE_DIRS}
    ${Qt5Network_INCLUDE_DIRS}
    ${CMAKE_CURRENT_SOURCE_DIR}/models
    ${CMAKE_CURRENT_SOURCE_DIR}/handlers
    ${CMAKE_CURRENT_SOURCE_DIR}/requests
)

link_directories(
    ${CMAKE_PREFIX_PATH}/lib
)

add_executable(${PROJECT_NAME} ${SRCS})
add_dependencies(${PROJECT_NAME} QHTTPENGINE)
target_link_libraries(${PROJECT_NAME} Qt5Core Qt5Network ssl crypto qhttpengine warn_interface)
set_property(TARGET ${PROJECT_NAME} PROPERTY CXX_STANDARD 14)
set_property(TARGET ${PROJECT_NAME} PROPERTY CXX_STANDARD_REQUIRED ON)

install(TARGETS ${PROJECT_NAME} RUNTIME DESTINATION bin LIBRARY DESTINATION lib ARCHIVE DESTINATION lib)
