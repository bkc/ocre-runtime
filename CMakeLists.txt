cmake_minimum_required(VERSION 3.20.0)

find_package(Zephyr REQUIRED HINTS $ENV{ZEPHYR_BASE})

project(OcreRuntime VERSION ${APP_VERSION_MAJOR}.${APP_VERSION_MINOR}.${APP_PATCHLEVEL}.${APP_VERSION_TWEAK} LANGUAGES C ASM)

set(CMAKE_EXPORT_COMPILE_COMMANDS ON)

execute_process(
  	COMMAND git describe --long --dirty --always
  	WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}
  	OUTPUT_VARIABLE RAW_GIT_VERSION
  	OUTPUT_STRIP_TRAILING_WHITESPACE
  )
  string(REPLACE "-dirty" "+" BUILD_INFO ${RAW_GIT_VERSION})

  string(TIMESTAMP BUILD_DATE "%d-%m-%Y, %H:%M" UTC )

  cmake_host_system_information(RESULT BUILD_MACHINE QUERY HOSTNAME)

  string(PREPEND BUILD_INFO dev-)
  
  string(REPLACE "-" ";" GIT_INFO_LIST ${RAW_GIT_VERSION})
  list(GET GIT_INFO_LIST 0 VERSION_NUMBER)
  message("VERSION NUMBER: ${VERSION_NUMBER}")
  zephyr_compile_options(-DVERSION_INFO="${VERSION_NUMBER}")
  message("BUILD DATE: ${BUILD_DATE}")
  zephyr_compile_options(-DVERSION_BUILD_DATE="${BUILD_DATE}")
  message("BUILD MACHINE: ${BUILD_MACHINE}")
  zephyr_compile_options(-DVERSION_BUILD_MACHINE="${BUILD_MACHINE}")
  message("BUILD_INFO: ${BUILD_INFO}")
  zephyr_compile_options(-DVERSION_BUILD_INFO="${BUILD_INFO}")

# Determine the ISA of the target and set appropriately
  if (DEFINED CONFIG_ISA_THUMB2)
    set (TARGET_ISA THUMB)
  elseif (DEFINED CONFIG_ISA_ARM)
    set (TARGET_ISA ARM)
  elseif (DEFINED CONFIG_X86)
    set (TARGET_ISA X86_32)
  elseif (DEFINED CONFIG_XTENSA)
    set (TARGET_ISA XTENSA)
  elseif (DEFINED CONFIG_RISCV)
    set (TARGET_ISA RISCV32)
  elseif (DEFINED CONFIG_ARCH_POSIX)
    # Technically, this is not correct as the CPU architecture is not set.  This assumes POSIX is x86 32-bit
    set (TARGET_ISA X86_32)  
  else ()
    message (FATAL_ERROR "Unsupported ISA: ${CONFIG_ARCH}")
  endif ()
  message("TARGET ISA: ${TARGET_ISA}")

############################
set (WAMR_BUILD_PLATFORM "zephyr")
set (WAMR_BUILD_TARGET ${TARGET_ISA})
set (WAMR_BUILD_INTERP 1)
set (WAMR_BUILD_FAST_INTERP 0)
set (WAMR_BUILD_AOT 0)
set (WAMR_BUILD_JIT 0)
set (WAMR_BUILD_LIBC_BUILTIN 1)
set (WAMR_BUILD_LIBC_WASI 0)
set (WAMR_BUILD_APP_FRAMEWORK 0)
set (WAMR_BUILD_APP_LIST WAMR_APP_BUILD_BASE )
set (WAMR_MPU_STACK_COUNT 5)

# Override the global heap usage
if (NOT DEFINED WAMR_BUILD_GLOBAL_HEAP_POOL)
  set (WAMR_BUILD_GLOBAL_HEAP_POOL 1)
endif ()

# Override the global heap size for small devices
if (NOT DEFINED WAMR_BUILD_GLOBAL_HEAP_SIZE)
  #set (WAMR_BUILD_GLOBAL_HEAP_SIZE 131072) # 128 KB
  set (WAMR_BUILD_GLOBAL_HEAP_SIZE 32767) # 32 KB
endif ()

set (WAMR_ROOT_DIR ${ZEPHYR_WASM_MICRO_RUNTIME_MODULE_DIR})

include (${WAMR_ROOT_DIR}/build-scripts/runtime_lib.cmake)


##############

zephyr_include_directories(inc)

zephyr_library_include_directories(${ZEPHYR_BASE}/subsys/net/ip)
zephyr_library_link_libraries(LITTLEFS)

set(lib_sources
  # Libraries
  src/ocre/sm/sm.c
  src/ocre/fs/fs.c
  src/ocre/timer/ocre_timer.c
  src/ocre/container_healthcheck/ocre_container_healthcheck.c
  src/ocre/ocre_container_runtime/ocre_container_runtime.c
  src/ocre/component/component.c
)

set(component_sources
  # Components
  src/components/container_supervisor/cs_main.c
  src/components/container_supervisor/cs_sm.c
 
  # Ocre APIs
  src/ocre/api/ocre_api.c
) 

zephyr_library_sources(${lib_sources} ${component_sources} ${WAMR_RUNTIME_LIB_SOURCE} src/main.c)


zephyr_include_directories(
  src/
  src/components/app_manager
  src/components/device_manager
  ${WAMR_ROOT_DIR}/core/shared/coap/er-coap
  ${WAMR_ROOT_DIR}/core/shared/coap/extension
)
