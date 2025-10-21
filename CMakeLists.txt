cmake_minimum_required(VERSION 3.10)

set(CMAKE_INSTALL_PREFIX "${CMAKE_BINARY_DIR}/install" CACHE STRING "Final install location")
set(CMAKE_BUILD_TYPE_INIT "Release")
set(CMAKE_CXX_STANDARD 14)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

project(cloudparse
    VERSION 0.2.1
    DESCRIPTION "Cloud parse library for PCL"
    LANGUAGES CXX
)

option(BUILD_TESTING "Builds only the test executable." OFF)
option(CODE_COVERAGE "Collect coverage from test library" OFF)
option(BUILD_SHARED_LIBS "Build shared libraries" ON)

include("${CMAKE_CURRENT_LIST_DIR}/cmake/color.cmake")
include(ExternalProject)

find_package(PCL 1.8 REQUIRED QUIET COMPONENTS io)

if(PCL_FOUND)
    message(STATUS "PCL status:")
    message(STATUS "    version: ${PCL_VERSION}")
    message(STATUS "    directory: ${PCL_DIR}")
else()
    message(FATAL_ERROR "   ERROR: PCL minimum required version 1.8. Not found")
endif()

# #############################################################################
# COMPILE LIBRARY
# #############################################################################
add_library(${PROJECT_NAME}
    ${CMAKE_CURRENT_LIST_DIR}/src/parser.cpp
)
add_library(${PROJECT_NAME}::${PROJECT_NAME} ALIAS ${PROJECT_NAME})

# #############################################################################
# COMPILE UNITTESTS
# #############################################################################
if(BUILD_TESTING)
    message("${BoldWhite} Testing enabled ${ColourReset}")
    enable_testing()
    add_subdirectory(external)
    add_subdirectory(tests)

    if(CODE_COVERAGE)
        message("${BoldWhite} Coverage enabled ${ColourReset}")
        include("${CMAKE_CURRENT_LIST_DIR}/cmake/CodeCoverage.cmake")
        append_coverage_compiler_flags()
        setup_target_for_coverage_lcov(NAME coverage EXECUTABLE ctest -j ${n_cores}
            BASE_DIRECTORY ../coverage
            DEPENDENCIES
            testlib
            testconcreteparses
        )
    endif()
endif()

# #############################################################################
# LIBRARY PROPERTIES
# #############################################################################
set_target_properties(${PROJECT_NAME} PROPERTIES
    VERSION ${PROJECT_VERSION}
    SOVERSION 1
    PUBLIC_HEADER ${CMAKE_CURRENT_LIST_DIR}/include/cloudparse/parser.hpp
)

# #############################################################################
# LIBRARY HEADERS
# #############################################################################
target_include_directories(${PROJECT_NAME} PUBLIC
    $<BUILD_INTERFACE:${CMAKE_CURRENT_LIST_DIR}/include>
    $<INSTALL_INTERFACE:include>
    ${PCL_INCLUDE_DIRS}
)

# #############################################################################
# LIBRARY DEPENDENCIES
# #############################################################################
target_link_libraries(${PROJECT_NAME} PRIVATE ${PCL_IO_LIBRARIES})

# #############################################################################
# COMPILATION FLAGS
# #############################################################################
if(MSVC)
    # MSVC-specific flags
    target_compile_options(${PROJECT_NAME} PRIVATE
        /W3                    # Warning level 3
        /wd4996               # Disable deprecation warnings
        /permissive-          # Standards conformance (enables alternative operators: or, and, not)
    )
    target_compile_definitions(${PROJECT_NAME} PRIVATE
        _CRT_SECURE_NO_WARNINGS
        _SCL_SECURE_NO_WARNINGS
    )
elseif(CMAKE_CXX_COMPILER_ID MATCHES "GNU|Clang")
    # GCC/Clang-specific flags
    target_compile_options(${PROJECT_NAME} PRIVATE
        -Wno-cpp
        $<$<NOT:$<PLATFORM_ID:Darwin>>:-mno-avx>  # Disable AVX on non-macOS x86 platforms
        $<$<NOT:$<PLATFORM_ID:Darwin>>:-mno-avx2>
    )
endif()

# #############################################################################
# INSTALL DIRECTORY
# #############################################################################
include(GNUInstallDirs)

install(TARGETS ${PROJECT_NAME}
    EXPORT ${PROJECT_NAME}Targets
    RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR}
    LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}
    ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR}
    PUBLIC_HEADER DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}/cloudparse
)

# Install all headers from include directory
install(DIRECTORY ${CMAKE_CURRENT_LIST_DIR}/include/
    DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}
    FILES_MATCHING PATTERN "*.hpp" PATTERN "*.h"
)

# Export targets for use by add_subdirectory (no config file needed)
export(EXPORT ${PROJECT_NAME}Targets
    FILE ${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_NAME}Targets.cmake
    NAMESPACE ${PROJECT_NAME}::
)
