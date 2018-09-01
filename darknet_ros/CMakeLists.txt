cmake_minimum_required(VERSION 2.8.12)
project(darknet_ros)

# Set c++11 cmake flags
set(CMAKE_CXX_FLAGS "-std=c++11 ${CMAKE_CXX_FLAGS}")

# Define path of darknet folder here.
find_path(DARKNET_PATH
  NAMES "README.md"
  HINTS "~/darknet/")
message("Darknet path dir = ${DARKNET_PATH}")
add_definitions(-DDARKNET_FILE_PATH="${DARKNET_PATH}")

# Find CUDA
FIND_PACKAGE(CUDA)
if (CUDA_FOUND)
  find_package(CUDA REQUIRED)
  message(STATUS "CUDA Version: ${CUDA_VERSION_STRINGS}")
  message(STATUS "CUDA Libararies: ${CUDA_LIBRARIES}")
  set(
	  CUDA_NVCC_FLAGS
	  ${CUDA_NVCC_FLAGS};
	  -O3
	  -gencode arch=compute_30,code=sm_30
	  -gencode arch=compute_35,code=sm_35
	  -gencode arch=compute_50,code=[sm_50,compute_50]
	  -gencode arch=compute_52,code=[sm_52,compute_52]
	  -gencode arch=compute_61,code=sm_61
	  -gencode arch=compute_62,code=sm_62
	  )
  add_definitions(-DGPU)
else()
	list(APPEND LIBRARIES "m")
endif()

# Find X11
MESSAGE ( STATUS "Searching for X11..." )
FIND_PACKAGE ( X11 REQUIRED )
IF ( X11_FOUND )
	INCLUDE_DIRECTORIES ( ${X11_INCLUDE_DIR} )
	LINK_LIBRARIES ( ${X11_LIBRARIES} )
	MESSAGE ( STATUS " X11_INCLUDE_DIR: " ${X11_INCLUDE_DIR} )
	MESSAGE ( STATUS " X11_LIBRARIES: " ${X11_LIBRARIES} )
ENDIF ( X11_FOUND )

# Find rquired packeges
find_package(Boost REQUIRED COMPONENTS thread)
find_package(OpenCV REQUIRED)
include_directories(${OpenCV_INCLUDE_DIRS})
find_package(catkin REQUIRED
	COMPONENTS
	cv_bridge
	roscpp
	rospy
	std_msgs
	actionlib
	darknet_ros_msgs
	image_transport
	)

# Enable OPENCV in darknet
add_definitions(-DOPENCV)
add_definitions(-O4 -g)

catkin_package(
	INCLUDE_DIRS
	include
	LIBRARIES
	${PROJECT_NAME}_lib
	CATKIN_DEPENDS
	cv_bridge
	roscpp
	actionlib
	rospy
	std_msgs
	darknet_ros_msgs
	image_transport
	DEPENDS
	Boost
	)

include_directories(
	${DARKNET_PATH}/src
	${DARKNET_PATH}/include
	include
	${Boost_INCLUDE_DIRS}
	${catkin_INCLUDE_DIRS}
	)

if(DARKNET_PATH)               # <--- Use 'BOOST_DIR', not 'DEFINED ${BOOST_DIR}'

	include_directories(
		include
		${catkin_INCLUDE_DIRS}
		${CUDA_INCLUDE_DIRS}
		${Eigen_INCLUDE_DIRS}
		${DARKNET_PATH}/include
		${DARKNET_PATH}/src
		${OpenCV_LIBS}
		)

	add_library(${PROJECT_NAME}_lib
		src/YoloObjectDetector.cpp
		src/image_interface.c
	)
	target_link_libraries(${PROJECT_NAME}_lib
		${catkin_LIBRARIES} 
		${Eigen_LIBRARIES}
		${OpenCV_LIBRARIES}
		${DARKNET_PATH}/libdarknet.so
		)

	#add_executable(open_pose_node src/open_pose_node.cpp)
	#target_link_libraries(open_pose_node ${catkin_LIBRARIES} OpenPose)
	add_executable(${PROJECT_NAME} src/yolo_object_detector_node.cpp)
	target_link_libraries(${PROJECT_NAME}
		${catkin_LIBRARIES}
		${PROJECT_NAME}_lib
		)
else()
	message(WARNING "Undefined DARKNET_HOME env var, this package will not be compiled.")
endif()

add_dependencies(${PROJECT_NAME}_lib
	darknet_ros_msgs_generate_messages_cpp
	)
