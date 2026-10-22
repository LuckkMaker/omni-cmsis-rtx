# CMSIS-RTOS2（RTX5）构建入口。
#
# RTX_Config.h 由工程提供（方案 B）：工程把 RTE/CMSIS 或 boards/<board>/config
# 放进 include 路径即可覆盖默认值；模块只编译实现，不携带配置。
#
# 模块根用相对路径推导（注册表的 OMNI_MODULE_* 变量在函数作用域里不一定可见）。

get_filename_component(_rtx_root "${CMAKE_CURRENT_LIST_DIR}/.." ABSOLUTE)
file(GLOB _rtx_sources CONFIGURE_DEPENDS "${_rtx_root}/Source/*.c")

add_library(omni_module_cmsis_rtx OBJECT
    ${_rtx_sources}
    "${_rtx_root}/Config/RTX_Config.c"
    "${_rtx_root}/Config/handlers.c"
    "${OMNI_ROOT}/arch/arm/cmsis_6/rtos2/src/os_systick.c")

target_include_directories(omni_module_cmsis_rtx PUBLIC
    "${_rtx_root}/Include"
    "${_rtx_root}/Source"
    "${OMNI_ROOT}/arch/arm/cmsis_6/rtos2/include")
target_link_libraries(omni_module_cmsis_rtx PUBLIC
    omni::framework omni::config omni::hal_conf omni::hal)

omni_apply_arch(omni_module_cmsis_rtx)
omni_apply_warnings(omni_module_cmsis_rtx)