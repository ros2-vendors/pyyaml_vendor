cmake_minimum_required(VERSION 3.7)

project(pyyaml_vendor)

find_package(ament_cmake REQUIRED)

# PyYAML (YAML Python module) with its libyaml C extension (_yaml). Builds under
# colcon as a *_vendor; libyaml comes from ros2's libyaml_vendor. Pinned to 6.0.3
# (Ubuntu 26.04 python3-yaml).
#
# Works BOTH natively and cross-compiled:
#   * native  (CMAKE_CROSSCOMPILING=OFF): plain `setup.py`, system Python/compiler.
#   * cross   (CMAKE_CROSSCOMPILING=ON) : inject the cross-Python env (compiler from
#     the toolchain, _PYTHON_HOST_PLATFORM + sysconfigdata, per target OS).
# Nothing is hardcoded to VxWorks or a specific arch; add an elseif() for a new OS.

set(PYYAML_VERSION 6.0.3)

set(PREFIX "${CMAKE_INSTALL_PREFIX}")

if(CMAKE_CROSSCOMPILING)
  # ---- cross build: assemble the target-specific CPython env ----
  # Python minor must come from the harness env (no guessing).
  set(PYMIN "$ENV{TGT_PYTHON_MINOR}")
  if(NOT PYMIN)
    message(FATAL_ERROR "pyyaml_vendor: cross build but TGT_PYTHON_MINOR not set in the env")
  endif()
  set(PYTAG "python3.${PYMIN}")

  if(CMAKE_C_COMPILER)
    set(CROSS_CC "${CMAKE_C_COMPILER}")
  elseif(DEFINED ENV{CC})
    set(CROSS_CC "$ENV{CC}")
  else()
    message(FATAL_ERROR "pyyaml_vendor: cross build but no CMAKE_C_COMPILER / CC")
  endif()

  # Derive the target arch from the cross compiler's own target triple
  # (self-contained — does NOT rely on a harness-provided var). arm64 -> aarch64.
  execute_process(
    COMMAND ${CROSS_CC} -print-target-triple -c ${CMAKE_CURRENT_SOURCE_DIR}/package.xml
    OUTPUT_VARIABLE _triple OUTPUT_STRIP_TRAILING_WHITESPACE ERROR_QUIET)
  if(NOT _triple)
    message(FATAL_ERROR "pyyaml_vendor: '${CROSS_CC} -print-target-triple' returned nothing")
  endif()
  string(REGEX REPLACE "-.*$" "" PYYAML_TGT_ARCH "${_triple}")
  string(REPLACE "arm64" "aarch64" PYYAML_TGT_ARCH "${PYYAML_TGT_ARCH}")

  # per-OS CPython cross settings (branch on the toolchain system name): each OS
  # sets its host-platform, sysconfigdata name, AND where its SDK's cross CPython
  # headers/libs live (DEV). These are inherently OS/SDK-specific.
  if(CMAKE_SYSTEM_NAME STREQUAL "VxWorks")
    set(PY_HOST_PLATFORM "vxworks-${PYYAML_TGT_ARCH}")
    set(PY_SYSCONFIG     "_sysconfigdata__vxworks_vxworks")
    if(NOT DEFINED ENV{WIND_CC_SYSROOT})
      message(FATAL_ERROR "pyyaml_vendor: VxWorks cross build but WIND_CC_SYSROOT not set")
    endif()
    set(DEV "$ENV{WIND_CC_SYSROOT}/usr/3pp/develop/usr")
  else()
    message(FATAL_ERROR
      "pyyaml_vendor: unsupported cross target OS '${CMAKE_SYSTEM_NAME}'. "
      "Add an elseif() with its _PYTHON_HOST_PLATFORM, sysconfigdata name, and DEV path.")
  endif()

  message(STATUS "pyyaml_vendor: CROSS os=${CMAKE_SYSTEM_NAME} arch=${PYYAML_TGT_ARCH} "
                 "host_platform=${PY_HOST_PLATFORM}")

  set(PYYAML_BUILD_ENV
      PYTHON=${PYTAG}
      "CC=${CROSS_CC}"
      "LDSHARED=${CROSS_CC} -shared"
      "CFLAGS=-I ${DEV}/include/${PYTAG} -I ${PREFIX}/include/libyaml_vendor"
      "LDFLAGS=-L ${DEV}/lib -lpython3 -L ${PREFIX}/lib -lyaml"
      _PYTHON_HOST_PLATFORM=${PY_HOST_PLATFORM}
      _PYTHON_SYSCONFIGDATA_NAME=${PY_SYSCONFIG}
      "PYTHONPATH=${DEV}/lib/${PYTAG}/:${DEV}/lib/${PYTAG}/site-packages")
  set(PYYAML_LIB_SUBDIR "build/lib.${PY_HOST_PLATFORM}-${PYMIN}")
else()
  # ---- native build: system Python + compiler, no cross env ----
  message(STATUS "pyyaml_vendor: NATIVE build")
  find_package(Python3 REQUIRED COMPONENTS Interpreter)
  set(PYTAG "python3.${Python3_VERSION_MINOR}")
  set(PYYAML_BUILD_ENV "")                 # inherit the host environment
  set(PYYAML_LIB_SUBDIR "")                # native setup.py already puts .so under lib/
endif()

include(ExternalProject)

externalproject_add(pyyaml_external
  GIT_REPOSITORY https://github.com/yaml/pyyaml.git
  GIT_TAG ${PYYAML_VERSION}
  GIT_CONFIG advice.detachedHead=false
  UPDATE_COMMAND ""
  TIMEOUT 600
  CONFIGURE_COMMAND ""
  BUILD_IN_SOURCE TRUE
  BUILD_COMMAND ${CMAKE_COMMAND} -E env ${PYYAML_BUILD_ENV}
      ${PYTAG} setup.py --with-libyaml bdist
  INSTALL_COMMAND ${CMAKE_COMMAND} -E make_directory ${PREFIX}/lib/${PYTAG}/site-packages
    COMMAND bash -c "cp -r <SOURCE_DIR>/lib/* ${PREFIX}/lib/${PYTAG}/site-packages/ ; \
                     if [ -n '${PYYAML_LIB_SUBDIR}' ]; then \
                       cp -r <SOURCE_DIR>/${PYYAML_LIB_SUBDIR}/yaml/*.so \
                             ${PREFIX}/lib/${PYTAG}/site-packages/yaml/ 2>/dev/null || true ; \
                     fi"
)

if(BUILD_TESTING)
  find_package(ament_lint_auto REQUIRED)
  ament_lint_auto_find_test_dependencies()
endif()

ament_package()
