# Copyright (c) 2024 LuckkMaker
# All rights reserved.
#
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
#     http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.

set(RTX_CFG_SRC_SOURCE_FILE ${CMAKE_CURRENT_LIST_DIR}/Config/RTX_Config.c)
set(RTX_CFG_SRC_TARGET_FILE ${OMNI_RTOS_CFG_DIR}/RTX_Config.c)
omni_add_configure_file(${RTX_CFG_SRC_SOURCE_FILE} ${RTX_CFG_SRC_TARGET_FILE})

set(RTX_CFG_SOURCE_FILE ${CMAKE_CURRENT_LIST_DIR}/RTX_Config.in)
set(RTX_CFG_TARGET_FILE ${OMNI_RTOS_CFG_DIR}/RTX_Config.h)
omni_add_configure_file(${RTX_CFG_SOURCE_FILE} ${RTX_CFG_TARGET_FILE})

set(RTE_CONPONENTS_FILE ${CMAKE_CURRENT_LIST_DIR}/RTE_Components.in)
# TODO:(Luckk) If Use RTX, then generate RTE_Components.h in RTE directory
set(RTE_CONPONENTS_TARGET_FILE ${OMNI_RTOS_CFG_DIR}/RTE_Components.h)
omni_add_configure_file(${RTE_CONPONENTS_FILE} ${RTE_CONPONENTS_TARGET_FILE})

add_library(omni-cmsis-rtx INTERFACE)

file(GLOB OMNI_CMSIS_RTX_SOURCES
    "CMSIS/RTOS2/Source/os_systick.c"
    "Source/GCC/irq_armv7m.S"
    "Source/rtx_delay.c"
    "Source/rtx_evflags.c"
    "Source/rtx_evr.c"
    "Source/rtx_kernel.c"
    "Source/rtx_lib.c"
    "Source/rtx_memory.c"
    "Source/rtx_mempool.c"
    "Source/rtx_msgqueue.c"
    "Source/rtx_mutex.c"
    "Source/rtx_semaphore.c"
    "Source/rtx_system.c"
    "Source/rtx_thread.c"
    "Source/rtx_timer.c"
)

target_compile_definitions(omni-cmsis-rtx INTERFACE
    -D_RTE_
)

target_sources(omni-cmsis-rtx INTERFACE
    ${OMNI_CMSIS_RTX_SOURCES}
    $<$<BOOL:${CONFIG_RTOS_CMSIS_RTX}>:${RTX_CFG_SRC_TARGET_FILE}>
)

target_include_directories(omni-cmsis-rtx INTERFACE
    CMSIS/RTOS2/Include
    Include
    Source
)
