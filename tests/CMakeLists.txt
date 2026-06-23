include(FetchContent)

# get location of google test suite
FetchContent_Declare(
  googletest
  URL https://github.com/google/googletest/archive/refs/tags/v1.14.0.zip
)

# allow use of google test suite
FetchContent_MakeAvailable(googletest)

# get all cpp files in tests directory
file(GLOB_RECURSE TEST_SOURCES "*.cpp")

# make a target called harp_tests and an executable comprised of all cpp files in tests directory
add_executable(harp_tests
    ${TEST_SOURCES}
)

# linking harp_tests to gtest_main harp makes sure  
# gtest_main and harp are compiled before harp_tests.
# harp_tests will inherit any dependencies from gtest_main & harp.
target_link_libraries(harp_tests
    PRIVATE
        gtest_main
        harp_lib
)

# include headers from src
target_include_directories(harp_tests
    PRIVATE
        ${CMAKE_SOURCE_DIR}/src
)

# make ctests aware of our tests
add_test(NAME harp_tests COMMAND harp_tests)
