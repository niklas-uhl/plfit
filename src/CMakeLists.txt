configure_file(${CMAKE_CURRENT_SOURCE_DIR}/config.h.in ${CMAKE_CURRENT_BINARY_DIR}/config.h)

include(GNUInstallDirs)

if(WIN32)
  set(PKGCONFIG_LIBS_PRIVATE "")
else()
  set(PKGCONFIG_LIBS_PRIVATE "-lm")
endif()

set(PLFIT_CORE_SRCS error.c gss.c kolmogorov.c lbfgs.c mt.c plfit.c options.c rbinom.c sampling.c stats.c hzeta.c)

add_library(plfit ${PLFIT_CORE_SRCS})
target_include_directories(
  plfit PUBLIC
  ${CMAKE_CURRENT_BINARY_DIR}
  ${PROJECT_BINARY_DIR}/include
  ${CMAKE_CURRENT_SOURCE_DIR}
  ${PROJECT_SOURCE_DIR}/include
)
set_target_properties(plfit PROPERTIES SOVERSION 0)
target_link_libraries(plfit PRIVATE ${MATH_LIBRARY})

target_compile_definitions(plfit PRIVATE HAVE_CONFIG_H)
if(PLFIT_USE_SSE)
  target_compile_definitions(plfit PRIVATE USE_SSE)
endif()
if(PLFIT_USE_OPENMP AND OPENMP_FOUND)
    target_link_libraries(plfit PUBLIC OpenMP::OpenMP_C)
endif()

# if(CMAKE_C_COMPILER_ID STREQUAL "GNU" AND NOT APPLE)
#   set_target_properties(plfit PROPERTIES LINK_FLAGS "-Wl,--version-script=${PROJECT_SOURCE_DIR}/src/plfit.map")
#   set_target_properties(plfit PROPERTIES LINK_DEPENDS ${PROJECT_SOURCE_DIR}/src/plfit.map)
# endif()

# Generate export header
include(GenerateExportHeader)
generate_export_header(plfit
  STATIC_DEFINE PLFIT_STATIC
  EXPORT_FILE_NAME ${PROJECT_BINARY_DIR}/include/plfit_export.h
)

join_paths(libdir_for_pc_file "\${exec_prefix}" "${CMAKE_INSTALL_LIBDIR}")
join_paths(includedir_for_pc_file "\${prefix}" "${CMAKE_INSTALL_INCLUDEDIR}")
configure_file(${PROJECT_SOURCE_DIR}/libplfit.pc.in ${PROJECT_BINARY_DIR}/libplfit.pc @ONLY)

install(TARGETS plfit LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR})
install(
    FILES
    ${PROJECT_SOURCE_DIR}/include/plfit.h
    ${PROJECT_SOURCE_DIR}/include/plfit_decls.h
    ${PROJECT_SOURCE_DIR}/include/plfit_error.h
    ${PROJECT_BINARY_DIR}/include/plfit_export.h
    ${PROJECT_SOURCE_DIR}/include/plfit_mt.h
    ${PROJECT_SOURCE_DIR}/include/plfit_sampling.h
    ${PROJECT_BINARY_DIR}/include/plfit_version.h
    DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}/plfit
)
install(
    FILES ${PROJECT_BINARY_DIR}/libplfit.pc
    DESTINATION ${CMAKE_INSTALL_LIBDIR}/pkgconfig
)

add_executable(plfit-bin main.c getopt.c)
target_link_libraries(plfit-bin plfit ${MATH_LIBRARY})
set_target_properties(plfit-bin PROPERTIES OUTPUT_NAME plfit)
install(TARGETS plfit-bin RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR})

add_executable(plgen plgen.c getopt.c)
target_link_libraries(plgen plfit ${MATH_LIBRARY})
install(TARGETS plgen RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR})

if(PLFIT_COMPILE_PYTHON_MODULE)
    # Check whether we have SWIG and the Python libraries installed
    find_package(SWIG REQUIRED)
    include(${SWIG_USE_FILE})

    include(FindPython3)
	find_package(Python3 REQUIRED COMPONENTS Interpreter Development)

    set(CMAKE_SWIG_FLAGS "")
    set_property(SOURCE plfit.i PROPERTY SWIG_MODULE_NAME plfit)
    if (NOT BUILD_SHARED_LIBS)
        swig_add_library(plfit_python LANGUAGE python SOURCES plfit.i ${PLFIT_CORE_SRCS})
        target_link_libraries(plfit_python ${Python3_LIBRARIES} m)
    else()
        swig_add_library(plfit_python LANGUAGE python SOURCES plfit.i)
        target_link_libraries(plfit_python ${Python3_LIBRARIES} plfit m)
    endif()

    target_include_directories(
        plfit_python PUBLIC
        ${CMAKE_CURRENT_BINARY_DIR}
        ${CMAKE_BINARY_DIR}/include
        ${CMAKE_CURRENT_SOURCE_DIR}
        ${CMAKE_SOURCE_DIR}/include
        ${Python3_INCLUDE_DIRS}
    )

    install(TARGETS plfit_python DESTINATION ${Python3_SITEARCH})
	install(FILES ${CMAKE_CURRENT_BINARY_DIR}/plfit.py DESTINATION ${Python3_SITELIB})

endif(PLFIT_COMPILE_PYTHON_MODULE)
