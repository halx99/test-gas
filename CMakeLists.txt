###############################################################################
# Arm (GAS)
###############################################################################

project(test)

if(CPU_TYPE STREQUAL "arm64" OR CPU_TYPE STREQUAL "arm")

enable_language(ASM)

set(CMAKE_ASM_FLAGS "${CMAKE_C_FLAGS} ${CMAKE_ASM_FLAGS}")

string(TOUPPER ${CMAKE_BUILD_TYPE} CMAKE_BUILD_TYPE_UC)
set(EFFECTIVE_ASM_FLAGS "${CMAKE_ASM_FLAGS} ${CMAKE_ASM_FLAGS_${CMAKE_BUILD_TYPE_UC}}")
message(STATUS "CMAKE_ASM_FLAGS = ${EFFECTIVE_ASM_FLAGS}")

# Test whether we need gas-preprocessor.pl
if(CPU_TYPE STREQUAL "arm")
  file(WRITE ${CMAKE_CURRENT_BINARY_DIR}/gastest.S "
    .text
    .fpu neon
    .arch armv7a
    .object_arch armv4
    .arm
    pld [r0]
    vmovn.u16 d0, q0")
else()
  file(WRITE ${CMAKE_CURRENT_BINARY_DIR}/gastest.S "
    .text
    MYVAR .req x0
    movi v0.16b, #100
    mov MYVAR, #100
    .unreq MYVAR")
endif()

separate_arguments(CMAKE_ASM_FLAGS_SEP UNIX_COMMAND "${CMAKE_ASM_FLAGS}")

message(STATUS "Executing command: ${CMAKE_ASM_COMPILER} ${CMAKE_ASM_FLAGS_SEP}
    -x assembler-with-cpp -c ${CMAKE_CURRENT_BINARY_DIR}/gastest.S")
execute_process(COMMAND ${CMAKE_ASM_COMPILER} ${CMAKE_ASM_FLAGS_SEP}
    -x assembler-with-cpp -c ${CMAKE_CURRENT_BINARY_DIR}/gastest.S
  RESULT_VARIABLE RESULT OUTPUT_VARIABLE OUTPUT ERROR_VARIABLE ERROR)
if(NOT RESULT EQUAL 0)
  message(STATUS "GAS appears to be broken.  Trying gas-preprocessor.pl ...")
  message(STATUS "Executing command: gas-preprocessor.pl ${CMAKE_ASM_COMPILER}
      ${CMAKE_ASM_FLAGS_SEP} -x assembler-with-cpp -c
      ${CMAKE_CURRENT_BINARY_DIR}/gastest.S")
  execute_process(COMMAND gas-preprocessor.pl ${CMAKE_ASM_COMPILER}
      ${CMAKE_ASM_FLAGS_SEP} -x assembler-with-cpp -c
      ${CMAKE_CURRENT_BINARY_DIR}/gastest.S
    RESULT_VARIABLE RESULT OUTPUT_VARIABLE OUTPUT ERROR_VARIABLE ERROR)
  if(NOT RESULT EQUAL 0)
    simd_fail("SIMD extensions disabled: GAS is not working properly")
    return()
  else()
    message(STATUS "Using gas-preprocessor.pl")
    configure_file(gas-preprocessor.in gas-preprocessor @ONLY)
    set(CMAKE_ASM_COMPILER ${CMAKE_CURRENT_BINARY_DIR}/gas-preprocessor)
  endif()
else()
  message(STATUS "GAS is working properly")
endif()

endif()

file(REMOVE ${CMAKE_CURRENT_BINARY_DIR}/gastest.S)

message(STATUS "SIMD extensions: ${CPU_TYPE}")

add_library(test STATIC empty.cpp)
