cmake_minimum_required(VERSION 2.8.3)
set (CMAKE_CXX_STANDARD 11)
project(i3dr_deimos)

# Install udev rules for usb permission using 'install.sh' script
set (udev_install_cmd "./udev/install.sh")
message(STATUS "Running udev install command: ${udev_install_cmd}")
execute_process(COMMAND ${udev_install_cmd}
  WORKING_DIRECTORY ${PROJECT_SOURCE_DIR}
  OUTPUT_VARIABLE udev_install_output)
message(STATUS "udev install response: ${udev_install_output}")

find_package(catkin REQUIRED COMPONENTS 
	nodelet rospy roscpp dynamic_reconfigure message_generation
	camera_info_manager image_transport sensor_msgs stereo_msgs cv_bridge
) 

find_package(Boost REQUIRED COMPONENTS thread system)

find_package(OpenCV REQUIRED)

FIND_PACKAGE(PkgConfig)
PKG_CHECK_MODULES(PC_GLIB QUIET glib-2.0)

FIND_LIBRARY(
	GLIB_LIBRARIES
	NAMES glib-2.0
	HINTS ${PC_GLIB_LIBDIR}
	${PC_GLIB_LIBRARY_DIRS}
)

# Files in glib's main include path may include glibconfig.h, which,
# for some odd reason, is normally in $LIBDIR/glib-2.0/include.
GET_FILENAME_COMPONENT(_GLIB_LIBRARY_DIR ${GLIB_LIBRARIES} PATH)
FIND_PATH(
	GLIBCONFIG_INCLUDE_DIR
	NAMES glibconfig.h
	HINTS ${PC_LIBDIR} ${PC_LIBRARY_DIRS} ${_GLIB_LIBRARY_DIR}
	PATH_SUFFIXES glib-2.0/include
)

FIND_PATH(
	GLIB_INCLUDE_DIR
	NAMES glib.h
	HINTS ${PC_GLIB_INCLUDEDIR}
	${PC_GLIB_INCLUDE_DIRS}
	PATH_SUFFIXES glib-2.0
)

SET(GLIB_INCLUDE_DIRS ${GLIB_INCLUDE_DIR} ${GLIBCONFIG_INCLUDE_DIR})

generate_dynamic_reconfigure_options()

include_directories(
	include
	${catkin_INCLUDE_DIRS}
	${CMAKE_CURRENT_SOURCE_DIR}/src/xunit
	${GLIB_INCLUDE_DIRS}
	${PCL_INCLUDE_DIRS}
)

catkin_package(
	INCLUDE_DIRS include
	CATKIN_DEPENDS roscpp nodelet std_msgs sensor_msgs
	LIBRARIES ${PROJECT_NAME}
#  DEPENDS system_lib
)

add_library(xunit
  src/xunit/xunit_lib.c
  src/xunit/xunit_lib_cug.c
)

add_library(xunitTara
  src/xunit/xunit_lib_tara.h
  src/xunit/xunit_lib_tara.cpp
) 

target_link_libraries(xunitTara ${udev_LIBRARIES} ${GLIB2_LIBRARIES})  
target_link_libraries(xunit ${udev_LIBRARIES})

add_executable(deimos_node src/deimos_node.cpp src/deimos_ros.cpp src/uvc_cam.cpp)
target_link_libraries(deimos_node ${Boost_LIBRARIES} ${catkin_LIBRARIES} xunitTara ${GLIB2_LIBRARIES} -lv4l2 -lyaml-cpp -ludev -lglib-2.0)

#############
## Install ##
#############

install(TARGETS deimos_node
  ARCHIVE DESTINATION ${CATKIN_PACKAGE_LIB_DESTINATION}
  LIBRARY DESTINATION ${CATKIN_PACKAGE_LIB_DESTINATION}
  RUNTIME DESTINATION ${CATKIN_PACKAGE_BIN_DESTINATION}
)
