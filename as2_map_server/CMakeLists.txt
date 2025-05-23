cmake_minimum_required(VERSION 3.5)
set(PROJECT_NAME as2_map_server)
project(${PROJECT_NAME})

# Default to C++17 if not set
if(NOT CMAKE_CXX_STANDARD)
  set(CMAKE_CXX_STANDARD 17)
endif()

# Set Release as default build type if not set
if(NOT CMAKE_BUILD_TYPE)
  set(CMAKE_BUILD_TYPE Release)
endif()

set(CMAKE_POSITION_INDEPENDENT_CODE ON)

# find dependencies
set(PROJECT_DEPENDENCIES
  ament_cmake
  rclcpp
  as2_core
  as2_msgs
  std_msgs
  pluginlib
)

# find_package(backward_ros REQUIRED)

foreach(DEPENDENCY ${PROJECT_DEPENDENCIES})
  find_package(${DEPENDENCY} REQUIRED)
endforeach()

# Include necessary directories
include_directories(
  include
  include/${PROJECT_NAME}
)

###### PLUGIN BASE LIBRARY ######
add_library(${PROJECT_NAME}_base SHARED include/${PROJECT_NAME}/plugin_base.hpp)
set_target_properties(${PROJECT_NAME}_base PROPERTIES LINKER_LANGUAGE CXX)
ament_target_dependencies(${PROJECT_NAME}_base ${PROJECT_DEPENDENCIES})

install(
  DIRECTORY include/
  DESTINATION include
)

ament_export_libraries(${PROJECT_NAME}_base)
ament_export_targets(export_${PROJECT_NAME}_base)
ament_export_include_directories(include)

install(
  TARGETS ${PROJECT_NAME}_base
  EXPORT export_${PROJECT_NAME}_base
  ARCHIVE DESTINATION lib
  LIBRARY DESTINATION lib
  RUNTIME DESTINATION bin
  INCLUDES DESTINATION include
)


###### AS2_MAP_SERVER LIBRARIES ######
set(SOURCE_CPP_FILES
  src/map_server_node.cpp
  src/map_server.cpp
)

add_library(${PROJECT_NAME} SHARED ${SOURCE_CPP_FILES})
ament_target_dependencies(${PROJECT_NAME} ${PROJECT_DEPENDENCIES})

install(
  DIRECTORY include/
  DESTINATION include
)

ament_export_libraries(${PROJECT_NAME})
ament_export_targets(export_${PROJECT_NAME})
ament_export_include_directories(include)

install(
  TARGETS ${PROJECT_NAME}
  EXPORT export_${PROJECT_NAME}
  ARCHIVE DESTINATION lib
  LIBRARY DESTINATION lib
  RUNTIME DESTINATION bin
  INCLUDES DESTINATION include
)

###### AS2_MAP_SERVER NODE EXECUTABLE ######
add_executable(${PROJECT_NAME}_node src/map_server_node.cpp)
target_link_libraries(${PROJECT_NAME}_node ${PROJECT_NAME})
ament_target_dependencies(${PROJECT_NAME}_node ${PROJECT_DEPENDENCIES})

target_include_directories(${PROJECT_NAME}_node PUBLIC
  $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>
  $<INSTALL_INTERFACE:include>
)

install(TARGETS
  ${PROJECT_NAME}_node
  DESTINATION lib/${PROJECT_NAME}
)

#####
set(PLUGIN_LIST
  scan2occ_grid
  depth2occ_grid
  point_cloud2occ_grid
)

foreach(PLUGIN_NAME ${PLUGIN_LIST})
  add_subdirectory(plugins/${PLUGIN_NAME})
endforeach()

pluginlib_export_plugin_description_file(${PROJECT_NAME} plugins.xml)

###### SHARES INSTALLATION ######
install(DIRECTORY
  launch
  DESTINATION share/${PROJECT_NAME}
)

# For each plugin, install shared files
foreach(PLUGIN_NAME ${PLUGIN_LIST})
  install(DIRECTORY
    # launch
    plugins/${PLUGIN_NAME}/launch
    DESTINATION share/${PROJECT_NAME}
  )
  install(DIRECTORY
    # config
    plugins/${PLUGIN_NAME}/config
    DESTINATION share/${PROJECT_NAME}/plugins/${PLUGIN_NAME}
  )
endforeach()

# Build tests if testing is enabled
if(BUILD_TESTING)
  find_package(ament_lint_auto REQUIRED)
  file(GLOB_RECURSE EXCLUDE_FILES
    build/*
    install/*
  )
  set(AMENT_LINT_AUTO_FILE_EXCLUDE ${EXCLUDE_FILES})
  ament_lint_auto_find_test_dependencies()

  add_subdirectory(tests)
endif()

# Create the ament package
ament_package()