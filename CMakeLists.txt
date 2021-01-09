###I.Global####################################################################
cmake_minimum_required(VERSION 3.5)
project(ros2cart)
set(CMAKE_CXX_STANDARD 14)
set(CMAKE_CXX_STANDARD_REQUIRED true)
set(LIBRARY_OUTPUT_PATH ${CMAKE_CURRENT_SOURCE_DIR}/../out)
set(EXECUTABLE_OUTPUT_PATH ${CMAKE_CURRENT_SOURCE_DIR}/../out)
add_compile_options(-Wall -Wextra)

#1.
find_package(cartographer REQUIRED)
include("${CARTOGRAPHER_CMAKE_DIR}/functions.cmake")
google_initialize_cartographer_project()
include(FindPkgConfig)
find_package(LuaGoogle REQUIRED)
find_package(PCL REQUIRED COMPONENTS common)
find_package(urdfdom_headers REQUIRED)

if(DEFINED urdfdom_headers_VERSION)
  if(${urdfdom_headers_VERSION} GREATER 0.4.1)
    add_definitions(-DURDFDOM_HEADERS_HAS_SHARED_PTR_DEFS)
  endif()
endif()

#2.Includes&Libraries
include_directories(${CMAKE_CURRENT_SOURCE_DIR}
    /usr/include
    /usr/include/eigen3
    /usr/include/opencv4
    /opt/ros/foxy/include
    /root/app/ros2ex/out/install/include
    ${LUA_INCLUDE_DIR}
    ${PCL_INCLUDE_DIRS}
    ${urdfdom_headers_INCLUDE_DIRS}
    $<BUILD_INTERFACE:${PROJECT_BINARY_DIR}> 
    $<BUILD_INTERFACE:${PROJECT_SOURCE_DIR}> 
    $<INSTALL_INTERFACE:include>)
file(GLOB OpenCV_LIBS /usr/lib/x86_64-linux-gnu/libopencv*.so)
file(GLOB ROS2_LIBS LIST_DIRECTORIES false /opt/ros/foxy/lib/*.so)
file(GLOB ROS2EX_LIBS LIST_DIRECTORIES false /root/app/ros2ex/out/install/lib/*.so)
link_libraries(${OpenCV_LIBS} ${ROS2_LIBS} ${ROS2EX_LIBS} ${PCL_LIBRARIES})

#3.ROS2CartLib
set(CART_SRCS
  ${CMAKE_CURRENT_SOURCE_DIR}/cartographer_ros/map_builder_bridge.cc
  ${CMAKE_CURRENT_SOURCE_DIR}/cartographer_ros/msg_conversion.cc
  ${CMAKE_CURRENT_SOURCE_DIR}/cartographer_ros/node.cc
  ${CMAKE_CURRENT_SOURCE_DIR}/cartographer_ros/node_constants.cc
  ${CMAKE_CURRENT_SOURCE_DIR}/cartographer_ros/node_options.cc
  ${CMAKE_CURRENT_SOURCE_DIR}/cartographer_ros/ros_log_sink.cc
  ${CMAKE_CURRENT_SOURCE_DIR}/cartographer_ros/sensor_bridge.cc
  ${CMAKE_CURRENT_SOURCE_DIR}/cartographer_ros/tf_bridge.cc
  ${CMAKE_CURRENT_SOURCE_DIR}/cartographer_ros/time_conversion.cc
  ${CMAKE_CURRENT_SOURCE_DIR}/cartographer_ros/trajectory_options.cc
  ${CMAKE_CURRENT_SOURCE_DIR}/cartographer_ros/submap.cc)
add_library(${PROJECT_NAME} STATIC ${CART_SRCS})
target_link_libraries(${PROJECT_NAME} cartographer)#???

#4.ROS2CartMap
add_executable(cartographer_node ${CMAKE_CURRENT_SOURCE_DIR}/cartographer_ros/node_main.cc)
target_link_libraries(cartographer_node ${PROJECT_NAME})

#5.ROS2CartGrid
add_executable(occupancy_grid_node ${CMAKE_CURRENT_SOURCE_DIR}/cartographer_ros/occupancy_grid_node_main.cc)
target_link_libraries(occupancy_grid_node ${PROJECT_NAME})


# PCL
set(BLACKLISTED_PCL_DEFINITIONS " -march=native -msse4.2 -mfpmath=sse ")
foreach(DEFINITION ${PCL_DEFINITIONS})
  list (FIND BLACKLISTED_PCL_DEFINITIONS "${DEFINITION}" DEFINITIONS_INDEX)
  if (${DEFINITIONS_INDEX} GREATER -1)
    continue()
  endif()
  set(TARGET_COMPILE_FLAGS "${TARGET_COMPILE_FLAGS} ${DEFINITION}")
endforeach()


set(TARGET_COMPILE_FLAGS "${TARGET_COMPILE_FLAGS} ${GOOG_CXX_FLAGS}")
set_target_properties(${PROJECT_NAME} PROPERTIES COMPILE_FLAGS ${TARGET_COMPILE_FLAGS})


install(DIRECTORY configuration_files DESTINATION share/${PROJECT_NAME}/)
install(TARGETS ${PROJECT_NAME} ARCHIVE DESTINATION lib)
install(TARGETS cartographer_node occupancy_grid_node DESTINATION lib/${PROJECT_NAME})