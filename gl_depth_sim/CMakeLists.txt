cmake_minimum_required(VERSION 3.5.0)
project(gl_depth_sim VERSION 0.2.0 LANGUAGES C CXX)

SET(CMAKE_CXX_STANDARD 14)
SET(CMAKE_CXX_EXTENSIONS OFF)


# Required for core functionality
find_package(Eigen3 REQUIRED)
find_package(PCL REQUIRED common io)

find_package(OpenCV REQUIRED)
SET(OpenGL_GL_PREFERENCE "GLVND")

#find_package(Threads REQUIRED)
#set(THREADS_PREFER_PTHREAD_FLAG ON)

# Extensions
find_package(assimp REQUIRED) # Just used for loading models in mesh_loader.h
find_package(OpenCV REQUIRED) # Used for interface extension

add_library(glad SHARED src/${PROJECT_NAME}/egl.c src/${PROJECT_NAME}/gl.c)
target_include_directories(glad PRIVATE
    "$<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>"
    "$<INSTALL_INTERFACE:include>")

# Primary rendering library
# Independent of ROS, but does need glfw3 and assimp for model loading
add_library(${PROJECT_NAME} SHARED
  src/${PROJECT_NAME}/sim_depth_camera.cpp
  src/${PROJECT_NAME}/mesh_loader.cpp
  src/${PROJECT_NAME}/mesh.cpp
  src/${PROJECT_NAME}/renderable_mesh.cpp
  src/${PROJECT_NAME}/shader_program.cpp)
target_compile_options(${PROJECT_NAME} PUBLIC -mno-avx)
target_include_directories(${PROJECT_NAME} PUBLIC
    "$<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>"
    "$<INSTALL_INTERFACE:include>")
target_link_libraries(${PROJECT_NAME} PUBLIC
  ${ASSIMP_LIBRARIES}
  glad
  dl
  Eigen3::Eigen
  )

# Libaries for interfacing with opencv and pcl
add_library(${PROJECT_NAME}_interfaces SHARED
  src/interfaces/pcl_interface.cpp)

target_include_directories(${PROJECT_NAME} PUBLIC
    "$<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>"
    "$<INSTALL_INTERFACE:include>")
target_include_directories(${PROJECT_NAME}_interfaces SYSTEM PUBLIC
    ${PCL_INCLUDE_DIRS})
target_link_libraries(${PROJECT_NAME}_interfaces PUBLIC
  ${PROJECT_NAME}
  ${OpenCV_LIBRARIES}
  ${PCL_LIBRARIES}
)

# Simulated laser scanner library
add_library(${PROJECT_NAME}_laser_scanner SHARED
  src/${PROJECT_NAME}/sim_laser_scanner.cpp
)
target_include_directories(${PROJECT_NAME} PUBLIC
    "$<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>"
    "$<INSTALL_INTERFACE:include>"
)
target_link_libraries(${PROJECT_NAME}_laser_scanner PUBLIC
  ${PROJECT_NAME}
  ${PROJECT_NAME}_interfaces
)

install(DIRECTORY include/${PROJECT_NAME}
  DESTINATION include)

install(TARGETS
  glad
  ${PROJECT_NAME}
  ${PROJECT_NAME}_interfaces
  ${PROJECT_NAME}_laser_scanner
        EXPORT ${PROJECT_NAME}-targets
        RUNTIME DESTINATION bin
        LIBRARY DESTINATION lib
        ARCHIVE DESTINATION lib)
install(EXPORT ${PROJECT_NAME}-targets NAMESPACE gl_depth_sim:: DESTINATION lib/cmake/${PROJECT_NAME})

install(FILES package.xml DESTINATION share/${PROJECT_NAME})

# Create cmake config files
include(CMakePackageConfigHelpers)
configure_package_config_file(${CMAKE_CURRENT_LIST_DIR}/cmake/${PROJECT_NAME}-config.cmake.in
  ${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_NAME}-config.cmake
  INSTALL_DESTINATION lib/cmake/${PROJECT_NAME}
  NO_CHECK_REQUIRED_COMPONENTS_MACRO)

write_basic_package_version_file(${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_NAME}-config-version.cmake
  VERSION ${PROJECT_VERSION} COMPATIBILITY ExactVersion)

install(FILES
  "${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_NAME}-config.cmake"
  "${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_NAME}-config-version.cmake"
  DESTINATION lib/cmake/${PROJECT_NAME})

export(EXPORT ${PROJECT_NAME}-targets NAMESPACE gl_depth_sim:: FILE ${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_NAME}-targets.cmake)
