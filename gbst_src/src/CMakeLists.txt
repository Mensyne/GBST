file(GLOB_RECURSE CPU_SOURCES *.cc *.h)
list(REMOVE_ITEM CPU_SOURCES ${gbst_SOURCE_DIR}/src/cli_main.cc)

#-- Object library
# Object library is necessary for jvm-package, which creates its own shared
# library.
if (USE_CUDA)
  file(GLOB_RECURSE CUDA_SOURCES *.cu *.cuh)
  add_library(objgbst OBJECT ${CPU_SOURCES} ${CUDA_SOURCES} ${PLUGINS_SOURCES})
  target_compile_definitions(objgbst
    PRIVATE -DGBST_USE_CUDA=1)
  target_include_directories(objgbst PRIVATE ${gbst_SOURCE_DIR}/cub/)
  target_compile_options(objgbst PRIVATE
    $<$<COMPILE_LANGUAGE:CUDA>:--expt-extended-lambda>
    $<$<COMPILE_LANGUAGE:CUDA>:--expt-relaxed-constexpr>
    $<$<COMPILE_LANGUAGE:CUDA>:-lineinfo>
    $<$<AND:$<NOT:$<CXX_COMPILER_ID:MSVC>>,$<COMPILE_LANGUAGE:CUDA>>:--std=c++11>
    $<$<COMPILE_LANGUAGE:CUDA>:${GEN_CODE}>)

  if (USE_NCCL)
    find_package(Nccl REQUIRED)
    target_include_directories(objgbst PRIVATE ${NCCL_INCLUDE_DIR})
    target_compile_definitions(objgbst PRIVATE -DGBST_USE_NCCL=1)
    list(APPEND SRC_LIBS ${NCCL_LIBRARY})
  endif (USE_NCCL)

  if (USE_NVTX)
    target_include_directories(objgbst PRIVATE "${NVTX_HEADER_DIR}")
    target_compile_definitions(objgbst PRIVATE -DGBST_USE_NVTX=1)
  endif (USE_NVTX)

  # OpenMP is mandatory for cuda version
  find_package(OpenMP REQUIRED)
  target_compile_options(objgbst PRIVATE
    $<$<COMPILE_LANGUAGE:CUDA>:-Xcompiler=${OpenMP_CXX_FLAGS}>
  )

  set_target_properties(objgbst PROPERTIES
    CUDA_SEPARABLE_COMPILATION OFF)
else (USE_CUDA)
  add_library(objgbst OBJECT ${CPU_SOURCES} ${PLUGINS_SOURCES})
endif (USE_CUDA)

target_include_directories(objgbst
  PRIVATE
  ${gbst_SOURCE_DIR}/include
  ${gbst_SOURCE_DIR}/dmlc-core/include
  ${gbst_SOURCE_DIR}/rabit/include)
target_compile_options(objgbst
  PRIVATE
  $<$<AND:$<CXX_COMPILER_ID:MSVC>,$<COMPILE_LANGUAGE:CXX>>:/MP>
  $<$<AND:$<NOT:$<CXX_COMPILER_ID:MSVC>>,$<COMPILE_LANGUAGE:CXX>>:-funroll-loops>)
if (WIN32 AND MINGW)
  target_compile_options(objgbst PUBLIC -static-libstdc++)
endif (WIN32 AND MINGW)

set_target_properties(objgbst PROPERTIES
  POSITION_INDEPENDENT_CODE ON
  CXX_STANDARD 11
  CXX_STANDARD_REQUIRED ON)
target_compile_definitions(objgbst
  PRIVATE
  -DDMLC_LOG_CUSTOMIZE=1  # enable custom logging
  $<$<NOT:$<CXX_COMPILER_ID:MSVC>>:_MWAITXINTRIN_H_INCLUDED>
  ${GBST_DEFINITIONS})
if (USE_DEBUG_OUTPUT)
  target_compile_definitions(objgbst PRIVATE -DGBST_USE_DEBUG_OUTPUT=1)
endif (USE_DEBUG_OUTPUT)

if (GBST_MM_PREFETCH_PRESENT)
  target_compile_definitions(objgbst
    PRIVATE
    -DGBST_MM_PREFETCH_PRESENT=1)
endif(GBST_MM_PREFETCH_PRESENT)
if (GBST_BUILTIN_PREFETCH_PRESENT)
  target_compile_definitions(objgbst
    PRIVATE
    -DGBST_BUILTIN_PREFETCH_PRESENT=1)
endif (GBST_BUILTIN_PREFETCH_PRESENT)

if (USE_OPENMP)
  find_package(OpenMP REQUIRED)
  if (OpenMP_CXX_FOUND OR OPENMP_FOUND)
    target_compile_options(objgbst PRIVATE  $<$<COMPILE_LANGUAGE:CXX>:${OpenMP_CXX_FLAGS}>)
    if ((NOT OpenMP_CXX_LIBRARIES) AND (NOT MSVC))  # old CMake doesn't define this variable
      set(OpenMP_CXX_LIBRARIES "gomp;pthread")
    endif ((NOT OpenMP_CXX_LIBRARIES) AND (NOT MSVC))
    list(APPEND SRC_LIBS ${OpenMP_CXX_LIBRARIES})
    set(LINKED_LIBRARIES_PRIVATE "${LINKED_LIBRARIES_PRIVATE};${SRC_LIBS}" PARENT_SCOPE)
  endif (OpenMP_CXX_FOUND OR OPENMP_FOUND)
endif (USE_OPENMP)

# For MSVC: Call msvc_use_static_runtime() once again to completely
# replace /MD with /MT. See https://github.com/dmlc/xgboost/issues/4462
# for issues caused by mixing of /MD and /MT flags
msvc_use_static_runtime()

# This grouping organises source files nicely in visual studio
auto_source_group("${CUDA_SOURCES}")
auto_source_group("${CPU_SOURCES}")

#-- End object library
