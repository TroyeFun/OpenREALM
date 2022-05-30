
cmake_minimum_required(VERSION 3.15)

set(LIBRARY_NAME realm_stages)

list(APPEND CMAKE_MODULE_PATH ${CMAKE_CURRENT_SOURCE_DIR}/cmake)

################################################################################
# Dependencies
################################################################################

# Check BoW Framework
set(BOW_FRAMEWORK "FBoW" CACHE STRING "DBoW2 or FBoW") 
set_property(CACHE BOW_FRAMEWORK PROPERTY STRINGS "DBoW2" "FBoW") 
if(BOW_FRAMEWORK MATCHES "DBoW2") 
    find_package(DBoW2 REQUIRED) 
elseif(BOW_FRAMEWORK MATCHES "FBoW") 
    find_package(fbow QUIET) 
    if(NOT fbow_FOUND) 
        add_subdirectory(${PROJECT_SOURCE_DIR}/3rd/FBoW ${PROJECT_BINARY_DIR}/3rd/FBoW) 
    endif() 
else() 
    message(FATAL_ERROR "Invalid BoW framework: ${BOW_FRAMEWORK}") 
endif()

if(BOW_FRAMEWORK MATCHES "DBoW2")
    set(BoW_LIBRARY ${DBoW2_LIBS})
    target_compile_definitions(${PROJECT_NAME} PUBLIC USE_DBOW2)
    message(STATUS "BoW framework: ${BOW_FRAMEWORK} (found in ${DBoW2_INCLUDE_DIRS})")
elseif(BOW_FRAMEWORK MATCHES "FBoW")
    if(fbow_FOUND)
        set(BoW_LIBRARY ${fbow_LIBS})
        message(STATUS "BoW framework: ${BOW_FRAMEWORK} (found in ${fbow_INCLUDE_DIRS})")
    else()
        target_include_directories(${PROJECT_NAME}
                                PUBLIC
                                $<BUILD_INTERFACE:${PROJECT_SOURCE_DIR}/3rd/FBoW/include>
                                $<INSTALL_INTERFACE:include/fbow>)
        set(BoW_LIBRARY fbow)
        message(STATUS "BoW framework: ${BOW_FRAMEWORK} (Using submodule)")
    endif()
else()
    message(FATAL_ERROR "Invalid BoW framework: ${BOW_FRAMEWORK}")
endif()


# Fix to avoid OpenCV package confusion with ROS melodic
find_package(OpenCV 3.3.1 EXACT QUIET)
if (NOT OpenCV_FOUND)
    find_package(OpenCV 3 QUIET)
endif()
if (NOT OpenCV_FOUND)
    find_package(OpenCV 4 REQUIRED)
    message(WARNING "OpenCV 4 Support is experimental, use at your own risk!")
endif()

################################################################################
# Sources
################################################################################


set(root ${CMAKE_CURRENT_SOURCE_DIR})

set(HEADER_FILES
        include/realm_stages/conversions.h
        include/realm_stages/stage_base.h
        include/realm_stages/stage_settings.h
        include/realm_stages/stage_settings_factory.h
)

set(SOURCE_FILES
        src/conversions.cpp
        src/stage_base.cpp
        src/stage_settings_factory.cpp
)

################################################################################
# Options
################################################################################

if(WITH_EXIV2)
    add_compile_definitions(WITH_EXIV2)
endif()

if(WITH_PCL)
    add_compile_definitions(WITH_PCL)
endif()

if(WITH_densifier)
    list(APPEND HEADER_FILES include/realm_stages/densification.h)
    list(APPEND SOURCE_FILES src/densification.cpp)
endif()

if(WITH_ortho)
    list(APPEND HEADER_FILES
            include/realm_stages/ortho_rectification.h
            include/realm_stages/surface_generation.h
            include/realm_stages/mosaicing.h
            include/realm_stages/tileing.h)
    list(APPEND SOURCE_FILES
            src/ortho_rectification.cpp
            src/surface_generation.cpp
            src/mosaicing.cpp
            src/tileing.cpp)
endif()

if(WITH_vslam_base)
    list(APPEND HEADER_FILES include/realm_stages/pose_estimation.h)
    list(APPEND SOURCE_FILES src/pose_estimation.cpp)
endif()

# Organize the source and header files into groups
source_group("Headers" FILES ${HEADER_FILES})
source_group("Source" FILES ${SOURCE_FILES})
source_group("Forms" FILES ${FORM_FILES})
source_group("Resources" FILES ${RESOURCE_FILES})

if(CMAKE_VERSION VERSION_GREATER 3.8)
    source_group(TREE ${root} FILES ${HEADER_FILES} ${SOURCE_FILES})
endif()

# Define the folder containing the header files for this library
set(realm_stages_INCLUDE_DIR ${CMAKE_CURRENT_SOURCE_DIR}/include PARENT_SCOPE)


################################################################################
# Build
################################################################################


include_directories(
        ${CMAKE_CURRENT_BINARY_DIR}
        ${CMAKE_BINARY_DIR}
        ${CMAKE_CURRENT_SOURCE_DIR}/include
        ${OpenCV_INCLUDE_DIRS}

)

add_library(${LIBRARY_NAME}
        ${SOURCE_FILES}
        ${HEADER_FILES}
        ${FORM_HEADERS}
        ${HEADERS_MOC}
        ${RESOURCES_RCC}
)

if(WITH_densifier)
    target_link_libraries(${LIBRARY_NAME} PUBLIC realm_densifier_base)
endif()

if(WITH_ortho)
    target_link_libraries(${LIBRARY_NAME} PUBLIC realm_ortho)
endif()

if(WITH_vslam_base)
    target_link_libraries(${LIBRARY_NAME} PUBLIC realm_vslam_base)
endif()

target_include_directories(${LIBRARY_NAME} PUBLIC
        $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include> # for headers when building
        $<INSTALL_INTERFACE:${OpenREALM_INC_INSTALL_DIR}> # for client in install mode
)

target_link_libraries(${LIBRARY_NAME}
        PUBLIC
            realm_core
            realm_io
        PRIVATE
            ${OpenCV_LIBRARIES}
)


################################################################################
# Install
################################################################################


set_target_properties(${LIBRARY_NAME} PROPERTIES OUTPUT_NAME "open_${LIBRARY_NAME}-${OpenREALM_VERSION}")

install(TARGETS ${LIBRARY_NAME} EXPORT OpenREALMTargets
        RUNTIME       DESTINATION ${OpenREALM_RUNTIME_INSTALL_DIR}
        LIBRARY       DESTINATION ${OpenREALM_LIBRARY_INSTALL_DIR}
        ARCHIVE       DESTINATION ${OpenREALM_ARCHIVE_INSTALL_DIR}
        FRAMEWORK     DESTINATION ${OpenREALM_FRAMEWORK_INSTALL_DIR})

# Headers
install(
        DIRECTORY include/${LIBRARY_NAME}
        DESTINATION ${OpenREALM_INC_INSTALL_DIR}
        FILES_MATCHING
        PATTERN "*.h"
        PATTERN "*.hpp"
)


################################################################################
# Test
################################################################################

