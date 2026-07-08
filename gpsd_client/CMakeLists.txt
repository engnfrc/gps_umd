cmake_minimum_required(VERSION 3.10)
project(gpsd_client)

if (NOT CMAKE_CXX_STANDARD)
  set(CMAKE_CXX_STANDARD 17)
endif()

find_package(ament_cmake REQUIRED)

find_package(gps_msgs REQUIRED)
find_package(rclcpp REQUIRED)
find_package(rclcpp_components REQUIRED rclcpp_components)
find_package(sensor_msgs REQUIRED)

# Expose sensor_msgs package version to C++ via compile definitions.
set(SENSOR_MSGS_VERSION_MAJOR 0)
set(SENSOR_MSGS_VERSION_MINOR 0)
set(SENSOR_MSGS_VERSION_PATCH 0)

if(DEFINED sensor_msgs_VERSION)
  string(REPLACE "." ";" _sensor_msgs_version_list "${sensor_msgs_VERSION}")
  list(LENGTH _sensor_msgs_version_list _sensor_msgs_version_len)
  if(_sensor_msgs_version_len GREATER 0)
    list(GET _sensor_msgs_version_list 0 SENSOR_MSGS_VERSION_MAJOR)
  endif()
  if(_sensor_msgs_version_len GREATER 1)
    list(GET _sensor_msgs_version_list 1 SENSOR_MSGS_VERSION_MINOR)
  endif()
  if(_sensor_msgs_version_len GREATER 2)
    list(GET _sensor_msgs_version_list 2 SENSOR_MSGS_VERSION_PATCH)
  endif()
endif()

math(EXPR SENSOR_MSGS_VERSION_NUM
  "${SENSOR_MSGS_VERSION_MAJOR} * 10000 + ${SENSOR_MSGS_VERSION_MINOR} * 100 + ${SENSOR_MSGS_VERSION_PATCH}")

# Try to find libgps, first with CMake's usual library search method, then by
# querying pkg-config.
find_library(libgps_LIBRARIES NAMES gps)
find_path(libgps_INCLUDE_DIRS NAMES libgpsmm.h gps.h)

if(NOT libgps_LIBRARIES)
  message(STATUS "Checking pkg-config for libgps")
  find_package(PkgConfig)
  if(PkgConfig_FOUND)
    pkg_check_modules(libgps libgps)
  endif()
endif()

if(NOT libgps_LIBRARIES)
  message(FATAL_ERROR "Could not find libgps "
                      "(hint for Debian/Ubuntu: apt install libgps-dev)")
else()
  message(STATUS "Found libgps: ${libgps_LIBRARIES}")
endif()

add_library(${PROJECT_NAME} SHARED
  src/client.cpp)

target_include_directories(${PROJECT_NAME}
  PUBLIC
    ${rclcpp_components_INCLUDE_DIRS}
    ${libgps_INCLUDE_DIRS}
    ${gps_msgs_INCLUDE_DIRS}
    ${sensor_msgs_INCLUDE_DIRS})

if(sensor_msgs_VERSION VERSION_LESS 5.0.0)
  target_compile_definitions(${PROJECT_NAME} PRIVATE "NO_UNKNOWN_SERVICE_PROVIDER")
endif()

target_compile_definitions(${PROJECT_NAME}
  PRIVATE
    "COMPOSITION_BUILDING_DLL"
    "SENSOR_MSGS_VERSION_MAJOR=${SENSOR_MSGS_VERSION_MAJOR}"
    "SENSOR_MSGS_VERSION_MINOR=${SENSOR_MSGS_VERSION_MINOR}"
    "SENSOR_MSGS_VERSION_PATCH=${SENSOR_MSGS_VERSION_PATCH}"
    "SENSOR_MSGS_VERSION_NUM=${SENSOR_MSGS_VERSION_NUM}")

rclcpp_components_register_nodes(${PROJECT_NAME} "gpsd_client::GPSDClientComponent")
target_link_libraries(${PROJECT_NAME} PUBLIC
  ${libgps_LIBRARIES}
  ${gps_msgs_TARGETS}
  ${sensor_msgs_TARGETS}
  rclcpp::rclcpp
  rclcpp_components::component
  rclcpp_components::component_manager
  sensor_msgs::sensor_msgs_library
)

#############
## Install ##
#############

install(TARGETS ${PROJECT_NAME}
  ARCHIVE DESTINATION lib
  LIBRARY DESTINATION lib
  RUNTIME DESTINATION bin
)

install(DIRECTORY launch
    DESTINATION share/${PROJECT_NAME}/)

install(DIRECTORY config
    DESTINATION share/${PROJECT_NAME}/)

ament_export_dependencies(ament_cmake)
ament_export_dependencies(rclcpp)
ament_export_libraries(${PROJECT_NAME})

ament_package()
