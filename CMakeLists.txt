cmake_minimum_required(VERSION 3.0)
project(miniz C CXX)

set(MINIZ_API_VERSION 2)
set(MINIZ_MINOR_VERSION 1)
set(MINIZ_PATCH_VERSION 0)
set(MINIZ_VERSION 
    ${MINIZ_API_VERSION}.${MINIZ_MINOR_VERSION}.${MINIZ_PATCH_VERSION})

if(CMAKE_BUILD_TYPE STREQUAL "")
  # CMake defaults to leaving CMAKE_BUILD_TYPE empty. This screws up
  # differentiation between debug and release builds.
  set(CMAKE_BUILD_TYPE "Release" CACHE STRING
    "Choose the type of build, options are: None (CMAKE_CXX_FLAGS or \
CMAKE_C_FLAGS used) Debug Release RelWithDebInfo MinSizeRel." FORCE)
endif ()

option(BUILD_TESTS "Build tests" ON)
option(BUILD_EXAMPLES "Build examples" ON)
option(AMALGAMATE_SOURCES "Amalgamate sources into miniz.h/c" OFF)
option(BUILD_HEADER_ONLY "Build a header-only version" OFF)
option(BUILD_SHARED_LIBS "Build shared library instead of static" ON)

if(BUILD_HEADER_ONLY)
  set(AMALGAMATE_SOURCES ON CACHE BOOL "Build a header-only version" FORCE)
endif(BUILD_HEADER_ONLY)

if(AMALGAMATE_SOURCES)
  # Amalgamate
  file(COPY src/miniz.h DESTINATION ${CMAKE_CURRENT_BINARY_DIR}/amalgamation)
  file(READ src/miniz.h MINIZ_H)
  file(READ src/miniz_common.h MINIZ_COMMON_H)
  file(READ src/miniz_tdefl.h MINIZ_TDEFL_H)
  file(READ src/miniz_tinfl.h MINIZ_TINFL_H)
  file(READ src/miniz_zip.h MINIZ_ZIP_H)
  file(APPEND ${CMAKE_CURRENT_BINARY_DIR}/amalgamation/miniz.h
     "${MINIZ_COMMON_H} ${MINIZ_TDEFL_H} ${MINIZ_TINFL_H} ${MINIZ_ZIP_H}")

  file(COPY src/miniz.c DESTINATION ${CMAKE_CURRENT_BINARY_DIR}/amalgamation/)
  file(READ src/miniz_tdefl.c MINIZ_TDEFL_C)
  file(READ src/miniz_tinfl.c MINIZ_TINFL_C)
  file(READ src/miniz_zip.c MINIZ_ZIP_C)
  file(APPEND ${CMAKE_CURRENT_BINARY_DIR}/amalgamation/miniz.c
     "${MINIZ_TDEFL_C} ${MINIZ_TINFL_C} ${MINIZ_ZIP_C}")

  file(READ ${CMAKE_CURRENT_BINARY_DIR}/amalgamation/miniz.h AMAL_MINIZ_H)
  file(READ ${CMAKE_CURRENT_BINARY_DIR}/amalgamation/miniz.c AMAL_MINIZ_C)
  foreach(REPLACE_STRING miniz;miniz_common;miniz_tdefl;miniz_tinfl;miniz_zip;miniz_export)
    string(REPLACE "#include \"${REPLACE_STRING}.h\"" "" AMAL_MINIZ_H "${AMAL_MINIZ_H}")
    string(REPLACE "#include \"${REPLACE_STRING}.h\"" "" AMAL_MINIZ_C "${AMAL_MINIZ_C}")
  endforeach()

  string(CONCAT AMAL_MINIZ_H "#define MINIZ_EXPORT\n" "${AMAL_MINIZ_H}")

  if(BUILD_HEADER_ONLY)
    string(CONCAT AMAL_MINIZ_H "${AMAL_MINIZ_H}" "\n#ifndef MINIZ_HEADER_FILE_ONLY\n"
             "${AMAL_MINIZ_C}" "\n#endif // MINIZ_HEADER_FILE_ONLY\n")
    file(WRITE ${CMAKE_CURRENT_BINARY_DIR}/amalgamation/miniz.h "${AMAL_MINIZ_H}")
    add_library(${PROJECT_NAME} INTERFACE)
    
    # Might not be a good idea to force this on the library user
    # as it could bloat the global namespace
    # https://github.com/libevent/libevent/issues/460
    # target_compile_definitions(${PROJECT_NAME} 
    #     INTERFACE $<$<C_COMPILER_ID:GNU>:_GNU_SOURCE>)
    
    set_property(TARGET ${PROJECT_NAME} APPEND 
      PROPERTY INTERFACE_INCLUDE_DIRECTORIES
      $<BUILD_INTERFACE:${CMAKE_CURRENT_BINARY_DIR}/amalgamation>
      $<INSTALL_INTERFACE:include>
    )
  else(BUILD_HEADER_ONLY)
    string(CONCAT AMAL_MINIZ_C "#include \"miniz.h\"\n" "${AMAL_MINIZ_C}")
    file(WRITE ${CMAKE_CURRENT_BINARY_DIR}/amalgamation/miniz.h "${AMAL_MINIZ_H}")
    file(WRITE ${CMAKE_CURRENT_BINARY_DIR}/amalgamation/miniz.c "${AMAL_MINIZ_C}")
    set(miniz_SOURCE ${CMAKE_CURRENT_BINARY_DIR}/amalgamation/miniz.h
                     ${CMAKE_CURRENT_BINARY_DIR}/amalgamation/miniz.c)
    add_library(${PROJECT_NAME} STATIC ${miniz_SOURCE})
    target_include_directories(${PROJECT_NAME} PUBLIC
      $<BUILD_INTERFACE:${CMAKE_CURRENT_BINARY_DIR}/amalgamation>
      $<INSTALL_INTERFACE:include>
    )
  endif(BUILD_HEADER_ONLY)

  set(INSTALL_HEADERS ${CMAKE_CURRENT_BINARY_DIR}/amalgamation/miniz.h)
else(AMALGAMATE_SOURCES)
  include(GenerateExportHeader)

  add_library(${PROJECT_NAME} src/miniz.c src/miniz_zip.c src/miniz_tinfl.c src/miniz_tdefl.c)
  generate_export_header(${PROJECT_NAME})

  if(NOT BUILD_SHARED_LIBS)
    string(TOUPPER ${PROJECT_NAME} PROJECT_UPPER)
    set_target_properties(${PROJECT_NAME}
        PROPERTIES INTERFACE_COMPILE_DEFINITIONS ${PROJECT_UPPER}_STATIC_DEFINE)
  else()
    set_property(TARGET ${PROJECT_NAME} PROPERTY C_VISIBILITY_PRESET hidden)
  endif()

  set_property(TARGET ${PROJECT_NAME} PROPERTY VERSION ${MINIZ_VERSION})
  set_property(TARGET ${PROJECT_NAME} PROPERTY SOVERSION ${MINIZ_API_VERSION})
  
  target_include_directories(${PROJECT_NAME} PUBLIC
    $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/src>
    $<BUILD_INTERFACE:${CMAKE_CURRENT_BINARY_DIR}>
    $<INSTALL_INTERFACE:include>
  )
  
  file(GLOB INSTALL_HEADERS ${CMAKE_CURRENT_SOURCE_DIR}/*.h)
  list(APPEND
       INSTALL_HEADERS ${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_NAME}_export.h)
endif(AMALGAMATE_SOURCES)

if(NOT BUILD_HEADER_ONLY)
  target_compile_definitions(${PROJECT_NAME} 
    PRIVATE $<$<C_COMPILER_ID:GNU>:_GNU_SOURCE>)
endif()

set_property(TARGET ${PROJECT_NAME} PROPERTY
  INTERFACE_${PROJECT_NAME}_MAJOR_VERSION ${MINIZ_API_VERSION})
set_property(TARGET ${PROJECT_NAME} APPEND PROPERTY
  COMPATIBLE_INTERFACE_STRING ${PROJECT_NAME}_MAJOR_VERSION
)

install(TARGETS ${PROJECT_NAME} EXPORT ${PROJECT_NAME}Targets
  RUNTIME  DESTINATION bin
  ARCHIVE  DESTINATION lib
  LIBRARY  DESTINATION lib
  # users can use <miniz.h> or <miniz/miniz.h>
  INCLUDES DESTINATION include include/${PROJECT_NAME}
)

include(CMakePackageConfigHelpers)
write_basic_package_version_file(
  "${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_NAME}/${PROJECT_NAME}ConfigVersion.cmake"
  VERSION ${MINIZ_VERSION}
  COMPATIBILITY AnyNewerVersion
)

export(EXPORT ${PROJECT_NAME}Targets
  FILE "${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_NAME}/${PROJECT_NAME}Targets.cmake"
  NAMESPACE ${PROJECT_NAME}::
)
configure_file(Config.cmake.in
  "${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_NAME}/${PROJECT_NAME}Config.cmake"
  @ONLY
)

set(ConfigPackageLocation lib/cmake/${PROJECT_NAME})
install(EXPORT ${PROJECT_NAME}Targets
  FILE
    ${PROJECT_NAME}Targets.cmake
  NAMESPACE
    ${PROJECT_NAME}::
  DESTINATION
    ${ConfigPackageLocation}
)
install(
  FILES
    "${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_NAME}/${PROJECT_NAME}Config.cmake"
    "${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_NAME}/${PROJECT_NAME}ConfigVersion.cmake"
  DESTINATION
    ${ConfigPackageLocation}
  COMPONENT
    Devel
)

if(BUILD_TESTS)
  add_subdirectory(tests)
endif(BUILD_TESTS)

if(BUILD_EXAMPLES)
  add_subdirectory(examples)
endif(BUILD_EXAMPLES)

install(FILES ${INSTALL_HEADERS} DESTINATION "include/${PROJECT_NAME}")
