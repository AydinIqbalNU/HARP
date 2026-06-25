include(FetchContent)

FetchContent_Declare(
    googletest
    URL https://github.com/google/googletest/archive/refs/tags/v1.14.0.zip
)

FetchContent_MakeAvailable(googletest)

add_executable(harp_tests
    test_main.cpp
    test_smoke.cpp
)

target_link_libraries(harp_tests PRIVATE
    gtest_main
)

add_test(NAME harp_tests COMMAND harp_tests)
