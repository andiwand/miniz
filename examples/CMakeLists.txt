add_executable(example1 example1.c)
target_link_libraries(example1 miniz)

add_executable(example2 example2.c)
target_link_libraries(example2 miniz)

add_executable(example3 example3.c)
target_link_libraries(example3 miniz)

add_executable(example4 example4.c)
target_link_libraries(example4 miniz)

add_executable(example5 example5.c)
target_link_libraries(example5 miniz)

add_executable(example6 example6.c)
target_link_libraries(example6 miniz)
if(${UNIX})
  target_link_libraries(example6 m)
endif()
