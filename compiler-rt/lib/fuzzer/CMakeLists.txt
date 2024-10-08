set(LIBFUZZER_SOURCES
  FuzzerClangCounters.cpp
  FuzzerCrossOver.cpp
  FuzzerDriver.cpp
  FuzzerExtFunctionsDlsym.cpp
  FuzzerExtFunctionsDlsymWin.cpp
  FuzzerExtFunctionsWeak.cpp
  FuzzerExtraCounters.cpp
  FuzzerIO.cpp
  FuzzerIOPosix.cpp
  FuzzerIOWindows.cpp
  FuzzerLoop.cpp
  FuzzerMerge.cpp
  FuzzerMutate.cpp
  FuzzerSHA1.cpp
  FuzzerShmemPosix.cpp
  FuzzerShmemWindows.cpp
  FuzzerTracePC.cpp
  FuzzerUtil.cpp
  FuzzerUtilDarwin.cpp
  FuzzerUtilFuchsia.cpp
  FuzzerUtilLinux.cpp
  FuzzerUtilPosix.cpp
  FuzzerUtilWindows.cpp
  )

CHECK_CXX_SOURCE_COMPILES("
  static thread_local int blah;
  int main() {
  return 0;
  }
  " HAS_THREAD_LOCAL)

set(LIBFUZZER_CFLAGS ${SANITIZER_COMMON_CFLAGS})

if("${CMAKE_SYSTEM_NAME}" STREQUAL "Linux" AND COMPILER_RT_LIBCXX_PATH)
  list(APPEND LIBFUZZER_CFLAGS -nostdinc++ -D_LIBCPP_ABI_VERSION=Fuzzer)
endif()

append_list_if(COMPILER_RT_HAS_OMIT_FRAME_POINTER_FLAG -fno-omit-frame-pointer LIBFUZZER_CFLAGS)

if (CMAKE_CXX_FLAGS MATCHES "fsanitize-coverage")
  list(APPEND LIBFUZZER_CFLAGS -fno-sanitize-coverage=trace-pc-guard,edge,trace-cmp,indirect-calls,8bit-counters)
endif()

if(NOT HAS_THREAD_LOCAL)
  list(APPEND LIBFUZZER_CFLAGS -Dthread_local=__thread)
endif()

if(APPLE)
  set(FUZZER_SUPPORTED_OS osx)
endif()

add_compiler_rt_object_libraries(RTfuzzer
  OS ${FUZZER_SUPPORTED_OS}
  ARCHS ${FUZZER_SUPPORTED_ARCH}
  SOURCES ${LIBFUZZER_SOURCES}
  CFLAGS ${LIBFUZZER_CFLAGS})

add_compiler_rt_object_libraries(RTfuzzer_main
  OS ${FUZZER_SUPPORTED_OS}
  ARCHS ${FUZZER_SUPPORTED_ARCH}
  SOURCES FuzzerMain.cpp
  CFLAGS ${LIBFUZZER_CFLAGS})

add_compiler_rt_runtime(clang_rt.fuzzer
  STATIC
  OS ${FUZZER_SUPPORTED_OS}
  ARCHS ${FUZZER_SUPPORTED_ARCH}
  OBJECT_LIBS RTfuzzer RTfuzzer_main
  CFLAGS ${LIBFUZZER_CFLAGS}
  PARENT_TARGET fuzzer)

add_compiler_rt_runtime(clang_rt.fuzzer_no_main
  STATIC
  OS ${FUZZER_SUPPORTED_OS}
  ARCHS ${FUZZER_SUPPORTED_ARCH}
  OBJECT_LIBS RTfuzzer
  CFLAGS ${LIBFUZZER_CFLAGS}
  PARENT_TARGET fuzzer)

if("${CMAKE_SYSTEM_NAME}" STREQUAL "Linux" AND COMPILER_RT_LIBCXX_PATH)
  macro(partially_link_libcxx name dir arch)
    set(cxx_${arch}_merge_dir "${CMAKE_CURRENT_BINARY_DIR}/cxx_${arch}_merge.dir")
    file(MAKE_DIRECTORY ${cxx_${arch}_merge_dir})
    add_custom_command(TARGET clang_rt.${name}-${arch} POST_BUILD
      COMMAND ${CMAKE_LINKER} --whole-archive "$<TARGET_LINKER_FILE:clang_rt.${name}-${arch}>" --no-whole-archive ${dir}/src/libcxx_fuzzer_${arch}-build/lib/libc++.a -r -o ${name}.o
      COMMAND ${CMAKE_OBJCOPY} --localize-hidden ${name}.o
      COMMAND ${CMAKE_COMMAND} -E remove "$<TARGET_LINKER_FILE:clang_rt.${name}-${arch}>"
      COMMAND ${CMAKE_AR} qcs "$<TARGET_LINKER_FILE:clang_rt.${name}-${arch}>" ${name}.o
      WORKING_DIRECTORY ${cxx_${arch}_merge_dir}
    )
  endmacro()

  foreach(arch ${FUZZER_SUPPORTED_ARCH})
    get_target_flags_for_arch(${arch} TARGET_CFLAGS)
    set(LIBCXX_${arch}_PREFIX ${CMAKE_CURRENT_BINARY_DIR}/libcxx_fuzzer_${arch})
    add_custom_libcxx(libcxx_fuzzer_${arch} ${LIBCXX_${arch}_PREFIX}
      CFLAGS ${TARGET_CFLAGS}
             -D_LIBCPP_ABI_VERSION=Fuzzer
             -D_LIBCPP_DISABLE_VISIBILITY_ANNOTATIONS=1
             -fvisibility=hidden
      CMAKE_ARGS -DLIBCXX_ENABLE_EXCEPTIONS=OFF
                 -DLIBCXX_CXX_ABI=none)
    target_compile_options(RTfuzzer.${arch} PRIVATE -isystem ${COMPILER_RT_LIBCXX_PATH}/include)
    add_dependencies(RTfuzzer.${arch} libcxx_fuzzer_${arch}-build)
    target_compile_options(RTfuzzer_main.${arch} PRIVATE -isystem ${COMPILER_RT_LIBCXX_PATH}/include)
    add_dependencies(RTfuzzer_main.${arch} libcxx_fuzzer_${arch}-build)
    partially_link_libcxx(fuzzer_no_main ${LIBCXX_${arch}_PREFIX} ${arch})
    partially_link_libcxx(fuzzer ${LIBCXX_${arch}_PREFIX} ${arch})
  endforeach()
endif()

if(COMPILER_RT_INCLUDE_TESTS)
  add_subdirectory(tests)
endif()
