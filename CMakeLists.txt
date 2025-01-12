cmake_minimum_required(VERSION 3.15)

project(realm)

# Project-wide build flags
set(CMAKE_CXX_STANDARD 20)

add_subdirectory(realm-core)

set(REALM_INCLUDE_CERTS 0)
set(REALM_NO_CONFIG)
set(REALM_ENABLE_SYNC)
set(REALM_NO_TESTS)
set(REALM_BUILD_LIB_ONLY)
set(REALM_INSTALL_LIBEXECDIR)
set(REALM_ENABLE_ENCRYPTION 1)

add_compile_definitions(REALM_NO_CONFIG)
add_compile_definitions(REALM_ENABLE_SYNC)
add_compile_definitions(REALM_ENABLE_ENCRYPTION)
add_compile_definitions(REALM_INSTALL_LIBEXECDIR)
add_compile_definitions(REALM_BUILD_LIB_ONLY)
add_compile_definitions(REALM_VERSION_MAJOR=0)
add_compile_definitions(REALM_VERSION_MINOR=0)
add_compile_definitions(REALM_VERSION_PATCH=1)

set(SOURCES
    src/cpprealm/sdk.cpp
    src/cpprealm/object.cpp
) # REALM_SOURCES

set(HEADERS
    src/cpprealm/sdk.hpp
    src/cpprealm/app.hpp
    src/cpprealm/db.hpp
    src/cpprealm/notifications.hpp
    src/cpprealm/object.hpp
    src/cpprealm/persisted.hpp
    src/cpprealm/results.hpp
    src/cpprealm/schema.hpp
    src/cpprealm/task.hpp
    src/cpprealm/thread_safe_reference.hpp
    src/cpprealm/type_info.hpp
    src/cpprealm/flex_sync.hpp
) # REALM_INSTALL_HEADERS

add_library(cpprealm STATIC ${SOURCES} ${HEADERS})
add_executable(cpprealm_exe_tests tests/tests.cpp tests/str_tests.cpp tests/list_tests.cpp tests/query_tests.cpp tests/test_utils.hpp tests/sync_test_utils.hpp tests/test_objects.hpp tests/test_utils.cpp)
#add_test(cpprealm_tests)

target_include_directories(cpprealm PRIVATE realm-core/src)
target_include_directories(cpprealm PUBLIC /usr/local/include)
target_include_directories(cpprealm PUBLIC /usr/include/c++/10)
target_include_directories(cpprealm PUBLIC src)
target_include_directories(cpprealm_exe_tests PUBLIC src)
target_include_directories(cpprealm_exe_tests PUBLIC realm-core/src)

target_sources(cpprealm PRIVATE ${SOURCES})
set_property(TARGET cpprealm PROPERTY CXX_STANDARD 20)
set_property(TARGET Storage PROPERTY REALM_BUILD_LIB_ONLY 1)
target_compile_options(cpprealm PUBLIC -std=c++2a)

# on Apple platforms we use the built-in CFRunLoop
# everywhere else it's libuv, except UWP where it doesn't build
if(NOT APPLE AND NOT WINDOWS_STORE)
    set(REALM_HAVE_UV 1)

    option(REALM_FETCH_MISSING_DEPENDENCIES "Download missing dependencies with CMake's FetchContent where possible" ON)
    if(REALM_FETCH_MISSING_DEPENDENCIES)
        find_package(LibUV)
    else()
        find_package(LibUV REQUIRED)
    endif()
    if(LibUV_FOUND)
        set(libuv_target LibUV::LibUV)
    elseif(REALM_FETCH_MISSING_DEPENDENCIES)
        message(STATUS "LibUV not found, building from source with FetchContent")
        include(FetchContent)
        set(libUV_Git_TAG "v1.35.0")
        if(MSVC)
            set(liUV_Git_TAG "v1.43.0")
        endif()
        FetchContent_Declare(
            libuv
            GIT_REPOSITORY https://github.com/libuv/libuv.git
            GIT_TAG ${libUV_Git_TAG}
        )
        FetchContent_Populate(libuv)
        add_subdirectory(${libuv_SOURCE_DIR} ${libuv_BINARY_DIR} EXCLUDE_FROM_ALL)
        set(libuv_target uv_a)
    endif()
    
    target_link_libraries(cpprealm ${libuv_target})
    get_property(libuv_include_dir TARGET ${libuv_target} PROPERTY INCLUDE_DIRECTORIES)

    target_compile_definitions(cpprealm PUBLIC REALM_DISABLE_METADATA_ENCRYPTION=1 REALM_ENABLE_SYNC=1)
    target_compile_definitions(ObjectStore PUBLIC REALM_HAVE_UV=1)
    target_include_directories(ObjectStore PRIVATE ${libuv_include_dir})

else()
    target_compile_definitions(cpprealm PUBLIC REALM_ENABLE_SYNC=1)
endif()

if(CMAKE_COMPILER_IS_GNUCXX AND CMAKE_CXX_COMPILER_VERSION VERSION_GREATER_EQUAL 10 AND CMAKE_CXX_COMPILER_VERSION VERSION_LESS 11)
target_compile_options(cpprealm PUBLIC -fcoroutines)
endif()

target_link_libraries(cpprealm Storage ObjectStore Sync z curl)

set_target_properties(cpprealm PROPERTIES PUBLIC_HEADER "${HEADERS}")

install(TARGETS cpprealm
        PUBLIC_HEADER DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}/cpprealm)

target_link_libraries(cpprealm_exe_tests PUBLIC cpprealm z curl ObjectStore Sync Storage)

add_test(cpprealm_tests cpprealm_exe_tests)
enable_testing()