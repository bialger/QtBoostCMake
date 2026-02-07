# Error if building out of a build directory
file(TO_CMAKE_PATH "${PROJECT_BINARY_DIR}/CMakeLists.txt" LOC_PATH)
if(EXISTS "${LOC_PATH}")
        message(FATAL_ERROR "You cannot build in a source directory (or any directory with "
                "CMakeLists.txt file). Please make a build subdirectory. Feel free to "
                "remove CMakeCache.txt and CMakeFiles.")
endif()

set(BOOST_VERSION "1.83.0")
set(BOOST_VERSION_UNDERSCORE "1_83_0")
set(BOOST_SOURCE_DIR "${CMAKE_CURRENT_BINARY_DIR}/boost_${BOOST_VERSION_UNDERSCORE}")
# b2 stage (default) puts libs in <source>/stage/lib or stage/lib/<variant>/, headers stay in <source>
set(BOOST_STAGE_DIR "${BOOST_SOURCE_DIR}/stage")
set(BOOST_STAGE_LIB_DIR "${BOOST_SOURCE_DIR}/stage/lib")
set(ENV{BOOSTROOT} ${BOOST_SOURCE_DIR})

# If we already built Boost here (stage), use it directly and skip find_package/build
if(EXISTS "${BOOST_SOURCE_DIR}/boost/version.hpp")
        if(EXISTS "${BOOST_STAGE_DIR}")
                set(Boost_ROOT ${BOOST_SOURCE_DIR})
        endif()
endif()

find_package(Boost 1.83 QUIET)

if(NOT Boost_FOUND)
        message(STATUS "Boost not found. Downloading and building Boost...")

        set(BOOST_ARCHIVE_URL "https://archives.boost.io/release/${BOOST_VERSION}/source/boost_${BOOST_VERSION_UNDERSCORE}.tar.gz")
        set(BOOST_ARCHIVE_FILE "${CMAKE_CURRENT_BINARY_DIR}/boost_${BOOST_VERSION_UNDERSCORE}.tar.gz")

        message(STATUS "Downloading Boost from archives.boost.io...")
        file(DOWNLOAD ${BOOST_ARCHIVE_URL} ${BOOST_ARCHIVE_FILE}
                SHOW_PROGRESS)

        message(STATUS "Unpacking the archive...")
        execute_process(COMMAND ${CMAKE_COMMAND} -E tar xvf ${BOOST_ARCHIVE_FILE}
                WORKING_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}
                OUTPUT_QUIET
        )

        message(STATUS "Running bootstrap...")
        if(WIN32)
                execute_process(COMMAND "${BOOST_SOURCE_DIR}/bootstrap.bat" gcc
                        WORKING_DIRECTORY ${BOOST_SOURCE_DIR})
                set(B2_EXECUTABLE "${BOOST_SOURCE_DIR}/b2.exe")
        else()
                execute_process(COMMAND "${BOOST_SOURCE_DIR}/bootstrap.sh" --with-toolset=gcc
                        WORKING_DIRECTORY ${BOOST_SOURCE_DIR})
                set(B2_EXECUTABLE "${BOOST_SOURCE_DIR}/b2")
        endif()

        message(STATUS "Building Boost with b2 stage (this may take a long time)...")
        execute_process(
                COMMAND ${B2_EXECUTABLE} stage
                toolset=gcc
                address-model=64
                link=static
                runtime-link=static
                threading=multi
                --layout=versioned
                --build-type=complete
                WORKING_DIRECTORY ${BOOST_SOURCE_DIR}
        )

        set(Boost_ROOT "${BOOST_SOURCE_DIR}")
        find_package(Boost 1.83 QUIET)
else()
        message(STATUS "Boost found in the system.")
endif()

# When using our staged Boost, FindBoost does not set Boost_LIBRARIES; collect built libs from stage.
# b2 --build-type=complete produces both release (-mt-s-) and debug (-mt-sd-) variants; link only one
# to avoid multiple definition errors.
if(Boost_FOUND AND EXISTS "${BOOST_STAGE_LIB_DIR}")
        file(GLOB_RECURSE _stage_libs "${BOOST_STAGE_LIB_DIR}/*.a" "${BOOST_STAGE_LIB_DIR}/*.lib")
        if(_stage_libs)
                if(CMAKE_BUILD_TYPE STREQUAL "Debug")
                        list(FILTER _stage_libs INCLUDE REGEX "[-]mt[-]sd[-]|[-]sgd[-]")
                else()
                        list(FILTER _stage_libs EXCLUDE REGEX "[-]mt[-]sd[-]|[-]sgd[-]")
                endif()
                set(Boost_LIBRARIES ${_stage_libs})
        endif()
elseif(Boost_FOUND)
        set(Boost_LIBRARIES boost_filesystem boost_system boost_chrono boost_date_time boost_regex boost_program_options boost_serialization boost_thread boost_wave boost_iostreams boost_locale boost_unit_test_framework boost_wave boost_iostreams boost_locale boost_unit_test_framework)
        if(LINUX)
                set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} -L/usr/local/lib")
        endif()
endif()
# Do not pass NOTFOUND to the linker
if(Boost_LIBRARIES)
        list(FILTER Boost_LIBRARIES EXCLUDE REGEX "NOTFOUND$")
endif()

add_library(boost INTERFACE)
target_include_directories(boost INTERFACE ${Boost_INCLUDE_DIRS})
if(Boost_LIBRARIES)
        target_link_libraries(boost INTERFACE ${Boost_LIBRARIES})
endif()
