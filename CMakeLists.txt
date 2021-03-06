cmake_minimum_required(VERSION 2.8.3)
project(snap_cam)

# check if this is being built as a ROS package
if (CATKIN_DEVEL_PREFIX)
    set(CATKIN_BUILD 1)
    message(STATUS "Building ${PROJECT_NAME} with catkin")
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -DCATKIN_BUILD")
else()
    set(CATKIN_BUILD 0)
    message(STATUS "Building ${PROJECT_NAME} without catkin")
endif()

set(CMAKE_CXX_FLAGS "-std=c++11 ${CMAKE_CXX_FLAGS} -Wno-format-security -Wno-literal-suffix")

find_package(OpenCV REQUIRED)

if(CATKIN_BUILD)
    ## Find catkin macros and libraries
    ## if COMPONENTS list like find_package(catkin REQUIRED COMPONENTS xyz)
    ## is used, also find other catkin packages
    find_package(catkin REQUIRED COMPONENTS
      cv_bridge
      image_transport
      roscpp
      rospy
      sensor_msgs
      std_msgs
      mavlink
    )

    catkin_package(
      INCLUDE_DIRS include
      LIBRARIES snap_cam
    #  CATKIN_DEPENDS catkin_lib
      DEPENDS yaml-cpp
    )
endif()

add_subdirectory( src/OpticalFlow OpticalFlow )
set( OpticalFlow_LIBS "OpticalFlow" )

include_directories(
	include
	${catkin_INCLUDE_DIRS}
	${OpenCV_INCLUDE_DIRS}
	${OpticalFlow_INCLUDE_DIRS}
	src # for mavlink
	./libcamera/inc
	./libcamera/src
)

find_library(camera_LIBRARIES NAMES camera PATHS /usr/lib)

if (NOT camera_LIBRARIES)
    message(WARNING "Could not find camera libraries, not building targets that depend on it")
endif()

## Declare a C++ library
if (camera_LIBRARIES)
    add_library(snap_cam
      src/SnapCam.cpp
    )
    target_link_libraries(snap_cam
      ${OpenCV_LIBS}
      ${camera_LIBRARIES}
    )

    add_executable(optical_flow src/optical_flow.cpp src/ringbuffer.cpp)
    target_link_libraries(optical_flow yaml-cpp
        ${catkin_LIBRARIES}
        ${OpenCV_LIBS}
        snap_cam
        ${OpticalFlow_LIBS}
    )
endif()

# add the ROS executables
if(CATKIN_BUILD)
    if (camera_LIBRARIES)
        add_executable(snap_cam_publisher_node src/nodes/snap_cam_publisher_node.cpp)
        target_link_libraries(snap_cam_publisher_node yaml-cpp ${catkin_LIBRARIES} ${OpenCV_LIBS} snap_cam)
    endif()
endif()
