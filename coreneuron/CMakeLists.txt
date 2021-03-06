# =============================================================================
# Copyright (C) 2016-2020 Blue Brain Project
#
# See top-level LICENSE file for details.
# =============================================================================

# commonly included directories
include_directories(utils/randoms ${MPI_INCLUDE_PATH} ${PROJECT_BINARY_DIR}/coreneuron
                    ${CMAKE_BINARY_DIR}/coreneuron ${CMAKE_BINARY_DIR}/include)

# put libraries (e.g. dll) in bin directory
set(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/bin)

# =============================================================================
# gather various source files
# =============================================================================
file(GLOB_RECURSE CORENEURON_HEADER_FILES "*.h*")
file(GLOB_RECURSE CORENEURON_TEMPLATE_FILES "*.ipp")
file(
  GLOB
  CORENEURON_CODE_FILES
  "apps/main1.cpp"
  "apps/corenrn_parameters.cpp"
  "gpu/*.cpp"
  "io/*.cpp"
  "io/reports/*.cpp"
  "mechanism/*.cpp"
  "mpi/*.cpp"
  "network/*.cpp"
  "permute/*.cpp"
  "sim/*.cpp")
file(GLOB_RECURSE CORENEURON_UTILS_FILES "utils/*.c*")
list(APPEND CORENEURON_CODE_FILES ${CORENEURON_UTILS_FILES})
file(GLOB_RECURSE CORENEURON_CUDA_FILES "*.cu")
file(GLOB SCOPMATH_CODE_FILES "sim/scopmath/*.cpp")
file(COPY ${CORENEURON_PROJECT_SOURCE_DIR}/external/Random123/include/Random123
     DESTINATION ${CMAKE_BINARY_DIR}/include)

set(DIMPLIC_CODE_FILE "mechanism/mech/dimplic.cpp")
set(ENGINEMECH_CODE_FILE "mechanism/mech/enginemech.cpp")

# for external mod files we need to generate modl_ref function in mod_func.c
set(MODFUNC_PERL_SCRIPT "mechanism/mech/mod_func.c.pl")

# ~~~
# as openacc limits passing function pointers as arguments, mod2c translated files
# depend on a _kinderiv.h file that is constructed by kinderiv.py
# ~~~
set(KINDERIV_PYTHON_SCRIPT "${CMAKE_CURRENT_SOURCE_DIR}/kinderiv.py")
set(KINDERIV_HEADER_FILE "${CMAKE_CURRENT_BINARY_DIR}/_kinderiv.h")

set(NMODL_UNITS_FILE "${CMAKE_BINARY_DIR}/share/nrnunits.lib")
set(INBUILT_MOD_PATH "mechanism/mech/modfile")
set(INBUILT_MOD_FILES
    expsyn.mod
    exp2syn.mod
    hh.mod
    netstim.mod
    passive.mod
    pattern.mod
    stim.mod)

# Has helper macro for generating cpp file from mod2c
include(NmodlHelper)

# =============================================================================
# Command for MOD to CPP conversion
# =============================================================================
set(CORENRN_NMODL_COMMAND env "MODLUNIT=${NMODL_UNITS_FILE}"
                          "PYTHONPATH=${CORENRN_NMODL_PYTHONPATH}" ${CORENRN_NMODL_BINARY})

if(${CORENRN_ENABLE_ISPC})
  set(NMODL_CODEGEN_TARGET ispc)
else()
  set(NMODL_CODEGEN_TARGET serial)
endif()

# configure every mod file to configure with mod2c/nmodl
foreach(MOD ${INBUILT_MOD_FILES})
  nmodl_to_cpp_target(KEY INBUILT_MOD MODFILE "${INBUILT_MOD_PATH}/${MOD}" TARGET
                      ${NMODL_CODEGEN_TARGET})
endforeach()

set_source_files_properties(${NMODL_INBUILT_MOD_OUTPUTS} PROPERTIES GENERATED TRUE)

# for ispc generated files, we need add extra compilation flags
if(CORENRN_ENABLE_ISPC)
  set(NMODL_ISPC_INBUILT_MOD_OUTPUTS ${NMODL_INBUILT_MOD_OUTPUTS})
  list(FILTER NMODL_ISPC_INBUILT_MOD_OUTPUTS INCLUDE REGEX ".*\.ispc$")
  set_source_files_properties(${NMODL_ISPC_INBUILT_MOD_OUTPUTS} PROPERTIES LANGUAGE ISPC)
  set_source_files_properties(${NMODL_ISPC_INBUILT_MOD_OUTPUTS} PROPERTIES COMPILE_FLAGS
                              "-I${CORENRN_NMODL_INCLUDE} --pic")
endif()

# =============================================================================
# coreneuron GPU library
# =============================================================================
if(CORENRN_ENABLE_GPU)
  # ~~~
  # artificial cells and some other cpp files (using Random123) should be compiled
  # without OpenACC to avoid use of GPU Random123 streams
  # ~~~
  set(OPENACC_EXCLUDED_FILES
      ${CMAKE_CURRENT_BINARY_DIR}/netstim.cpp
      ${CMAKE_CURRENT_BINARY_DIR}/netstim_inhpoisson.cpp
      ${CMAKE_CURRENT_BINARY_DIR}/pattern.cpp
      ${CMAKE_CURRENT_SOURCE_DIR}/utils/randoms/nrnran123.cpp
      ${CMAKE_CURRENT_SOURCE_DIR}/io/nrn_setup.cpp
      ${CMAKE_CURRENT_SOURCE_DIR}/io/setup_fornetcon.cpp
      ${CMAKE_CURRENT_SOURCE_DIR}/io/corenrn_data_return.cpp
      ${CMAKE_CURRENT_SOURCE_DIR}/io/global_vars.cpp)

  set_source_files_properties(${DIMPLIC_CODE_FILE} ${NMODL_INBUILT_MOD_OUTPUTS} PROPERTIES
                              COMPILE_FLAGS "")

  set_source_files_properties(${OPENACC_EXCLUDED_FILES} PROPERTIES COMPILE_FLAGS
                              "-DDISABLE_OPENACC")

  # TODO : only older PGI versions?
  if(${CMAKE_C_COMPILER_ID} STREQUAL "PGI")
    set_source_files_properties(${CMAKE_CURRENT_SOURCE_DIR}/scopmath_core/sparse_thread.c
                                PROPERTIES COMPILE_FLAGS "-ta=tesla:nollvm")
  endif()

  # compile cuda files for multiple architecture
  cuda_add_library(
    "cudacoreneuron" ${CORENEURON_CUDA_FILES}
    OPTIONS -arch=sm_30
            -gencode=arch=compute_30,code=sm_30
            -gencode=arch=compute_50,code=sm_50
            -gencode=arch=compute_52,code=sm_52
            -gencode=arch=compute_52,code=compute_52
            -Xcompiler
            -fPIC)
  set(link_cudacoreneuron cudacoreneuron)
else()
  set(link_cudacoreneuron "")
endif()

# =============================================================================
# run KINDERIV_PYTHON_SCRIPT to generate _kinderiv.h
# =============================================================================
add_custom_command(
  OUTPUT "${KINDERIV_HEADER_FILE}"
  DEPENDS ${NMODL_INBUILT_MOD_OUTPUTS} "${KINDERIV_PYTHON_SCRIPT}"
  COMMAND ${PYTHON_EXECUTABLE} ${KINDERIV_PYTHON_SCRIPT} ${CMAKE_CURRENT_BINARY_DIR}
  WORKING_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}
  COMMENT "Generating kinderiv.h by inspecting MOD files")
add_custom_target(kin_deriv_header DEPENDS "${KINDERIV_HEADER_FILE}")

# =============================================================================
# create libraries
# =============================================================================

# main coreneuron library
add_library(
  coreneuron ${COMPILE_LIBRARY_TYPE} ${CORENEURON_HEADER_FILES} ${CORENEURON_TEMPLATE_FILES}
             ${CORENEURON_CODE_FILES} ${cudacorenrn_objs} ${NMODL_INBUILT_MOD_OUTPUTS})

# need to have _kinderiv.h for mod2c generated files
add_dependencies(coreneuron kin_deriv_header)

# scopmath is created separately for nrnivmodl-core workflow
add_library(scopmath STATIC ${CORENEURON_HEADER_FILES} ${SCOPMATH_CODE_FILES})

# minimal main and dimplic file for nrnivmodl-core workflow
add_library(corenrnmech ${COMPILE_LIBRARY_TYPE} ${CORENEURON_HEADER_FILES} ${ENGINEMECH_CODE_FILE}
                        ${DIMPLIC_CODE_FILE})

target_link_libraries(
  coreneuron
  ${MPI_C_LIBRARIES}
  ${reportinglib_LIBRARY}
  ${sonatareport_LIBRARY}
  ${link_cudacoreneuron}
  ${CUDA_LIBRARIES}
  ${CALIPER_LIB}
  ${CALIPER_MPI_LIB}
  ${likwid_LIBRARIES})
target_include_directories(coreneuron SYSTEM
                           PRIVATE ${CORENEURON_PROJECT_SOURCE_DIR}/external/Random123/include)
target_include_directories(coreneuron SYSTEM
                           PRIVATE ${CORENEURON_PROJECT_SOURCE_DIR}/external/CLI11/include)

target_link_libraries(corenrnmech scopmath coreneuron)

set_target_properties(
  coreneuron corenrnmech scopmath ${link_cudacoreneuron}
  PROPERTIES ARCHIVE_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/lib LIBRARY_OUTPUT_DIRECTORY
             ${CMAKE_BINARY_DIR}/lib POSITION_INDEPENDENT_CODE ON)

# =============================================================================
# create special-core with halfgap.mod for tests
# =============================================================================
add_custom_command(
  TARGET coreneuron POST_BUILD
  COMMAND ${CMAKE_BINARY_DIR}/bin/nrnivmodl-core
          ${CORENEURON_PROJECT_SOURCE_DIR}/tests/integration/ring_gap/mod
  WORKING_DIRECTORY ${CMAKE_BINARY_DIR}/bin
  BYPRODUCTS ${CMAKE_BINARY_DIR}/bin/${CMAKE_SYSTEM_PROCESSOR}/special-core
  COMMENT "Running nrnivmodl-core with halfgap.mod")
add_dependencies(coreneuron scopmath)

# =============================================================================
# create nrnivmodl-core
# =============================================================================
# coreneuron executable
include_directories(${CORENEURON_PROJECT_SOURCE_DIR})
add_executable(nrniv-core "apps/coreneuron.cpp")

target_link_libraries(nrniv-core corenrnmech ${reportinglib_LIBRARY} ${sonatareport_LIBRARY})

set_target_properties(nrniv-core PROPERTIES RUNTIME_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/bin)

# =============================================================================
# Extract link definitions to be used with nrnivmodl-core
# =============================================================================
get_target_property(CORENRN_LINK_LIBS coreneuron LINK_LIBRARIES)
if(NOT CORENRN_LINK_LIBS)
  set(CORENRN_LINK_LIBS "")
endif()
set(CORENRN_LINK_LIBS
    "${CORENRN_LINK_LIBS}"
    PARENT_SCOPE)

# =============================================================================
# Copy files for nrnivmodl-core workflow during build time
# =============================================================================

configure_file(${MODFUNC_PERL_SCRIPT} ${CMAKE_BINARY_DIR}/share/coreneuron/mod_func.c.pl COPYONLY)
configure_file(${KINDERIV_PYTHON_SCRIPT} ${CMAKE_BINARY_DIR}/share/coreneuron/kinderiv.py COPYONLY)
configure_file(${DIMPLIC_CODE_FILE} ${CMAKE_BINARY_DIR}/share/coreneuron/dimplic.cpp COPYONLY)
configure_file(${ENGINEMECH_CODE_FILE} ${CMAKE_BINARY_DIR}/share/coreneuron/enginemech.cpp COPYONLY)

# Make headers avail to build tree
configure_file(engine.h.in ${CMAKE_BINARY_DIR}/include/coreneuron/engine.h @ONLY)

file(
  GLOB_RECURSE main_headers
  RELATIVE "${CMAKE_CURRENT_SOURCE_DIR}"
  *.h *.hpp *.ispc)

foreach(header ${main_headers})
  configure_file("${header}" "${CMAKE_BINARY_DIR}/include/coreneuron/${header}" COPYONLY)
endforeach()

configure_file("utils/profile/profiler_interface.h"
               ${CMAKE_BINARY_DIR}/include/coreneuron/nrniv/profiler_interface.h COPYONLY)

if(NOT CORENRN_ENABLE_NMODL)
  # ~~~
  # MOD2C puts UNITS_FILE in share, but we need to see it in share/mod2c We create
  # a link since we know neither the source location nor when its copied
  # ~~~
  file(MAKE_DIRECTORY "${CMAKE_BINARY_DIR}/share/mod2c/")
  add_custom_target(link_units ALL COMMAND ${CMAKE_COMMAND} -E create_symlink "${NMODL_UNITS_FILE}"
                                           "${CMAKE_BINARY_DIR}/share/mod2c/nrnunits.lib")
endif()

# main program required for building special-core
file(COPY apps/coreneuron.cpp DESTINATION ${CMAKE_BINARY_DIR}/share/coreneuron)

# =============================================================================
# Install main targets
# =============================================================================

# coreneuron main libraries
install(
  TARGETS coreneuron corenrnmech ${link_cudacoreneuron}
  EXPORT coreneuron
  LIBRARY DESTINATION lib
  ARCHIVE DESTINATION lib
  INCLUDES DESTINATION $<INSTALL_INTERFACE:include>)

# scopemath into share for nrnivmodl-core
install(
  TARGETS scopmath
  EXPORT coreneuron
  DESTINATION lib)

# headers and some standalone code files for nrnivmodl-core
install(
  DIRECTORY ${CMAKE_BINARY_DIR}/include/coreneuron
  DESTINATION include/
  FILES_MATCHING
  PATTERN "*.h*"
  PATTERN "*.ipp"
  PATTERN "*.ispc")
install(FILES ${KINDERIV_PYTHON_SCRIPT} ${MODFUNC_PERL_SCRIPT} ${DIMPLIC_CODE_FILE}
              ${ENGINEMECH_CODE_FILE} DESTINATION share/coreneuron)

# copy mod2c/nmodl for nrnivmodl-core
install(PROGRAMS ${CORENRN_NMODL_BINARY} DESTINATION bin)

if(NOT CORENRN_ENABLE_NMODL)
  install(FILES ${NMODL_UNITS_FILE} DESTINATION share/mod2c)
endif()

# install nrniv-core app
install(TARGETS nrniv-core DESTINATION bin)
install(FILES apps/coreneuron.cpp DESTINATION share/coreneuron)

# install random123 and nmodl headers
install(DIRECTORY ${CMAKE_BINARY_DIR}/include/ DESTINATION include)
