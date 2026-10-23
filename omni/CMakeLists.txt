# CMSIS-RTOS2（RTX5）构建入口。
#
# RTX_Config.h 由工程提供（方案 B）：工程把 RTE/CMSIS 或 boards/<board>/config
# 放进 include 路径即可覆盖默认值；模块只编译实现，不携带配置。
#
# 与 pack 的差异：`Config/handlers.c` 在 ARM.CMSIS-RTX.pdsc 里带
# condition="ARMv7-A Device"，是 Cortex-A 的异常处理（CUndefHandler 里用的
# __FPU_Enable 也只为 A-profile 的惰性 FPU 初始化），Cortex-M 不编它。
#
# EventRecorder：rtx_evr.h **无条件** include "EventRecorder.h" 与
# "EventRecorderConf.h"，实现由 cmsis-view 模块提供（Kconfig 里 select CMSIS_VIEW）。
# 模块注册按名字排序（cmsis_view 排在 cmsis_rtx 之后），这里不能靠 link 拿 include，
# 只能按路径加头文件目录。
#
# 模块根用相对路径推导（注册表的 OMNI_MODULE_* 变量在函数作用域里不一定可见）。

get_filename_component(_rtx_root "${CMAKE_CURRENT_LIST_DIR}/.." ABSOLUTE)
file(GLOB _rtx_sources CONFIGURE_DEPENDS "${_rtx_root}/Source/*.c")

# Cortex-A 才需要 Config/handlers.c（当前 omni 只支持 Cortex-M，留好分支）。
set(_rtx_handlers "")
if(OMNI_CPU MATCHES "cortex-a")
    set(_rtx_handlers "${_rtx_root}/Config/handlers.c")
endif()

# cmsis-view：EventRecorder 的头（EventRecorderConf.h 板可覆盖）。
get_filename_component(_rtx_modules_dir "${_rtx_root}/.." ABSOLUTE)
set(_cmsis_view_root "${_rtx_modules_dir}/cmsis-view")
if(NOT EXISTS "${_cmsis_view_root}/EventRecorder/Include/EventRecorder.h")
    message(FATAL_ERROR
        "CMSIS-RTX 需要 cmsis-view 模块提供 EventRecorder 头文件\n"
        "  期望位置: ${_cmsis_view_root}/EventRecorder/Include/EventRecorder.h\n"
        "  （modules/cmsis-view 子模块是否已初始化？）")
endif()
if(EXISTS "${PROJECT_SOURCE_DIR}/boards/${OMNI_BOARD}/config/EventRecorderConf.h")
    set(_rtx_evr_config "${PROJECT_SOURCE_DIR}/boards/${OMNI_BOARD}/config")
else()
    set(_rtx_evr_config "${_cmsis_view_root}/EventRecorder/Config")
endif()

# 板级 RTE_Components.h 选了 EventRecorder，就把对应 RTE_ 宏带给编译器：
# CMake 构建不定义 _RTE_，pack 那套宏不会自动生效，rtx_lib.c / rtx_evr.c 的
# 事件记录会退化成空实现（还会 #warning）。
#
# 两个来源都算「板要求事件记录」：
#   1. RTE_Components.h 里有 RTE_CMSIS_View_EventRecorder（pack 组件选中）
#   2. RTX_Config.h 里 OS_EVR_INIT 非 0（RTX 自己的开关，模板默认就是 1）
set(_rtx_rte_file "${PROJECT_SOURCE_DIR}/boards/${OMNI_BOARD}/config/RTE_Components.h")
set(_rtx_evr_enabled FALSE)
if(EXISTS "${_rtx_rte_file}")
    file(STRINGS "${_rtx_rte_file}" _rtx_evr_line
         REGEX "^[ \t]*#[ \t]*define[ \t]+RTE_CMSIS_View_EventRecorder([ \t]|$)")
    if(_rtx_evr_line)
        set(_rtx_evr_enabled TRUE)
    endif()
endif()
if(NOT _rtx_evr_enabled)
    set(_rtx_cfg_file "${PROJECT_SOURCE_DIR}/boards/${OMNI_BOARD}/config/RTX_Config.h")
    if(EXISTS "${_rtx_cfg_file}")
        file(STRINGS "${_rtx_cfg_file}" _rtx_evr_init
             REGEX "^[ \t]*#[ \t]*define[ \t]+OS_EVR_INIT[ \t]+[1-9]")
        if(_rtx_evr_init)
            set(_rtx_evr_enabled TRUE)
        endif()
    endif()
endif()

# RTX 的端口层（SVC_Handler / PendSV_Handler / SysTick_Handler / irqRtxLib）
# 在 Source/GCC/irq_armv7m.S 里。pdsc 的 GNUASM 条件显式接受 AC6
# （<accept Tcompiler="ARMCC" Toptions="AC6"/>）。omniConfig.cmake 已经
# enable_language(ASM)，所以 .S 直接当汇编源加进目标，由 armclang 汇编
# （.S 扩展名会自动走预处理器，rtx_def.h 里的 #include 才能展开）。
if(OMNI_CPU STREQUAL "cortex-m0" OR OMNI_CPU STREQUAL "cortex-m0plus")
    set(_rtx_irq_asm "${_rtx_root}/Source/GCC/irq_armv6m.S")
else()
    set(_rtx_irq_asm "${_rtx_root}/Source/GCC/irq_armv7m.S")
endif()

add_library(omni_module_cmsis_rtx OBJECT
    ${_rtx_sources}
    "${_rtx_irq_asm}"
    "${_rtx_root}/Config/RTX_Config.c"
    ${_rtx_handlers}
    "${OMNI_ROOT}/arch/arm/cmsis_6/rtos2/src/os_systick.c")

target_include_directories(omni_module_cmsis_rtx PUBLIC
    "${_rtx_root}/Include"
    "${_rtx_root}/Source"
    "${_cmsis_view_root}/EventRecorder/Include"
    "${_rtx_evr_config}"
    "${OMNI_ROOT}/arch/arm/cmsis_6/rtos2/include")

# os_systick.c 是 CMSIS 的通用源文件，要求 CMSIS_device_header 指向器件头。
# 板里有 RTE_Components.h 就用板的；没有就在构建目录生成最小替身（PRIVATE）。
if(NOT CONFIG_SOC_SERIES)
    message(FATAL_ERROR "CMSIS-RTX 需要 CONFIG_SOC_SERIES 才能定位器件头")
endif()
target_compile_definitions(omni_module_cmsis_rtx PUBLIC
    CMSIS_device_header="${CONFIG_SOC_SERIES}.h")

if(_rtx_evr_enabled)
    target_compile_definitions(omni_module_cmsis_rtx PUBLIC
        RTE_CMSIS_View_EventRecorder)
endif()

if(NOT EXISTS "${PROJECT_SOURCE_DIR}/boards/${OMNI_BOARD}/config/RTE_Components.h")
    set(_rtx_rte_dir "${CMAKE_CURRENT_BINARY_DIR}/rte")
    file(MAKE_DIRECTORY "${_rtx_rte_dir}")
    file(WRITE "${_rtx_rte_dir}/RTE_Components.h"
"/* 由 modules/cmsis-rtx/omni/CMakeLists.txt 生成，勿手改。
 * 只为模块内的 CMSIS 源文件（os_systick.c）提供器件头。
 * 板提供 boards/<board>/config/RTE_Components.h 时不会生成本文件。 */
#ifndef RTE_COMPONENTS_H
#define RTE_COMPONENTS_H

#ifndef CMSIS_device_header
#define CMSIS_device_header \"${CONFIG_SOC_SERIES}.h\"
#endif

#endif /* RTE_COMPONENTS_H */
")
    target_include_directories(omni_module_cmsis_rtx PRIVATE "${_rtx_rte_dir}")
endif()

target_link_libraries(omni_module_cmsis_rtx PUBLIC
    omni::framework omni::config omni::hal_conf omni::hal)

omni_apply_arch(omni_module_cmsis_rtx)
omni_apply_warnings(omni_module_cmsis_rtx)
