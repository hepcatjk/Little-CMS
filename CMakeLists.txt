cmake_minimum_required(VERSION 3.10)
project(lcms2 VERSION 2.17 LANGUAGES C)

set(CMAKE_C_STANDARD 99)
set(CMAKE_C_STANDARD_REQUIRED ON)

option(BUILD_SHARED_LIBS "Build shared libraries" ON)
option(BUILD_STATIC_LIBS "Build static libraries" ON)
option(BUILD_TOOLS "Build utility tools (requires JPEG and TIFF)" OFF)
option(BUILD_TESTS "Build test suite" OFF)
option(BUILD_PLUGINS "Build plugin samples" OFF)

if(NOT BUILD_SHARED_LIBS AND NOT BUILD_STATIC_LIBS)
    message(FATAL_ERROR "At least one of BUILD_SHARED_LIBS or BUILD_STATIC_LIBS must be enabled")
endif()

add_definitions(-DLCMS2_VERSION_MAJOR=${PROJECT_VERSION_MAJOR})
add_definitions(-DLCMS2_VERSION_MINOR=${PROJECT_VERSION_MINOR})
add_definitions(-DLCMS2_VERSION_STR="${PROJECT_VERSION}")

include_directories(${CMAKE_CURRENT_SOURCE_DIR}/include)

file(GLOB LCMS2_SOURCES ${CMAKE_CURRENT_SOURCE_DIR}/src/*.c)
file(GLOB LCMS2_HEADERS ${CMAKE_CURRENT_SOURCE_DIR}/include/*.h)

if(WIN32)
    configure_file(
        ${CMAKE_CURRENT_SOURCE_DIR}/src/lcms2.rc.in
        ${CMAKE_CURRENT_BINARY_DIR}/lcms2.rc
        @ONLY
    )
    list(APPEND LCMS2_SOURCES ${CMAKE_CURRENT_BINARY_DIR}/lcms2.rc src/lcms2.def)
    add_definitions(-DCMS_RELY_ON_WINDOWS_STATIC_MUTEX_INIT)
endif()

include(TestBigEndian)
test_big_endian(IS_BIG_ENDIAN)
if(IS_BIG_ENDIAN)
    add_definitions(-DCMS_USE_BIG_ENDIAN=1)
endif()

set(CMAKE_THREAD_PREFER_PTHREAD TRUE)
set(THREADS_PREFER_PTHREAD_FLAG TRUE)
find_package(Threads)
if(Threads_FOUND AND NOT WIN32)
    add_definitions(-DCMS_USE_PTHREADS=1)
endif()

if(BUILD_SHARED_LIBS)
    add_library(lcms2_shared SHARED ${LCMS2_SOURCES} ${LCMS2_HEADERS})
    set_target_properties(lcms2_shared PROPERTIES
        OUTPUT_NAME lcms2
        VERSION ${PROJECT_VERSION}
        SOVERSION ${PROJECT_VERSION_MAJOR}
    )
    if(Threads_FOUND)
        target_link_libraries(lcms2_shared PRIVATE Threads::Threads)
    endif()
endif()

if(BUILD_STATIC_LIBS)
    add_library(lcms2_static STATIC ${LCMS2_SOURCES} ${LCMS2_HEADERS})
    set_target_properties(lcms2_static PROPERTIES
        OUTPUT_NAME lcms2
    )
    if(Threads_FOUND)
        target_link_libraries(lcms2_static PRIVATE Threads::Threads)
    endif()
endif()

if(BUILD_SHARED_LIBS)
    install(TARGETS lcms2_shared
        RUNTIME DESTINATION bin
        LIBRARY DESTINATION lib
        ARCHIVE DESTINATION lib
    )
endif()
if(BUILD_STATIC_LIBS)
    install(TARGETS lcms2_static
        ARCHIVE DESTINATION lib
    )
endif()
install(FILES ${LCMS2_HEADERS} DESTINATION include)

if(BUILD_TOOLS)
    find_package(JPEG)
    find_package(TIFF)
    if(JPEG_FOUND AND TIFF_FOUND)
        file(GLOB UTILS_COMMON_SOURCES ${CMAKE_CURRENT_SOURCE_DIR}/utils/common/*.c)
        set(TOOLS jpgicc linkicc psicc tifficc transicc)
        foreach(TOOL ${TOOLS})
            add_executable(${TOOL} utils/${TOOL}/${TOOL}.c ${UTILS_COMMON_SOURCES})
            target_include_directories(${TOOL} PRIVATE ${CMAKE_CURRENT_SOURCE_DIR}/utils/common)
            if(BUILD_SHARED_LIBS)
                target_link_libraries(${TOOL} PRIVATE lcms2_shared JPEG::JPEG TIFF::TIFF)
            else()
                target_link_libraries(${TOOL} PRIVATE lcms2_static JPEG::JPEG TIFF::TIFF)
            endif()
            install(TARGETS ${TOOL} RUNTIME DESTINATION bin)
        endforeach()
    else()
        message(WARNING "JPEG or TIFF not found, skipping tools build")
        set(BUILD_TOOLS OFF)
    endif()
endif()

if(BUILD_TESTS)
    enable_testing()
    file(GLOB TEST_SOURCES ${CMAKE_CURRENT_SOURCE_DIR}/testbed/*.c)
    foreach(TEST_SOURCE ${TEST_SOURCES})
        get_filename_component(TEST_NAME ${TEST_SOURCE} NAME_WE)
        add_executable(test_${TEST_NAME} ${TEST_SOURCE})
        if(BUILD_SHARED_LIBS)
            target_link_libraries(test_${TEST_NAME} PRIVATE lcms2_shared)
        else()
            target_link_libraries(test_${TEST_NAME} PRIVATE lcms2_static)
        endif()
        add_test(NAME ${TEST_NAME} COMMAND test_${TEST_NAME})
    endforeach()
endif()

if(BUILD_PLUGINS)
    file(GLOB PLUGIN_SOURCES ${CMAKE_CURRENT_SOURCE_DIR}/plugins/*.c)
    foreach(PLUGIN_SOURCE ${PLUGIN_SOURCES})
        get_filename_component(PLUGIN_NAME ${PLUGIN_SOURCE} NAME_WE)
        add_library(${PLUGIN_NAME} SHARED ${PLUGIN_SOURCE})
        if(BUILD_SHARED_LIBS)
            target_link_libraries(${PLUGIN_NAME} PRIVATE lcms2_shared)
        else()
            target_link_libraries(${PLUGIN_NAME} PRIVATE lcms2_static)
        endif()
        set_target_properties(${PLUGIN_NAME} PROPERTIES
            PREFIX ""
            OUTPUT_NAME ${PLUGIN_NAME}
        )
        install(TARGETS ${PLUGIN_NAME} LIBRARY DESTINATION lib/lcms2/plugins)
    endforeach()
endif()

# Note: Fuzzers in fuzzers/ are not built here, as they require specialized tools like libFuzzer.
# To build fuzzers, use the appropriate fuzzing environment (e.g., OSS-Fuzz).

configure_file(
    ${CMAKE_CURRENT_SOURCE_DIR}/lcms2.pc.in
    ${CMAKE_CURRENT_BINARY_DIR}/lcms2.pc
    @ONLY
)
install(FILES ${CMAKE_CURRENT_BINARY_DIR}/lcms2.pc DESTINATION lib/pkgconfig)

include(FeatureSummary)
feature_summary(WHAT ALL)