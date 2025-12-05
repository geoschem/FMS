esma_set_this ()
string (REPLACE fms_ "" precision ${this})

# The following logic is to avoid duplication in the supported GNUMake logic.
# Source files (and headers) are listed in a separate file, and we must
# process this into something that CMake can use for assembling a library.


set(srcs_and_includes
  affinity/fms_affinity.F90
  amip_interp/amip_interp.F90
  astronomy/astronomy.F90
  axis_utils/axis_utils.F90
  axis_utils/axis_utils2.F90
  block_control/block_control.F90
  column_diagnostics/column_diagnostics.F90
  constants/constants.F90
  constants/fmsconstants.F90
  coupler/atmos_ocean_fluxes.F90
  coupler/coupler_types.F90
  coupler/ensemble_manager.F90
  data_override/get_grid_version.F90
  data_override/data_override.F90
  diag_integral/diag_integral.F90
  diag_manager/diag_axis.F90
  diag_manager/diag_data.F90
  diag_manager/diag_grid.F90
  diag_manager/diag_manager.F90
  diag_manager/diag_output.F90
  diag_manager/diag_table.F90
  diag_manager/diag_util.F90
  drifters/cloud_interpolator.F90
  drifters/drifters.F90
  drifters/drifters_comm.F90
  drifters/drifters_core.F90
  drifters/drifters_input.F90
  drifters/drifters_io.F90
  drifters/quicksort.F90
  exchange/stock_constants.F90
  exchange/xgrid.F90
  field_manager/field_manager.F90
  field_manager/fm_util.F90
  fms/fms_io.F90
  fms/fms.F90
  fms2_io/blackboxio.F90
  fms2_io/fms_io_utils.F90
  fms2_io/fms_netcdf_domain_io.F90
  fms2_io/fms_netcdf_unstructured_domain_io.F90
  fms2_io/fms2_io.F90
  fms2_io/netcdf_io.F90
  horiz_interp/horiz_interp_bicubic.F90
  horiz_interp/horiz_interp_bilinear.F90
  horiz_interp/horiz_interp_conserve.F90
  horiz_interp/horiz_interp_spherical.F90
  horiz_interp/horiz_interp_type.F90
  horiz_interp/horiz_interp.F90
  interpolator/interpolator.F90
  memutils/memutils.F90
  monin_obukhov/monin_obukhov_inter.F90
  monin_obukhov/monin_obukhov.F90
  mosaic/gradient.F90
  mosaic/grid.F90
  mosaic/mosaic.F90
  mosaic2/grid2.F90
  mosaic2/mosaic2.F90
  mpp/mpp.F90
  mpp/mpp_data.F90
  mpp/mpp_domains.F90
  mpp/mpp_efp.F90
  mpp/mpp_io.F90
  mpp/mpp_memutils.F90
  mpp/mpp_parameter.F90
  mpp/mpp_utilities.F90
  parser/yaml_parser.F90
  platform/platform.F90
  random_numbers/mersennetwister.F90
  random_numbers/random_numbers.F90
  sat_vapor_pres/sat_vapor_pres_k.F90
  sat_vapor_pres/sat_vapor_pres.F90
  string_utils/fms_string_utils.F90
  time_interp/time_interp_external.F90
  time_interp/time_interp_external2.F90
  time_interp/time_interp.F90
  time_manager/get_cal_time.F90
  time_manager/time_manager.F90
  topography/gaussian_topog.F90
  topography/topography.F90
  tracer_manager/tracer_manager.F90
  tridiagonal/tridiagonal.F90
  libFMS.F90
  affinity/affinity.c
  mosaic/create_xgrid.c
  mosaic/gradient_c2l.c
  mosaic/interp.c
  mosaic/mosaic_util.c
  mosaic/read_mosaic.c
  mpp/mpp_memuse.c
  parser/yaml_parser_binding.c
  string_utils/fms_string_utils_binding.c
  include/file_version.h
  include/fms_platform.h
)


# Filter out header files
set (srcs)
foreach (file ${srcs_and_includes})
  string (REGEX MATCH ".+\\.h" header ${file})
  string (REGEX MATCH ".+\\.inc" inc ${file})
  if (header OR inc)
  else ()
    list (APPEND srcs ${file})
  endif ()
endforeach ()

set (lib fms_${precision})

# GCHP: do not use shared library
#esma_add_library (${lib} SRCS ${srcs} DEPENDENCIES NetCDF::NetCDF_Fortran MPI::MPI_Fortran TYPE SHARED)
esma_add_library (${lib} SRCS ${srcs} DEPENDENCIES NetCDF::NetCDF_Fortran MPI::MPI_Fortran)


target_include_directories (${lib} PUBLIC
  $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>
  $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/mpp/include>
  $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/fms>
  $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/fms2_io/include>
  $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/constants>
)

file (COPY include/fms_platform.h DESTINATION ${include_${this}})
file (COPY include/file_version.h DESTINATION ${include_${this}})

# Various compiler options and CPP definitions ...
target_compile_definitions (${lib} PRIVATE OVERLOAD_R4 MAPL_MODE SPMD use_libMPI use_netCDF PORTABLE_KINDS HAVE_GETTID INTERNAL_FILE_NML)
if (precision MATCHES "r4")
   target_compile_definitions (${lib} PRIVATE OVERLOAD_R8)
endif ()

if (APPLE)
   target_compile_definitions(${lib} PRIVATE __APPLE__)
endif ()

set (CMAKE_Fortran_FLAGS_RELEASE "${GEOS_Fortran_FLAGS_VECT}")

# This 'resets' the Intel DEBUG flags for FMS. The stock debug flags use
# 'all,noarg_temp_created' which seem to be too aggressive for FMS/MOM6. This
# moves them back to the 'bounds,uninit' GEOS used to build with.
if (CMAKE_Fortran_COMPILER_ID MATCHES "Intel" AND CMAKE_BUILD_TYPE MATCHES Debug)
   string(REPLACE "all,noarg_temp_created" "bounds,uninit" _tmp "${GEOS_Fortran_FLAGS_DEBUG}")
   set (CMAKE_Fortran_FLAGS_DEBUG "${_tmp}")
endif ()

if (CMAKE_Fortran_COMPILER_ID MATCHES "NAG")
  target_compile_options (${lib} PRIVATE $<$<COMPILE_LANGUAGE:Fortran>:-kind=byte>)
endif ()
if (CMAKE_Fortran_COMPILER_ID MATCHES "GNU")
  target_compile_options (${lib} PRIVATE $<$<COMPILE_LANGUAGE:Fortran>:-fcray-pointer>)
endif ()
if (CMAKE_Fortran_COMPILER_ID MATCHES "Intel")
  target_compile_options (${lib} PRIVATE $<$<COMPILE_LANGUAGE:Fortran>:-diag-disable 6843,7712>) # warnings about dummy args
  target_compile_options (${lib} PRIVATE $<$<COMPILE_LANGUAGE:Fortran>:-no-heap-arrays>) # mom issue with Intel 19
endif ()
if (CMAKE_Fortran_COMPILER_ID MATCHES "PGI")
  target_compile_definitions (${lib} PRIVATE NO_QUAD_PRECISION)
endif ()

if (precision MATCHES "r8")
  esma_fortran_generator_list(${lib} ${FREAL8})
endif ()






