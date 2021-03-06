set (VSTPLUGIN_PRJ_NAME "${PROJECT_NAME}_vst3")
set (VSTPLUGIN_BUNDLE_NAME "${PROJECT_NAME}.vst3")

set (VST3SDK_BASEDIR "${CMAKE_CURRENT_SOURCE_DIR}/external/VST_SDK/VST3_SDK")

# VST plugin specific settings
include (VSTConfig)

configure_file (VstPluginDefs.h.in "${CMAKE_CURRENT_BINARY_DIR}/VstPluginDefs.h")

# Build VST3 SDK
include("cmake/Vst3.cmake")

# Build the plugin
add_library(${VSTPLUGIN_PRJ_NAME} MODULE
    SfizzVstProcessor.cpp
    SfizzVstController.cpp
    SfizzVstEditor.cpp
    SfizzVstState.cpp
    GUIComponents.cpp
    VstPluginFactory.cpp
    X11RunLoop.cpp)

if(WIN32)
    target_sources(${VSTPLUGIN_PRJ_NAME} PRIVATE vst3.def)
endif()
target_link_libraries(${VSTPLUGIN_PRJ_NAME}
    PRIVATE ${PROJECT_NAME}::${PROJECT_NAME})
target_include_directories(${VSTPLUGIN_PRJ_NAME}
    PRIVATE "${CMAKE_CURRENT_BINARY_DIR}")
set_target_properties(${VSTPLUGIN_PRJ_NAME} PROPERTIES
    OUTPUT_NAME "${PROJECT_NAME}"
    PREFIX "")

plugin_add_vst3sdk(${VSTPLUGIN_PRJ_NAME})
plugin_add_vstgui(${VSTPLUGIN_PRJ_NAME})

# Add the ring buffer
target_include_directories(${VSTPLUGIN_PRJ_NAME} PRIVATE "external/ring_buffer")
target_sources(${VSTPLUGIN_PRJ_NAME} PRIVATE "external/ring_buffer/ring_buffer/ring_buffer.cpp")

if (${CMAKE_SYSTEM_NAME} MATCHES "Linux")
    target_link_libraries(${VSTPLUGIN_PRJ_NAME} PRIVATE
        "-Wl,--version-script=${CMAKE_CURRENT_SOURCE_DIR}/vst3.version")
endif()
sfizz_enable_lto_if_needed (${VSTPLUGIN_PRJ_NAME})
if (MINGW)
    set_target_properties (${VSTPLUGIN_PRJ_NAME} PROPERTIES LINK_FLAGS "-static")
endif()

# Create the bundle (see "VST 3 Locations / Format")
execute_process (
    COMMAND "${CMAKE_COMMAND}" -E make_directory "${PROJECT_BINARY_DIR}/${VSTPLUGIN_BUNDLE_NAME}/Contents/Resources")
file (COPY "${CMAKE_CURRENT_SOURCE_DIR}/resources/logo.png"
    DESTINATION "${PROJECT_BINARY_DIR}/${VSTPLUGIN_BUNDLE_NAME}/Contents/Resources")
if(WIN32)
    set_target_properties(${VSTPLUGIN_PRJ_NAME} PROPERTIES
        SUFFIX ".vst3"
        LIBRARY_OUTPUT_DIRECTORY "${PROJECT_BINARY_DIR}/${VSTPLUGIN_BUNDLE_NAME}/Contents/${VST3_PACKAGE_ARCHITECTURE}-win")
    foreach(config ${CMAKE_CONFIGURATION_TYPES})
        string(TOUPPER "${config}" config)
        set_target_properties(${VSTPLUGIN_PRJ_NAME} PROPERTIES
            "LIBRARY_OUTPUT_DIRECTORY_${config}" "${PROJECT_BINARY_DIR}/${VSTPLUGIN_BUNDLE_NAME}/Contents/${VST3_PACKAGE_ARCHITECTURE}-win")
    endforeach()
    file(COPY "${CMAKE_CURRENT_SOURCE_DIR}/win/Plugin.ico"
        "${CMAKE_CURRENT_SOURCE_DIR}/win/desktop.ini"
        DESTINATION "${PROJECT_BINARY_DIR}/${VSTPLUGIN_BUNDLE_NAME}")
elseif(APPLE)
    set_target_properties(${VSTPLUGIN_PRJ_NAME} PROPERTIES
        SUFFIX ""
        LIBRARY_OUTPUT_DIRECTORY "${PROJECT_BINARY_DIR}/${VSTPLUGIN_BUNDLE_NAME}/Contents/MacOS")
    file(COPY "${CMAKE_CURRENT_SOURCE_DIR}/mac/PkgInfo"
        DESTINATION "${PROJECT_BINARY_DIR}/${VSTPLUGIN_BUNDLE_NAME}/Contents")
    set(SFIZZ_VST3_BUNDLE_EXECUTABLE "${PROJECT_NAME}")
    set(SFIZZ_VST3_BUNDLE_VERSION "${PROJECT_VERSION}")
    configure_file("${CMAKE_CURRENT_SOURCE_DIR}/mac/Info.plist"
        "${PROJECT_BINARY_DIR}/${VSTPLUGIN_BUNDLE_NAME}/Contents/Info.plist" @ONLY)
    file(COPY "${CMAKE_CURRENT_SOURCE_DIR}/mac/Plugin.icns"
        DESTINATION "${PROJECT_BINARY_DIR}/${VSTPLUGIN_BUNDLE_NAME}/Contents/Resources")
else()
    set_target_properties(${VSTPLUGIN_PRJ_NAME} PROPERTIES
        LIBRARY_OUTPUT_DIRECTORY "${PROJECT_BINARY_DIR}/${VSTPLUGIN_BUNDLE_NAME}/Contents/${VST3_PACKAGE_ARCHITECTURE}-linux")
endif()

file(COPY "gpl-3.0.txt"
    DESTINATION "${PROJECT_BINARY_DIR}/${VSTPLUGIN_BUNDLE_NAME}")

if (CMAKE_CXX_COMPILER_ID MATCHES "GNU|Clang")
    target_compile_options(${VSTPLUGIN_PRJ_NAME} PRIVATE
        "-Wno-extra"
        "-Wno-multichar"
        "-Wno-reorder"
        "-Wno-class-memaccess"
        "-Wno-ignored-qualifiers"
        "-Wno-unknown-pragmas"
        "-Wno-unused-function"
        "-Wno-unused-parameter"
        "-Wno-unused-variable")
endif()

# To help debugging the link only
if (FALSE)
    target_link_options(${VSTPLUGIN_PRJ_NAME} PRIVATE "-Wl,-no-undefined")
endif()

# Installation
if (NOT MSVC)
    install (DIRECTORY "${PROJECT_BINARY_DIR}/${VSTPLUGIN_BUNDLE_NAME}"
        DESTINATION "${VSTPLUGIN_INSTALL_DIR}")
endif()
