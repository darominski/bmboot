cmake_minimum_required(VERSION 3.17)

if (BUILD_MONITOR OR BMBOOT_BUILD_PAYLOAD)
    if (CMAKE_SIZE STREQUAL "")
        message(FATAL_ERROR "You must provide CMAKE_SIZE option")
    endif()

    # Note that link options are NOT propagated via static libraries to the ultimate executable
    add_link_options(
            #-ffreestanding                  # no reason to use this since we *do* have a libc
            -specs=nosys.specs              # without this we don't get _exit, _sbrk and I/O functions
            )
endif()

project(bmboot C CXX ASM)

include(cmake/Bmboot.cmake)

# These flags will be added to all targets defined in this file
# No nicer way to do --gc-sections: https://gitlab.kitware.com/cmake/cmake/-/issues/23235
add_compile_options(-Wall -ffunction-sections -fdata-sections)
add_link_options(-Wl,--gc-sections)

function(add_bmboot_payload_library)
    set(TARGET bmboot_payload_runtime)
    add_library(${TARGET} STATIC
            src/executor/executor.cpp
            src/executor/executor_asm.S
            src/executor/payload/payload_runtime.cpp
            src/executor/payload/syscalls.cpp
            src/executor/payload/syscalls.h
            src/platform/zynqmp/executor/asm_vectors.S
            src/platform/zynqmp/executor/boot.S
            src/platform/zynqmp/executor/payload/vectors_el1.cpp
            src/platform/zynqmp/executor/sleep.cpp
            src/platform/zynqmp/executor/translation_table.S
            src/platform/zynqmp/executor/xil-crt0.S
            )

    target_compile_definitions(${TARGET} PUBLIC __bmboot__=1)
    target_compile_features(${TARGET} PUBLIC cxx_std_20)
    target_compile_options(${TARGET} PRIVATE -Wall)

    target_include_directories(${TARGET} PUBLIC
            include
            )

    target_include_directories(${TARGET} PRIVATE
            src
            src/executor
            src/executor/payload
            src/platform/zynqmp
            src/platform/zynqmp/executor
            src/platform/zynqmp/executor/payload
            )

    target_link_options(${TARGET} PUBLIC
            # These are necessary when syscalls.cpp is in a static library: https://stackoverflow.com/q/34986536
            -Wl,--undefined=_close
            -Wl,--undefined=_fstat
            -Wl,--undefined=_isatty
            -Wl,--undefined=_lseek
            -Wl,--undefined=_read
            -Wl,--undefined=_write
            )
endfunction()

if (BMBOOT_BUILD_PAYLOAD)
    add_bmboot_payload_library()

    add_bmboot_payload(bmboot_example_payload src/payloads/hello_world.cpp)

    # -----------------------------------------------------------------------------------------------------------
elseif (BUILD_MONITOR)
    # to be precise, what this configuration builds is the set of everything that bmboot_manager requires:
    #   - monitor_zynqmp_cpuN
    #   - test payloads

    # Monitor target (common)

    add_library(monitor_zynqmp OBJECT
            include/bmboot.hpp
            src/executor/executor.cpp
            src/executor/executor_asm.S
            src/executor/monitor/monitor_asm.S
            src/executor/monitor/monitor.cpp
            src/executor/monitor/smc_handlers.cpp
            src/platform/zynqmp/executor/asm_vectors.S
            src/platform/zynqmp/executor/boot.S
            src/platform/zynqmp/executor/monitor/interrupt_controller.cpp
            src/platform/zynqmp/executor/monitor/vectors_el3.cpp
            src/platform/zynqmp/executor/sleep.cpp
            src/platform/zynqmp/executor/translation_table.S
            src/platform/zynqmp/executor/xil-crt0.S
            src/utility/crc32.c
            )

    target_include_directories(monitor_zynqmp PRIVATE
            include
            src
            src/executor
            src/executor/monitor
            src/platform/zynqmp
            src/platform/zynqmp/executor
            src/platform/zynqmp/executor/monitor
            )
    target_compile_features(monitor_zynqmp PUBLIC cxx_std_20)
    target_link_libraries(monitor_zynqmp PRIVATE m)

    # Instantiate monitor target for each CPU core

    foreach(CPU ${BMBOOT_ALL_CPUS})
        set(TARGET monitor_zynqmp_cpu${CPU})

        add_executable("${TARGET}" $<TARGET_OBJECTS:monitor_zynqmp>)
        # TODO: according to https://cmake.org/cmake/help/v3.4/manual/cmake-buildsystem.7.html#object-libraries,
        #       we should not link against monitor_zynqmp. For the time being, it seems to work, anyway.
        target_link_libraries("${TARGET}" PRIVATE monitor_zynqmp)

        target_link_options(${TARGET} PRIVATE
                -Wl,-T,${CMAKE_CURRENT_SOURCE_DIR}/src/executor/monitor/monitor_cpu${CPU}.ld
                )

        add_custom_command(TARGET ${TARGET} POST_BUILD
                COMMAND ${CMAKE_OBJCOPY} -Obinary $<TARGET_FILE:${TARGET}> ${CMAKE_BINARY_DIR}/${TARGET}.bin
                COMMAND ${CMAKE_OBJDUMP} -dt $<TARGET_FILE:${TARGET}> > ${TARGET}.txt
                COMMENT "Building ${CMAKE_BINARY_DIR}/${TARGET}.bin")

        add_custom_command(TARGET ${TARGET} POST_BUILD
                COMMAND ${CMAKE_SIZE} $<TARGET_FILE:${TARGET}>
            )

    endforeach()

    # -----------------------------------------------------------------------------------------------------------

    # use this opportunity to also build test payloads

    add_bmboot_payload_library()

    foreach(PAYLOAD
            access_violation
            adrian_irq_demo
            exception_caught_demo
            hello_world
            pmu_demo
            timer_demo
            )
        add_bmboot_payload(payload_${PAYLOAD} src/payloads/${PAYLOAD}.cpp)
    endforeach()

    add_bmboot_payload(payload_MemoryLatency src/benchmarks/MemoryLatency/MemoryLatency.c src/benchmarks/MemoryLatency/MemoryLatency_arm.s)
    add_bmboot_payload(payload_fpga_latency
            src/benchmarks/fpga_latency/fpga_latency.cpp
            src/benchmarks/fpga_latency/fpga_latency.s)

    # -----------------------------------------------------------------------------------------------------------
else()
    set(GENERATED_INCLUDE ${CMAKE_BINARY_DIR}/generated_include)
    file(MAKE_DIRECTORY ${GENERATED_INCLUDE})

    include(ExternalProject)

    # add executor code as ExternalProject
    ExternalProject_Add(monitor_zynqmp
            SOURCE_DIR ${CMAKE_CURRENT_SOURCE_DIR}
            CMAKE_ARGS
                -DCMAKE_TOOLCHAIN_FILE=${CMAKE_CURRENT_LIST_DIR}/Aarch64BareMetal.cmake
                -DCMAKE_MAKE_PROGRAM=${CMAKE_MAKE_PROGRAM}
                -DCMAKE_BUILD_TYPE=RelWithDebInfo
                -DBUILD_MONITOR=ON
            INSTALL_COMMAND ""
            BUILD_ALWAYS ON
            BUILD_BYPRODUCTS
                "${CMAKE_CURRENT_BINARY_DIR}/monitor_zynqmp-prefix/src/monitor_zynqmp-build/monitor_zynqmp_cpu1.bin"
                "${CMAKE_CURRENT_BINARY_DIR}/monitor_zynqmp-prefix/src/monitor_zynqmp-build/monitor_zynqmp_cpu2.bin"
                "${CMAKE_CURRENT_BINARY_DIR}/monitor_zynqmp-prefix/src/monitor_zynqmp-build/monitor_zynqmp_cpu3.bin"
            )

    # We need to convert monitor binary to a C++ header
    include(cmake/FileEmbed.cmake)
    ExternalProject_Get_Property(monitor_zynqmp BINARY_DIR)
    set(MONITOR_ZYNQMP_HPP_ALL)

    foreach(CPU ${BMBOOT_ALL_CPUS})
        set(MONITOR_ZYNQMP_HPP ${GENERATED_INCLUDE}/monitor_zynqmp_cpu${CPU}.hpp)
        FileEmbed_Add(${BINARY_DIR}/monitor_zynqmp_cpu${CPU}.bin ${MONITOR_ZYNQMP_HPP} monitor_zynqmp_cpu${CPU}_payload)
        list(APPEND MONITOR_ZYNQMP_HPP_ALL ${MONITOR_ZYNQMP_HPP})
    endforeach()

    add_subdirectory(elfload)

    add_library(bmboot_manager STATIC
            include/bmboot.hpp
            include/bmboot/domain.hpp
            src/bmboot_internal.hpp
            src/manager/configuration.cpp
            src/manager/coredump_linux.cpp
            src/manager/domain.cpp
            src/manager/domain_helpers.cpp
            src/platform/zynqmp/manager/zynqmp_manager.cpp
            src/utility/crc32.c
            src/utility/to_string.cpp

            ${MONITOR_ZYNQMP_HPP_ALL}
            )

    add_dependencies(bmboot_manager monitor_zynqmp)

    target_include_directories(bmboot_manager PUBLIC include)
    target_include_directories(bmboot_manager PRIVATE
            src
            src/platform/zynqmp
            src/platform/zynqmp/manager

            ${GENERATED_INCLUDE}
            )

#    target_compile_features(bmboot_manager cpp20)
#    set_property(TARGET bmboot_manager PROPERTY CXX_STANDARD 20)
    target_compile_features(bmboot_manager PUBLIC cxx_std_20)

    target_link_libraries(bmboot_manager PUBLIC elfload)

    add_executable(bmctl
            src/tools/bmctl.cpp
            )
    target_link_libraries(bmctl PUBLIC bmboot_manager)

    # Testing
    if (DEFINED LIBRARIES_HOME)
        add_subdirectory("${LIBRARIES_HOME}/googletest-1.11.0" "${CMAKE_CURRENT_BINARY_DIR}/googletest")

        add_executable(bmtest
                src/tests/tests.cpp
                src/tools/bmtest.cpp
                )
        target_link_libraries(bmtest PUBLIC bmboot_manager GTest::gtest pthread dl)
        target_link_options(bmtest PUBLIC  -Wl,-dynamic-linker,/my-lib/ld-linux-aarch64.so.1 -Wl,-rpath,/my-lib)
    else()
        message(WARNING "LIBRARIES_HOME not provided, will not build tests")
    endif()

    add_executable(console src/tools/console.cpp)
    target_link_libraries(console PUBLIC bmboot_manager)

    add_executable(MemoryLatency src/benchmarks/MemoryLatency/MemoryLatency.c src/benchmarks/MemoryLatency/MemoryLatency_arm.s)
    target_link_libraries(MemoryLatency PUBLIC m)

    foreach(TOOL bmctl console MemoryLatency)
        # Make sure bmctl is linked fully statically
        # This is only a temporary workaround for the discrepancy between library versions expected by our compiler
        # and available on the target OS (PetaLinux 2019).
        # Can't do this for Bmtest because it uses pthread which *requires* dynamic linking. Or something like that.
        # TODO: the clean solution would be to use a toolchain that matches the target OS
        target_link_options(${TOOL} PRIVATE -static -static-libgcc -static-libstdc++)
    endforeach()

endif()
