CMAKE_MINIMUM_REQUIRED(VERSION 3.7)

PROJECT(
    carma
    VERSION 0.3.0
    LANGUAGES CXX
)

# ##############################################################################
#                                   SETTINGS                                   #
# ##############################################################################
SET(CMAKE_CXX_STANDARD 14)
SET(DEFAULT_BUILD_TYPE "RELEASE")
IF(NOT CMAKE_BUILD_TYPE AND NOT CMAKE_CONFIGURATION_TYPES)
  MESSAGE(STATUS "Setting build type to '${DEFAULT_BUILD_TYPE}' as none was specified.")
  SET(CMAKE_BUILD_TYPE "${DEFAULT_BUILD_TYPE}" CACHE STRING "Choose the type of build." FORCE)
  # Set the possible values of build type for cmake-gui
  SET_PROPERTY(CACHE CMAKE_BUILD_TYPE PROPERTY STRINGS
    "Debug" "Release" "MinSizeRel" "RelWithDebInfo")
ENDIF()

OPTION(BUILD_TESTS "Build tests for carma" OFF)
OPTION(BUILD_EXAMPLES "Build examples for carma" OFF)

# ##############################################################################
#                                 DEFINITIONS                                  #
# ##############################################################################
#ADD_DEFINITIONS(-DARMA_DONT_USE_WRAPPER)

# Don't copy arrays if the data is not owned by Numpy
# ADD_DEFINITIONS(-DCARMA_DONT_REQUIRE_OWNDATA)

# Don't copy c-style arrays definition
# ADD_DEFINITIONS(-DCARMA_DONT_REQUIRE_F_CONTIGUOUS)

# ##############################################################################
#                                   TARGET                                     #
# ##############################################################################
if (CMAKE_CXX_COMPILER_ID STREQUAL "GNU" 
        OR CMAKE_CXX_COMPILER_ID STREQUAL "Clang" 
        OR CMAKE_CXX_COMPILER_ID STREQUAL "AppleClang")
    SET(PROJECT_RELEASE_FLAGS "-march=native" "-mtune=native")
elseif (CMAKE_CXX_COMPILER_ID MATCHES "MSVC")
    # workaround like https://github.com/nlohmann/json/issues/1408
    # to avoid error like: carma\third_party\armadillo-code\include\armadillo_bits/arma_str.hpp(194): error C2039: '_snprintf': is not a member of 'std' (compiling source file carma\tests\src\bindings.cpp) 
    ADD_DEFINITIONS(-DHAVE_SNPRINTF)
endif ()
SET(PROJECT_RELEASE_DEFINITIONS ARMA_NO_DEBUG)
# see linkg for a more expensive approach https://github.com/VcDevel/Vc/blob/master/cmake/OptimizeForArchitecture.cmake

find_package(Armadillo REQUIRED)
find_package(pybind11 REQUIRED)

ADD_LIBRARY(carma INTERFACE)
TARGET_LINK_LIBRARIES(carma INTERFACE ${pybind11_LIBRARIES} ${ARMADILLO_LIBRARIES})
TARGET_INCLUDE_DIRECTORIES(carma
        INTERFACE
        ${pybind11_INCLUDE_DIRS}
        ${ARMADILLO_INCLUDE_DIRS}
        # path of the headers after installation
        $<INSTALL_INTERFACE:include> # <prefix>/include/carma
        # path of the headers before installation
        $<BUILD_INTERFACE:${PROJECT_SOURCE_DIR}/include>
        )
INSTALL(DIRECTORY "${PROJECT_SOURCE_DIR}/include" 
        DESTINATION .
        FILES_MATCHING PATTERN "*.h")

if (VALGRIND_TEST_WRAPPER)
    if (WIN32)
        MESSAGE(FATAL_ERROR "VALGRIND_TEST_WRAPPER is not supported in WIN32 environment")
    endif()
    find_program(VALGRIND_EXECUTABLE valgrind)
    if (NOT VALGRIND_EXECUTABLE)
        MESSAGE(FATAL_ERROR "VALGRIND_TEST_WRAPPER is requested but valgrind executable cannot be found")
    endif()
    MESSAGE(STATUS "Valgrind test mode enabled")
endif()

ENABLE_TESTING()

IF(BUILD_TESTS)
    add_subdirectory(tests)
ENDIF()
IF(BUILD_EXAMPLES)
    add_subdirectory(examples)
ENDIF()

#------------------------------------------------------

## search for clang-format and add target
if(NOT DEFINED CARMA_DEV_TARGET OR CARMA_DEV_TARGET)
    find_program(CLANG_FORMAT clang-format)
    if (CLANG_FORMAT)
        exec_program(${CLANG_FORMAT} ARGS -version
                OUTPUT_VARIABLE CLANG_FORMAT_RAW_VERSION)
        string(REGEX MATCH "[1-9][0-9]*\\.[0-9]+\\.[0-9]+"
                CLANG_FORMAT_VERSION ${CLANG_FORMAT_RAW_VERSION})
        if (CLANG_FORMAT_VERSION VERSION_GREATER_EQUAL "6.0.0") 
            add_custom_target(clang-format
                    COMMAND echo "running ${CLANG_FORMAT} ..."
                    COMMAND ${CMAKE_COMMAND}
                    -DPROJECT_SOURCE_DIR="${PROJECT_SOURCE_DIR}"
                    -DCLANG_FORMAT="${CLANG_FORMAT}"
                    -P ${PROJECT_SOURCE_DIR}/cmake/ClangFormatProcess.cmake)
            message(STATUS "clang-format target for updating code format is available")
        else()
            message(WARNING "incompatible clang-format found (<6.0.0); clang-format target is not available.")
            add_custom_target(clang-format
                    COMMAND ${CMAKE_COMMAND} -E echo ""
                    COMMAND ${CMAKE_COMMAND} -E echo "*** code formatting not available since clang-format version is incompatible ***"
                    COMMAND ${CMAKE_COMMAND} -E echo ""
                    COMMENT "Inform about not available code format."
                    )
        endif()
    else ()
        message(WARNING "clang-format no found; clang-format target is not available.")
        add_custom_target(clang-format
                COMMAND ${CMAKE_COMMAND} -E echo ""
                COMMAND ${CMAKE_COMMAND} -E echo "*** code formatting not available since clang-format has not been found ***"
                COMMAND ${CMAKE_COMMAND} -E echo ""
                COMMENT "Inform about not available code format."
                )
    endif ()
endif()

#
# Install support
#
include(CMakePackageConfigHelpers)
set(carma_INCLUDE_DIR include/
  CACHE PATH "The directory where carma headers are installed")
set(carma_CONFIG_CMAKE_PATH lib${LIB_SUFFIX}/cmake/carma
  CACHE PATH "The directory where the CMake files are installed")

configure_package_config_file(carmaConfig.cmake.in
  ${CMAKE_CURRENT_BINARY_DIR}/carmaConfig.cmake
  INSTALL_DESTINATION ${carma_CONFIG_CMAKE_PATH}
  PATH_VARS carma_INCLUDE_DIR)
write_basic_package_version_file(
  ${CMAKE_CURRENT_BINARY_DIR}/carmaConfigVersion.cmake
  VERSION ${PROJECT_VERSION}
  COMPATIBILITY SameMajorVersion)

# Install targets
install(DIRECTORY include/carma DESTINATION include)
install(FILES
  ${CMAKE_CURRENT_BINARY_DIR}/carmaConfig.cmake
  ${CMAKE_CURRENT_BINARY_DIR}/carmaConfigVersion.cmake
  DESTINATION ${carma_CONFIG_CMAKE_PATH})
