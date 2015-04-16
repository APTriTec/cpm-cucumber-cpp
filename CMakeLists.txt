cmake_minimum_required(VERSION 2.8 FATAL_ERROR)

#-----------------------------------------------------------------------
# CPM configuration
#-----------------------------------------------------------------------
set(CPM_MODULE_NAME cucumber_cpp)
set(CPM_LIB_TARGET_NAME ${CPM_MODULE_NAME})

if ((DEFINED CPM_DIR) AND (DEFINED CPM_UNIQUE_ID) AND (DEFINED CPM_TARGET_NAME))
  set(CPM_LIB_TARGET_NAME ${CPM_TARGET_NAME})
  set(CMAKE_MODULE_PATH ${CMAKE_MODULE_PATH} ${CPM_DIR})
  include(CPM)
else()
  set(CPM_DIR "${CMAKE_CURRENT_BINARY_DIR}/cpm-packages" CACHE TYPE STRING)
  find_package(Git)
  if(NOT GIT_FOUND)
    message(FATAL_ERROR "CPM requires Git.")
  endif()
  if (NOT EXISTS ${CPM_DIR}/CPM.cmake)
    execute_process(
      COMMAND "${GIT_EXECUTABLE}" clone https://github.com/iauns/cpm ${CPM_DIR}
      RESULT_VARIABLE error_code
      OUTPUT_VARIABLE head_sha)
    if(error_code)
      message(FATAL_ERROR "CPM failed to get the hash for HEAD")
    endif()
  endif()
  include(${CPM_DIR}/CPM.cmake)
endif()


CPM_AddModule("google-test"
  GIT_REPOSITORY "https://github.com/iauns/cpm-google-test"
  GIT_TAG 1.0.2
  EXPORT_MODULE true)

# All externals *must* define this.
CPM_ForceOnlyOneModuleVersion()

CPM_InitModule(${CPM_MODULE_NAME})


#------------------------------------------------------------------------------
# cucumber-cpp 
#------------------------------------------------------------------------------
# cucumber-cpp as a subdirectory so that it can capture your compiler flags
set(CUCUMBER_CPP_DIR ${CMAKE_CURRENT_SOURCE_DIR}/3rdParty/cucumber_cpp)
CPM_EnsureRepoIsCurrent(
  TARGET_DIR ${CUCUMBER_CPP_DIR}
  GIT_REPOSITORY "https://github.com/cucumber/cucumber-cpp"
  GIT_TAG v0.3
  USE_CACHING TRUE
  )

if(CMAKE_COMPILER_IS_GNUCXX)
    set(CMAKE_CXX_FLAGS_DEBUG "${CMAKE_CXX_FLAGS_DEBUG} -Weffc++")
    # TODO: A better fix should handle ld's --as-needed flag 
    set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} -Xlinker '--no-as-needed'")
elseif(MSVC)
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -DNOMINMAX") # exclude M$ min/max macros
    #set(CMAKE_CXX_FLAGS_DEBUG "${CMAKE_CXX_FLAGS_DEBUG} /analyze")
endif()


set(CUKE_INCLUDE_DIR ${CUCUMBER_CPP_DIR}/include)
file(GLOB_RECURSE CUKE_HEADERS "${CUKE_INCLUDE_DIR}/cucumber-cpp/*.hpp")

set(CUKE_SOURCES
    ${CUCUMBER_CPP_DIR}/src/ContextManager.cpp
    ${CUCUMBER_CPP_DIR}/src/CukeCommands.cpp
    ${CUCUMBER_CPP_DIR}/src/CukeEngine.cpp
    ${CUCUMBER_CPP_DIR}/src/CukeEngineImpl.cpp
    ${CUCUMBER_CPP_DIR}/src/StepManager.cpp
    ${CUCUMBER_CPP_DIR}/src/HookRegistrar.cpp
    ${CUCUMBER_CPP_DIR}/src/Regex.cpp
    ${CUCUMBER_CPP_DIR}/src/Scenario.cpp
    ${CUCUMBER_CPP_DIR}/src/Table.cpp
    ${CUCUMBER_CPP_DIR}/src/Tag.cpp
    ${CUCUMBER_CPP_DIR}/src/connectors/wire/WireServer.cpp
    ${CUCUMBER_CPP_DIR}/src/connectors/wire/WireProtocol.cpp
    ${CUCUMBER_CPP_DIR}/src/connectors/wire/WireProtocolCommands.cpp
    ${CUCUMBER_CPP_DIR}/src/drivers/GTestDriver.cpp
)

list(APPEND CUKE_SOURCES ${CUKE_HEADERS})

set(Boost_USE_STATIC_LIBS ${MSVC})

find_package(Boost 1.51 COMPONENTS thread system regex date_time REQUIRED)
include_directories(${Boost_INCLUDE_DIRS})
include_directories(${CUKE_INCLUDE_DIR})

if (WIN32 AND CMAKE_COMPILER_IS_GNUCXX)
  set(WINSOCK "-lws2_32")
endif()

add_library(${CPM_LIB_TARGET_NAME} STATIC ${CUKE_SOURCES} ${CUCUMBER_CPP_DIR}/src/main.cpp)
target_link_libraries(${CPM_LIB_TARGET_NAME} ${CPM_LIBRARIES}
  ${Boost_THREAD_LIBRARY} ${Boost_SYSTEM_LIBRARY} ${Boost_REGEX_LIBRARY}
  ${Boost_DATE_TIME_LIBRARY} ${WINSOCK})

CPM_ExportAdditionalIncludeDir(${CUCUMBER_CPP_DIR}/include)
CPM_ExportAdditionalIncludeDir(${Boost_INCLUDE_DIRS})


