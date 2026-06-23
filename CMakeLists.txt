# The first line of any CMake project should be a call to `cmake_minimum_required`, which checks
# that the installed CMake will be able to understand the following CMakeLists, and ensures that
# CMake's behaviour is compatible with the named version. This is a standard CMake command, so more
# information can be found in the CMake docs.
cmake_minimum_required(VERSION 3.21)
set(PROJECT_NAME "HARP")

set(CMAKE_CXX_STANDARD 20)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

file(STRINGS VERSION CURRENT_VERSION)

project(${PROJECT_NAME} VERSION ${CURRENT_VERSION})

enable_testing()
option(HARP_ENABLE_TESTS "Build HARP unit tests" ON)

# JUCE settings
set(JUCE_SILENCE_XCODE_DEPRECATION_WARNINGS ON)

add_subdirectory(JUCE)

# Reset warning levels
if(CMAKE_CXX_COMPILER_ID MATCHES "Clang|AppleClang|GNU")
    add_compile_options(-Wall -Wextra)
elseif(CMAKE_CXX_COMPILER_ID MATCHES "MSVC")
    add_compile_options(/W4)
endif()

juce_add_gui_app(${PROJECT_NAME}
    ICON_BIG "${CMAKE_SOURCE_DIR}/resources/icons/harp_logo_1.png"
    ICON_SMALL "${CMAKE_SOURCE_DIR}/resources/icons/harp_logo_1.png"
    DOCUMENT_EXTENSIONS wav mp3 aiff
    COMPANY_NAME "TEAMuP"
    PRODUCT_NAME "HARP"
)

juce_generate_juce_header(${PROJECT_NAME})

# Collect all HARP sources into a library so tests can link against them
set(HARP_SOURCES
    src/Main.cpp
    src/Application.cpp
    src/MainComponent.cpp
    src/ModelTab.h
    src/Model.h
    src/windows/AboutWindow.h
    src/windows/WelcomeWindow.h
    src/windows/settings/SettingsWindow.h
    src/windows/settings/GeneralSettingsTab.cpp
    src/windows/settings/LoginTab.cpp
    src/widgets/ModelSelectionWidget.h
    src/widgets/ModelInfoWidget.h
    src/widgets/ControlAreaWidget.h
    src/widgets/TrackAreaWidget.h
    src/widgets/StatusAreaWidget.h
    src/widgets/MediaClipboardWidget.h
    src/gui/MultiButton.cpp
    src/gui/HoverableLabel.h
    src/gui/TextBoxWithLabel.h
    src/gui/SliderWithLabel.h
    src/gui/ComboBoxWithLabel.h
    src/gui/HoverHandler.h
    src/gui/ControlComponent.h
    src/gui/ToggleWithLabel.h
    src/media/MediaDisplayComponent.cpp
    src/media/AudioDisplayComponent.cpp
    src/media/MidiDisplayComponent.cpp
    src/media/OutputLabelComponent.cpp
    src/media/pianoroll/KeyboardComponent.cpp
    src/media/pianoroll/NoteGridComponent.cpp
    src/media/pianoroll/PianoRollComponent.cpp
    src/media/pianoroll/SynthAudioSource.h
    src/clients/Client.h
    src/clients/GradioClient.h
    src/utils/Logging.h
    src/utils/Settings.h
    src/utils/Interface.h
    src/utils/Controls.h
    src/utils/Labels.h
    src/utils/Clients.h
    src/utils/Errors.h
    src/utils/Enums.h
    src/external/magic_enum.hpp
    src/external/fontawesome/src/FontAwesome.h
    src/external/fontawesome/src/FontAwesome.cpp
    src/external/fontawesome/data/FontAwesomeData.h
    src/external/fontawesome/data/FontAwesomeData.cpp
    src/external/fontawesome/data/FontAwesomeIcons.h
    src/external/fontaudio/src/FontAudio.h
    src/external/fontaudio/src/FontAudio.cpp
    src/external/fontaudio/data/FontAudioData.h
    src/external/fontaudio/data/FontAudioData.cpp
    src/external/fontaudio/data/FontAudioIcons.h
)

add_library(harp_lib STATIC ${HARP_SOURCES})

target_link_libraries(harp_lib
    PRIVATE
        juce::juce_gui_extra
        juce::juce_audio_basics
        juce::juce_audio_devices
        juce::juce_audio_formats
        juce::juce_audio_processors
        juce::juce_audio_utils
        juce::juce_core
        juce::juce_data_structures
        juce::juce_dsp
        juce::juce_events
        juce::juce_gui_basics
        stability_controls
)

# JUCE include paths needed for JuceHeader.h and module headers
target_include_directories(harp_lib
    PRIVATE
        ${CMAKE_BINARY_DIR}/JuceLibraryCode
        ${CMAKE_SOURCE_DIR}/JUCE/modules
)

# Ensure JUCE global module settings are considered included
target_compile_definitions(harp_lib
    PRIVATE
        JUCE_GLOBAL_MODULE_SETTINGS_INCLUDED=1
)

# Link GUI app to harp_lib
target_link_libraries(${PROJECT_NAME}
    PRIVATE
        harp_lib
    PUBLIC
        juce::juce_recommended_config_flags
        juce::juce_recommended_lto_flags
        juce::juce_recommended_warning_flags
)

target_compile_definitions(${PROJECT_NAME}
    PRIVATE
        JUCE_WEB_BROWSER=0
        JUCE_USE_CURL=1
        JUCE_LOAD_CURL_SYMBOLS_LAZILY=1
        JUCE_APPLICATION_NAME_STRING="$<TARGET_PROPERTY:${PROJECT_NAME},JUCE_PRODUCT_NAME>"
        JUCE_APPLICATION_VERSION_STRING="$<TARGET_PROPERTY:${PROJECT_NAME},JUCE_VERSION>"
        JUCE_USE_FLAC=1
        JUCE_USE_OGGVORBIS=1
        JUCE_USE_MP3AUDIOFORMAT=1
        JUCE_USE_WINDOWS_MEDIA_FORMAT=1
    PUBLIC
        APP_VERSION="${CURRENT_VERSION}"
        APP_COMPANY="TEAMuP"
        APP_NAME="HARP"
        APP_COPYRIGHT="Copyright 2026 TEAMuP. All rights reserved."
)

juce_add_binary_data(stability_controls
    SOURCES
        ${CMAKE_SOURCE_DIR}/src/clients/providers/stability/models/text-to-audio.json
        ${CMAKE_SOURCE_DIR}/src/clients/providers/stability/models/audio-to-audio.json
)

# Windows DLL copying logic — preserved for local dev, disabled on CI
if (WIN32 AND NOT DEFINED ENV{GITHUB_ACTIONS})
    function(find_vc_runtime_dlls out_var)
        message(STATUS "Searching for specific runtime DLLs...")

        set(target_dlls
            "msvcp140.dll"
            "vcruntime140_1.dll"
            "vcruntime140.dll"
        )

        set(VS_CRT_BASE_PATH "C:/Program Files/Microsoft Visual Studio/*/*/VC/Redist/MSVC/*/x64/*")

        file(GLOB_RECURSE all_runtime_dlls
            LIST_DIRECTORIES false
            "${VS_CRT_BASE_PATH}/*.dll"
        )

        set(runtime_dlls)

        foreach(dll ${all_runtime_dlls})
            get_filename_component(dll_name ${dll} NAME)
            if(dll_name IN_LIST target_dlls)
                list(APPEND runtime_dlls ${dll})
            endif()
        endforeach()

        if(runtime_dlls)
            message(STATUS "Found specific runtime DLLs:")
            foreach(dll ${runtime_dlls})
                message(STATUS "${dll}")
            endforeach()
            set(${out_var} ${runtime_dlls} PARENT_SCOPE)
        else()
            message(WARNING "Required specific runtime DLLs not found — skipping copy")
        endif()
    endfunction()

    find_vc_runtime_dlls(RUNTIME_DLLS)

    add_custom_command(TARGET ${PROJECT_NAME} POST_BUILD
        COMMAND ${CMAKE_COMMAND} -E copy_if_different
        ${RUNTIME_DLLS}
        $<TARGET_FILE_DIR:${PROJECT_NAME}>)
endif()

# macOS RPATH fix
if (APPLE)
    set_property(TARGET ${PROJECT_NAME} PROPERTY BUILD_RPATH "@loader_path/../Frameworks")
    set(CMAKE_SKIP_RPATH "NO" CACHE INTERNAL "")
endif()

# Tests
if(HARP_ENABLE_TESTS)
    add_subdirectory(tests)
endif()
