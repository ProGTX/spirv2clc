# Copyright 2020-2022 The spirv2clc authors.
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

add_library(libspirv2clc translator.cpp)

set_property(TARGET libspirv2clc PROPERTY POSITION_INDEPENDENT_CODE 1)

target_include_directories(libspirv2clc
  PUBLIC
    ${PROJECT_SOURCE_DIR}/include
    ${CMAKE_CURRENT_BINARY_DIR} # For the export header
  PRIVATE
    ${OPENCL_HEADERS_DIR}
    ${SPIRV-Headers_SOURCE_DIR}/include
    ${SPIRV_TOOLS_DIR}
    ${SPIRV_TOOLS_DIR}/source
    ${SPIRV_TOOLS_BINARY_DIR})

set_target_properties(libspirv2clc PROPERTIES
    CXX_VISIBILITY_PRESET hidden
    VISIBILITY_INLINES_HIDDEN ON
)

target_link_libraries(libspirv2clc SPIRV-Tools-opt)

generate_export_header(libspirv2clc)

if(NOT ${SPIRV2CLC_FULLY_STATIC})
    install(TARGETS libspirv2clc
        RUNTIME DESTINATION lib COMPONENT libraries
        LIBRARY DESTINATION lib COMPONENT libraries
        ARCHIVE DESTINATION lib COMPONENT libraries
    )
    install(TARGETS ${spirv_tools_target}
        RUNTIME DESTINATION lib COMPONENT libraries
        LIBRARY DESTINATION lib COMPONENT libraries
        ARCHIVE DESTINATION lib COMPONENT libraries
    )
    install(TARGETS SPIRV-Tools-opt
        RUNTIME DESTINATION lib COMPONENT libraries
        LIBRARY DESTINATION lib COMPONENT libraries
        ARCHIVE DESTINATION lib COMPONENT libraries
    )
else()
    set(tmp_dir "${CMAKE_CURRENT_BINARY_DIR}/spirv_deps")
    add_custom_command(
        OUTPUT ${tmp_dir}/index.tmp
        COMMAND ${CMAKE_COMMAND} -E make_directory ${tmp_dir}
        COMMAND ${CMAKE_COMMAND} -E make_directory ${tmp_dir}/libspirv2clc
        COMMAND ${CMAKE_COMMAND} -E make_directory ${tmp_dir}/SPIRV-Tools
        COMMAND ${CMAKE_COMMAND} -E make_directory ${tmp_dir}/SPIRV-Tools-opt
        COMMAND ${CMAKE_COMMAND} -E touch ${tmp_dir}/index.tmp
    )
    set(libspirv2clc_fully_static_path
        "${tmp_dir}/${CMAKE_STATIC_LIBRARY_PREFIX}libspirv2clc${CMAKE_STATIC_LIBRARY_SUFFIX}"
    )
    if(MSVC)
        add_custom_command(
            OUTPUT ${libspirv2clc_fully_static_path}
            # Join multiple libraries into a single one
            COMMAND ${CMAKE_AR} /OUT:${libspirv2clc_fully_static_path} $<TARGET_FILE:libspirv2clc> $<TARGET_FILE:SPIRV-Tools> $<TARGET_FILE:SPIRV-Tools-opt>
            WORKING_DIRECTORY ${tmp_dir}
            DEPENDS ${tmp_dir}/index.tmp
        )
    else()
        add_custom_command(
            OUTPUT ${libspirv2clc_fully_static_path}
            # Extract external libraries in their own folders
            COMMAND cd libspirv2clc && ${CMAKE_AR} -x $<TARGET_FILE:libspirv2clc> && cd ..
            COMMAND cd SPIRV-Tools && ${CMAKE_AR} -x $<TARGET_FILE:SPIRV-Tools> && cd ..
            COMMAND cd SPIRV-Tools-opt && ${CMAKE_AR} -x $<TARGET_FILE:SPIRV-Tools-opt> && cd ..
            # Join all extracted object files into a single archive
            COMMAND ${CMAKE_AR} -qcs ${libspirv2clc_fully_static_path} libspirv2clc/*.o SPIRV-Tools/*.o SPIRV-Tools-opt/*.o
            WORKING_DIRECTORY ${tmp_dir}
            DEPENDS ${tmp_dir}/index.tmp
        )
    endif()
    add_custom_target(libspirv2clc_fully_static ALL
        DEPENDS ${libspirv2clc_fully_static_path}
    )
    install(FILES ${libspirv2clc_fully_static_path}
        TYPE LIB
        COMPONENT libraries
    )
endif()

install(FILES
        "${PROJECT_SOURCE_DIR}/include/spirv2clc.h"
        "${CMAKE_CURRENT_BINARY_DIR}/libspirv2clc_export.h"
    DESTINATION include
    COMPONENT headers
)
install(DIRECTORY "${SPIRV_TOOLS_DIR}/include"
    DESTINATION .
    COMPONENT headers
    FILES_MATCHING
        PATTERN "spirv-tools/libspirv.h"
)
install(DIRECTORY "${SPIRV-Headers_SOURCE_DIR}/include"
    DESTINATION .
    COMPONENT headers
    FILES_MATCHING
        PATTERN "spirv/unified1/spirv.h"
)
