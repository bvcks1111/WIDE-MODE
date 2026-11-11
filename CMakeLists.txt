cmake_minimum_required(VERSION 3.22)

# Static MSVC runtime: the plugin runs on any Windows PC without the VC++ redistributable.
set(CMAKE_MSVC_RUNTIME_LIBRARY "MultiThreaded$<$<CONFIG:Debug>:Debug>" CACHE STRING "" FORCE)

project(WideMode VERSION 1.0.0 LANGUAGES C CXX)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

option(WIDEMODE_BUILD_TESTS "Build the standalone DSP unit tests" ON)
option(WIDEMODE_BUILD_PLUGIN "Build the JUCE plugin (needs internet on first configure)" ON)

# ---------------------------------------------------------------- DSP unit tests (no JUCE)
if(WIDEMODE_BUILD_TESTS)
    enable_testing()
    add_executable(wide_mode_dsp_tests Tests/DspTests.cpp)
    target_compile_features(wide_mode_dsp_tests PRIVATE cxx_std_17)
    add_test(NAME wide_mode_dsp COMMAND wide_mode_dsp_tests)
endif()

# ---------------------------------------------------------------- Plugin
if(WIDEMODE_BUILD_PLUGIN)
    include(FetchContent)
    FetchContent_Declare(JUCE
        GIT_REPOSITORY https://github.com/juce-framework/JUCE.git
        GIT_TAG        8.0.6
        GIT_SHALLOW    ON)
    FetchContent_MakeAvailable(JUCE)

    juce_add_plugin(WideMode
        COMPANY_NAME                "WideMode Audio"
        PRODUCT_NAME                "WIDE MODE"
        PLUGIN_MANUFACTURER_CODE    WdMd
        PLUGIN_CODE                 Wide
        FORMATS                     VST3 Standalone
        VST3_CATEGORIES             Fx Spatial
        IS_SYNTH                    FALSE
        NEEDS_MIDI_INPUT            FALSE
        NEEDS_MIDI_OUTPUT           FALSE
        IS_MIDI_EFFECT              FALSE
        COPY_PLUGIN_AFTER_BUILD     FALSE)

    target_sources(WideMode PRIVATE
        Source/PluginProcessor.cpp
        Source/PluginEditor.cpp
        Source/PresetManager.cpp)

    target_compile_definitions(WideMode PUBLIC
        JUCE_WEB_BROWSER=0
        JUCE_USE_CURL=0
        JUCE_VST3_CAN_REPLACE_VST2=0)

    target_link_libraries(WideMode
        PRIVATE
            juce::juce_audio_utils
        PUBLIC
            juce::juce_recommended_config_flags
            juce::juce_recommended_lto_flags
            juce::juce_recommended_warning_flags)
endif()
