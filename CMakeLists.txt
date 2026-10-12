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
#     the toolchain, _PYTHON_HOST_PLATFORM + sysconfigdata selected per target OS).
# Nothing is hardcoded to VxWorks or a specific arch; add an elseif() for a new OS.

set(PYYAML_VERSION 6.0.3)

if(DEFINED ENV{TGT_PYTHON_MINOR})
  set(PYMIN "$ENV{TGT_PYTHON_MINOR}")
else()
  set(PYMIN "9")
endif()
set(PYTAG "python3.${PYMIN}")
set(PREFIX "${CMAKE_INSTALL_PREFIX}")

if(CMAKE_CROSSCOMPILING)
  # ---- cross build: assemble the target-specific CPython env ----
  if(CMAKE_C_COMPILER)
    set(CROSS_CC "${CMAKE_C_COMPILER}")
  elseif(DEFINED ENV{CC})
    set(CROSS_CC "$ENV{CC}")
  else()
    message(FATAL_ERROR "pyyaml_vendor: cross build but no CMAKE_C_COMPILER / CC")
  endif()

  # arch from env (harness) or the compiler's target triple (arm64 -> aarch64)
  set(TGT_ARCH "$ENV{TGT_ARCH}")
  if(NOT TGT_ARCH)
    execute_process(
      COMMAND ${CROSS_CC} -print-target-triple -c ${CMAKE_CURRENT_SOURCE_DIR}/package.xml
      OUTPUT_VARIABLE _triple OUTPUT_STRIP_TRAILING_WHITESPACE ERROR_QUIET)
    string(REGEX REPLACE "-.*$" "" TGT_ARCH "${_triple}")
    string(REPLACE "arm64" "aarch64" TGT_ARCH "${TGT_ARCH}")
  endif()

  # per-OS CPython cross naming (branch on the toolchain system name)
  if(CMAKE_SYSTEM_NAME STREQUAL "VxWorks")
    set(PY_HOST_PLATFORM "vxworks-${TGT_ARCH}")
    set(PY_SYSCONFIG     "_sysconfigdata__vxworks_vxworks")
  elseif(CMAKE_SYSTEM_NAME STREQUAL "QNX")
    set(PY_HOST_PLATFORM "qnx-${TGT_ARCH}")
    set(PY_SYSCONFIG     "_sysconfigdata__qnx_qnx")
  else()
    message(FATAL_ERROR
      "pyyaml_vendor: unsupported cross target OS '${CMAKE_SYSTEM_NAME}'. "
      "Add an elseif() with its _PYTHON_HOST_PLATFORM + sysconfigdata name.")
  endif()

  set(SYSROOT "$ENV{WIND_CC_SYSROOT}")
  if(NOT SYSROOT)
    message(FATAL_ERROR "pyyaml_vendor: cross build but WIND_CC_SYSROOT not set")
  endif()
  set(DEV "${SYSROOT}/usr/3pp/develop/usr")

  message(STATUS "pyyaml_vendor: CROSS os=${CMAKE_SYSTEM_NAME} arch=${TGT_ARCH} "
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
