cmake_minimum_required(VERSION 3.16)

# set the output directory for built objects.
# This makes sure that the dynamic library goes into the build directory automatically.
set(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/$<CONFIGURATION>)
set(CMAKE_LIBRARY_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/$<CONFIGURATION>)

# prevent installing to system directories. 
set(CMAKE_INSTALL_PREFIX ${CMAKE_BINARY_DIR} CACHE INTERNAL "")

# Declare the project
project(SDL3DEngine)

# Set the name of the executable
set(EXECUTABLE_NAME ${PROJECT_NAME})

# Create an executable or a shared library based on the platform and add our sources to it
if (ANDROID)
	# The SDL java code is hardcoded to load libmain.so on android, so we need to change EXECUTABLE_NAME
	set(EXECUTABLE_NAME main)
	add_library(${EXECUTABLE_NAME} SHARED)
else()
	add_executable(${EXECUTABLE_NAME})
endif()

target_sources(${EXECUTABLE_NAME} 
PRIVATE 
    src/main.cpp
    src/iosLaunchScreen.storyboard
    src/Sample.appxManifest
)

# What is iosLaunchScreen.storyboard? This file describes what Apple's mobile platforms
# should show the user while the application is starting up. If you don't include one,
# then you get placed in a compatibility mode that does not allow HighDPI.
# This file is referenced inside Info.plist.in, where it is marked as the launch screen file.
# It is also ignored on non-Apple platforms. 

# What is Sample.appxManifest? This file describes deployment settings for Universal Windows Platform
# applications, like splash screen and icon resolutions, which are required for the application to run. 
# It is ignored on other platforms. Below, we add those images. 
if (WINDOWS_STORE)
    target_sources("${EXECUTABLE_NAME}" PRIVATE "src/logo44.png" "src/logo150.png" "src/logo50.png" "src/logo620x300.png")
elseif(APPLE)
    target_sources("${EXECUTABLE_NAME}" PRIVATE "src/logo.png")
endif()

# use C++11
target_compile_features(${EXECUTABLE_NAME} PUBLIC cxx_std_11)

if(CMAKE_SYSTEM_NAME MATCHES Emscripten)
	set(CMAKE_EXECUTABLE_SUFFIX ".html" CACHE INTERNAL "")
endif()

# If targeting Windows UWP, enable Windows Runtime Compilation
if(WINDOWS_STORE)
    target_compile_options(${EXECUTABLE_NAME} PUBLIC "/ZW" "/EHsc")
endif()




# -------------- CMakeLists.txt for SDL submodule
#
# Configure SDL by calling its CMake file.
# we use EXCLUDE_FROM_ALL so that its install targets and configs don't
# pollute upwards into our configuration.
add_subdirectory(SDL EXCLUDE_FROM_ALL)

# Link SDL to our executable. This also makes its include directory available to us. 
target_link_libraries(${EXECUTABLE_NAME} PUBLIC SDL3::SDL3)
# ----------------------------------------------------------------------




# -------------- CMakeLists.txt for ImGui submodule
# 
# Gather source files for ImGui library
set(IMGUI_SOURCE_FILES
    imgui/imgui.h
    imgui/imgui.cpp
    imgui/imconfig.h
    imgui/imgui_demo.cpp
    imgui/imgui_draw.cpp
    imgui/imgui_internal.h
    imgui/imstb_rectpack.h
    imgui/imstb_textedit.h
    imgui/imstb_truetype.h
    imgui/imgui_tables.cpp
    imgui/imgui_widgets.cpp
	
	# Add other ImGui source files if needed
    
	# Specific bindings for SDL
    imgui/backends/imgui_impl_sdl3.h
    imgui/backends/imgui_impl_sdl3.cpp
    imgui/backends/imgui_impl_sdlrenderer3.h
    imgui/backends/imgui_impl_sdlrenderer3.cpp
    imgui/backends/imgui_impl_opengl3.h
    imgui/backends/imgui_impl_opengl3.cpp
)

# Define the ImGui library target
add_library("imgui" STATIC ${IMGUI_SOURCE_FILES})

# Include directories for ImGui headers and the 'backends' directory
target_include_directories("imgui" PUBLIC ${CMAKE_CURRENT_SOURCE_DIR}/imgui)
target_include_directories("imgui" PUBLIC ${CMAKE_CURRENT_SOURCE_DIR}/imgui/backends)

# Link SDL to the ImGui library target
target_link_libraries("imgui" PRIVATE SDL3::SDL3)

# Set C++ standard for ImGui
target_compile_features(imgui PUBLIC cxx_std_11)

# Link ImGui to your executable or library
target_link_libraries(${EXECUTABLE_NAME} PUBLIC imgui)
# ----------------------------------------------------------------------




# set some extra configs for each platform
set_target_properties(${EXECUTABLE_NAME} PROPERTIES 
    # On macOS, make a proper .app bundle instead of a bare executable
    MACOSX_BUNDLE TRUE
    # Set the Info.plist file for Apple Mobile platforms. Without this file, your app
    # will not launch. 
    MACOSX_BUNDLE_INFO_PLIST "${CMAKE_CURRENT_SOURCE_DIR}/src/Info.plist.in"

    # in Xcode, create a Scheme in the schemes dropdown for the app.
    XCODE_GENERATE_SCHEME TRUE
    # Identification for Xcode
    XCODE_ATTRIBUTE_BUNDLE_IDENTIFIER "com.SDL3DEngine"
	XCODE_ATTRIBUTE_PRODUCT_BUNDLE_IDENTIFIER "com.SDL3DEngine"
	XCODE_ATTRIBUTE_CURRENTYEAR "${CURRENTYEAR}"
)

# on Visual Studio, set our app as the default project
set_property(DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR} PROPERTY VS_STARTUP_PROJECT "${EXECUTABLE_NAME}")

# On Apple Platforms, ensure that 
# the bundle is valid for distribution.
if(CMAKE_SYSTEM_NAME STREQUAL "Darwin")
INSTALL(CODE 
    "include(BundleUtilities)
    fixup_bundle(\"${CMAKE_INSTALL_PREFIX}/$<CONFIGURATION>/${EXECUTABLE_NAME}.app\" \"\" \"\")
    " 
    COMPONENT Runtime
)
endif()
