cmake_minimum_required(VERSION 3.7)

project(pyyaml_vendor)

find_package(ament_cmake REQUIRED)

# PyYAML (YAML Python module) with its libyaml C extension (_yaml). Builds under
# colcon as a *_vendor; libyaml comes from ros2's libyaml_vendor. Pinned to 6.0.3
# (Ubuntu 26.04 python3-yaml). Builds natively and cross-compiled.

set(PYYAML_VERSION 6.0.3)
set(PREFIX "${CMAKE_INSTALL_PREFIX}")

# Resolve Python the proper CMake way. In a cross build the toolchain makes
# find_package(Python3) return the HOST interpreter + the TARGET headers/library;
# natively it returns the host for both.
find_package(Python3 REQUIRED COMPONENTS Interpreter Development)
set(PYVER "${Python3_VERSION_MAJOR}.${Python3_VERSION_MINOR}")   # e.g. 3.9
set(PYTAG "python${PYVER}")
set(PY_DRIVER "${Python3_EXECUTABLE}")                           # runs setup.py (host interp)
list(GET Python3_INCLUDE_DIRS 0 PY_INCLUDE)                      # target CPython headers

if(CMAKE_CROSSCOMPILING)
  # Cross: setup.py needs the target "host platform" + sysconfigdata so it emits a
  # cross build tree and links the target CPython. Both are derived from the
  # standard target OS/arch CMake variables; the CPython prefix from the include dir.
  string(TOLOWER "${CMAKE_SYSTEM_NAME}" _os)                     # e.g. the lowercased OS
  set(PY_HOST_PLATFORM "${_os}-${CMAKE_SYSTEM_PROCESSOR}")       # <os>-<arch>
  set(PY_SYSCONFIG "_sysconfigdata__${_os}_${_os}")             # _sysconfigdata__<os>_<os>
  get_filename_component(DEV "${PY_INCLUDE}" DIRECTORY)          # <prefix>/include
  get_filename_component(DEV "${DEV}" DIRECTORY)                # <prefix>

  message(STATUS "pyyaml_vendor: CROSS driver=${PY_DRIVER} py=${PYTAG} "
                 "host_platform=${PY_HOST_PLATFORM}")

  set(PYYAML_BUILD_ENV
      PYTHON=${PY_DRIVER}
      "CC=${CMAKE_C_COMPILER}"
      "LDSHARED=${CMAKE_C_COMPILER} -shared"
      "CFLAGS=-I ${PY_INCLUDE} -I ${PREFIX}/include/libyaml_vendor"
      "LDFLAGS=-L ${DEV}/lib -lpython3 -L ${PREFIX}/lib -lyaml"
      _PYTHON_HOST_PLATFORM=${PY_HOST_PLATFORM}
      _PYTHON_SYSCONFIGDATA_NAME=${PY_SYSCONFIG}
      "PYTHONPATH=${DEV}/lib/${PYTAG}/:${DEV}/lib/${PYTAG}/site-packages")
  # setup.py build dir is lib.<host-platform>-<pyversion> (e.g. lib.<os>-<arch>-<pyver>).
  set(PYYAML_LIB_SUBDIR "build/lib.${PY_HOST_PLATFORM}-${PYVER}")
else()
  message(STATUS "pyyaml_vendor: NATIVE build py=${PYTAG}")
  set(PYYAML_BUILD_ENV "")                 # host environment
  set(PYYAML_LIB_SUBDIR "build/lib.*")     # native setup.py build dir
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
      ${PY_DRIVER} setup.py --with-libyaml build
  # Copy the whole built tree (yaml/ pure-python package + the _yaml C extension)
  # from setup.py's build/lib.* dir into the target site-packages.
  INSTALL_COMMAND ${CMAKE_COMMAND} -E make_directory ${PREFIX}/lib/${PYTAG}/site-packages
    COMMAND bash -c "cp -r <SOURCE_DIR>/${PYYAML_LIB_SUBDIR}/* ${PREFIX}/lib/${PYTAG}/site-packages/"
)

if(BUILD_TESTING)
  find_package(ament_lint_auto REQUIRED)
  ament_lint_auto_find_test_dependencies()
endif()

ament_package()
