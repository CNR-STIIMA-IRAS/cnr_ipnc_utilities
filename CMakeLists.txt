#############################
## PRELIMINARIES           ##
#############################
cmake_minimum_required(VERSION 3.16.0)

include(${CMAKE_CURRENT_SOURCE_DIR}/cmake/cnrConfigMacros.cmake)

get_project_name("package.xml" extracted_name extracted_version)

if("${CMAKE_MAJOR_VERSION}.${CMAKE_MINOR_VERSION}" LESS 3.0.2)
  project(${extracted_name})
else()
  project(${extracted_name} VERSION ${extracted_version} LANGUAGES CXX)
endif()

cnr_set_flags()
#############################
## END - PRELIMINARIES     ##
#############################


#############################
## OPTIONS                 ##
#############################
option(USE_ROS1                 "ROS SUPPORT" OFF)
option(ENABLE_TESTING           "ENABLE TESTING" OFF)
option(ENABLE_COVERAGE_TESTING  "ENABLE COVERAGE TESTING" OFF)
option(BUILD_EXAMPLES           "BUILD THE EXAMPLES" OFF)

if(USE_ROS1)
  set(ENABLE_TESTING ${CATKIN_ENABLE_TESTING})
endif()
#############################
## END - OPTIONS           ##
#############################




#################
## DEPENDECIES ##
#################

## common
find_package(Boost REQUIRED COMPONENTS system)

## common
find_package(PkgConfig REQUIRED)
pkg_check_modules(JSONCPP jsoncpp)

if(USE_ROS1)
  find_package(catkin REQUIRED)
  catkin_package( INCLUDE_DIRS include LIBRARIES ${PROJECT_NAME} )
endif(USE_ROS1)

cnr_install_directories(USE_ROS1 
  CNR_INSTALL_INCLUDE_DIR CNR_INSTALL_LIB_DIR CNR_INSTALL_BIN_DIR CNR_INSTALL_SHARE_DIR)
cnr_enable_testing(ENABLE_TESTING ENABLE_COVERAGE USE_ROS1)

#################
## Build       ##
#################
list(APPEND DEPENDENCIES_INCLUDE_DIRS ${Boost_INCLUDE_DIRS})
if(USE_ROS1)
  list(APPEND DEPENDENCIES_INCLUDE_DIRS ${catkin_INCLUDE_DIRS})
endif()

list(APPEND BUILD_INTERFACE_INCLUDE_DIRS ${DEPENDENCIES_INCLUDE_DIRS} ${CMAKE_CURRENT_SOURCE_DIR}/include)
list(APPEND INSTALL_INTERFACE_INCLUDE_DIRS include ${DEPENDENCIES_INCLUDE_DIRS} include)

list(APPEND DEPENDENCIES_LIBRARIES Boost::system pthread rt ${JSONCPP_LIBRARIES} ${BOOST_LIBRARIES})
if(USE_ROS1)
  list(APPEND DEPENDENCIES_LIBRARIES ${catkin_LIBRARIES})
endif()

## CNR_IPC_UTILITIES
add_library(${PROJECT_NAME}
              src/${PROJECT_NAME}/asio_ipnc.cpp
)

target_include_directories(${PROJECT_NAME} PUBLIC
  "$<BUILD_INTERFACE:${BUILD_INTERFACE_INCLUDE_DIRS}>"
    "$<INSTALL_INTERFACE:${INSTALL_INTERFACE_INCLUDE_DIRS}>")

target_link_libraries(${PROJECT_NAME} ${DEPENDENCIES_LIBRARIES} )

## CNR_IPC_EXAMPLES
if(BUILD_EXAMPLES)
  add_executable(server_client_example examples/server_client_example.cpp)
  target_link_libraries(server_client_example ${PROJECT_NAME})
endif()
#####################################################################

#############
## Install ##
#############
install(DIRECTORY include/${PROJECT_NAME}
        DESTINATION ${CNR_INSTALL_INCLUDE_DIR}
        FILES_MATCHING PATTERN "*.h"
        PATTERN ".git" EXCLUDE
)

install(TARGETS ${PROJECT_NAME}
        EXPORT export_cnr_ipnc_utilities
        ARCHIVE DESTINATION ${CNR_INSTALL_LIB_DIR}
        LIBRARY DESTINATION ${CNR_INSTALL_LIB_DIR}
        RUNTIME DESTINATION ${CNR_INSTALL_BIN_DIR}
)
###################
## END - Install ##
###################

##################################
## CMAKE CONFIG (IF NOT CATKIN) ##
##################################
if(NOT USE_ROS1) 
  set(export_targets ${export_targets};${PROJECT_NAME})
  export(EXPORT export_cnr_ipnc_utilities
    FILE "${PROJECT_BINARY_DIR}/export_cnr_ipnc_utilities.cmake")

  # Create the cnr_ipnc_utilitiesConfig.cmake
  set(CONF_INCLUDE_DIRS "${CMAKE_INSTALL_PREFIX}/include")
  configure_file(cnr_ipnc_utilitiesConfig.cmake.in
    "${PROJECT_BINARY_DIR}/${CMAKE_FILES_DIRECTORY}/cnr_ipnc_utilitiesConfig.cmake" @ONLY)

  # Create the cnr_ipnc_utilitiesConfigVersion.cmake
  configure_file(cnr_ipnc_utilitiesConfigVersion.cmake.in
    "${PROJECT_BINARY_DIR}/cnr_ipnc_utilitiesConfigVersion.cmake" @ONLY)
  
  install(FILES
    "${PROJECT_BINARY_DIR}/${CMAKE_FILES_DIRECTORY}/cnr_ipnc_utilitiesConfig.cmake"
    "${PROJECT_BINARY_DIR}/cnr_ipnc_utilitiesConfigVersion.cmake"
    DESTINATION "share/${PROJECT_NAME}/cmake" COMPONENT dev
  )

  install(EXPORT export_cnr_ipnc_utilities
    DESTINATION "share/${PROJECT_NAME}/cmake"
    FILE export_cnr_ipnc_utilities.cmake
    COMPONENT dev
  )
endif()


##########################
## Testing              ##
##########################
if(ENABLE_TESTING)
  include(GoogleTest)
  add_executable(cnr_ipnc_utilities_ipc_test
     test/test_ipc.cpp
  )
  if(${CMAKE_VERSION} VERSION_GREATER  "3.16.0")
    target_link_libraries(cnr_ipnc_utilities_ipc_test 
      ${PROJECT_NAME} Threads::Threads GTest::gtest_main)
  else()
    target_link_libraries(cnr_ipnc_utilities_ipc_test ${PROJECT_NAME} GTest::gtest_main)
    if(THREADS_HAVE_PTHREAD_ARG)
      target_compile_options(cnr_ipnc_utilities_ipc_test PUBLIC "-pthread")
    endif()
    if(CMAKE_THREAD_LIBS_INIT)
      target_link_libraries(cnr_ipnc_utilities_ipc_test "${CMAKE_THREAD_LIBS_INIT}")
    endif()
  endif()
  target_compile_definitions(cnr_ipnc_utilities_ipc_test PRIVATE TEST_DIR="${CMAKE_CURRENT_LIST_DIR}/test")  
  gtest_discover_tests(cnr_ipnc_utilities_ipc_test)
 endif(ENABLE_TESTING)


##########################
## Coverage Testing     ##
##########################
# if( ENABLE_TESTING AND ENABLE_COVERAGE_TESTING AND NOT WIN32 )
#   if(USE_ROS)
#     message(STATUS "The coverage testing is enabled")
#     set(COVERAGE_EXCLUDES "*/${PROJECT_NAME}/test*")
#     add_code_coverage(
#       NAME coverage_report
#       DEPENDENCIES cnr_ipnc_utilities_server_test
#     )
#   else()
#     ## NOTHING TO DO, SINCE GTEST ALREADY CREATES THE COVERAGE
#     ## IF THE COMPILING OPTIONS ARE SET
#   endif()
#endif()

