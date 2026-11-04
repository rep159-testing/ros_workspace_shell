cmake_minimum_required(VERSION 3.10)
project(ros_workspace_base NONE)

# Generates the same prefix level setup files and environment hooks as
# ament_cmake does, from the copies of the ament_package templates in
# templates/, so that neither building nor using this package needs any
# ament package.

if(WIN32)
  message(FATAL_ERROR "ros_workspace_base supports Linux and macOS only")
endif()

# REP 159: the install prefixes of the distributions this one extends with
# binary_import, nearest first. bloom passes them when releasing into such a
# distribution; a root distribution leaves the list empty.
set(ROS_WORKSPACE_PARENT_PREFIX_PATH "" CACHE STRING
  "Install prefixes of the binary_import parent distributions, nearest first")

find_package(Python3 REQUIRED COMPONENTS Interpreter)
# default interpreter written into the prefix level local_setup files
set(ament_package_PYTHON_EXECUTABLE "${Python3_EXECUTABLE}")
set(PYTHON_INSTALL_DIR
  "lib/python${Python3_VERSION_MAJOR}.${Python3_VERSION_MINOR}/site-packages")

# enable C language so that a trycompile can determine what the
# anticipated libdir will be.
enable_language(C)
include(GNUInstallDirs)

set(templates "${CMAKE_CURRENT_SOURCE_DIR}/templates")
set(gen "${CMAKE_CURRENT_BINARY_DIR}/generated")
set(share "share/${PROJECT_NAME}")

# Copy a file into the build tree, expanding it when it ends in .in, and
# return the path of the result in dest_var.
function(_stage_file src dest_dir dest_var)
  get_filename_component(name "${src}" NAME)
  if(name MATCHES "^(.*)\\.in$")
    set(name "${CMAKE_MATCH_1}")
    configure_file("${src}" "${dest_dir}/${name}" @ONLY)
  else()
    configure_file("${src}" "${dest_dir}/${name}" COPYONLY)
  endif()
  set(${dest_var} "${dest_dir}/${name}" PARENT_SCOPE)
endfunction()

# --- environment hooks for the whole prefix
# Each entry is name|template|dsv type|variable|value.
if(APPLE)
  set(LIBRARY_PATH_ENV_VAR "DYLD_LIBRARY_PATH")
else()
  set(LIBRARY_PATH_ENV_VAR "LD_LIBRARY_PATH")
endif()
set(hooks
  "library_path|${templates}/environment_hook/library_path.sh|prepend-non-duplicate|${LIBRARY_PATH_ENV_VAR}|lib"
  "path|${templates}/environment_hook/path.sh|prepend-non-duplicate-if-exists|PATH|bin"
  "pythonpath|${templates}/environment_hook/pythonpath.sh.in|prepend-non-duplicate|PYTHONPATH|${PYTHON_INSTALL_DIR}")
if(NOT CMAKE_INSTALL_LIBDIR STREQUAL "lib")
  list(APPEND hooks
    "multiarch_library_paths|${CMAKE_CURRENT_SOURCE_DIR}/env-hooks/multiarch_library_paths.sh.in|prepend-non-duplicate|${LIBRARY_PATH_ENV_VAR}|${CMAKE_INSTALL_LIBDIR}")
endif()

set(hook_paths "")
foreach(hook ${hooks})
  string(REPLACE "|" ";" hook "${hook}")
  list(GET hook 0 name)
  list(GET hook 1 src)
  list(SUBLIST hook 2 3 dsv)
  list(JOIN dsv ";" dsv)
  _stage_file("${src}" "${gen}/environment" staged)
  file(WRITE "${gen}/environment/${name}.dsv" "${dsv}\n")
  install(FILES "${staged}" "${gen}/environment/${name}.dsv"
    DESTINATION "${share}/environment")
  list(APPEND hook_paths "${share}/environment/${name}.sh")
endforeach()
list(SORT hook_paths)

# --- package level local_setup files and their dsv files
set(local_setup_dsv "")
set(sh_hooks "")
foreach(hook ${hook_paths})
  string(APPEND local_setup_dsv "source;${hook}\n")
  string(APPEND sh_hooks
    "ament_append_value AMENT_ENVIRONMENT_HOOKS \"$AMENT_CURRENT_PREFIX/${hook}\"\n")
endforeach()
set(extensions bash fish sh zsh)
foreach(ext ${extensions})
  # the hooks are shell scripts, so only the sh file sources them
  if(ext STREQUAL "sh")
    set(ENVIRONMENT_HOOKS "${sh_hooks}")
  else()
    set(ENVIRONMENT_HOOKS "")
  endif()
  _stage_file("${templates}/package_level/local_setup.${ext}.in" "${gen}/package" staged)
  install(FILES "${staged}" DESTINATION "${share}")
endforeach()
file(WRITE "${gen}/package/local_setup.dsv" "${local_setup_dsv}")
list(APPEND extensions dsv)
list(SORT extensions)
set(package_dsv "")
foreach(ext ${extensions})
  string(APPEND package_dsv "source;${share}/local_setup.${ext}\n")
endforeach()
file(WRITE "${gen}/package/package.dsv" "${package_dsv}")
install(FILES
  "${gen}/package/local_setup.dsv"
  "${gen}/package/package.dsv"
  package.xml
  DESTINATION "${share}")

# --- resource index entries read by the prefix level setup files
if(ROS_WORKSPACE_PARENT_PREFIX_PATH)
  # REP 159: record the parents as this package's parent_prefix_path
  # resource, which the prefix level setup files read to source each
  # parent's local_setup first.
  string(REPLACE ";" ":" parent_prefix_path "${ROS_WORKSPACE_PARENT_PREFIX_PATH}")
  set(SKIP_PARENT_PREFIX_PATH "")
else()
  set(parent_prefix_path "")
  # skip using ament_index/resource_index/parent_prefix_path
  # if for Debian packages it is known that there are no underlays
  # note the template only checks for an empty (use) / non-empty (skip) string
  set(SKIP_PARENT_PREFIX_PATH "SKIP_PARENT_PREFIX_PATH")
endif()
set(index "share/ament_index/resource_index")
file(WRITE "${gen}/index/packages/${PROJECT_NAME}" "")
file(WRITE "${gen}/index/package_run_dependencies/${PROJECT_NAME}" "")
file(WRITE "${gen}/index/parent_prefix_path/${PROJECT_NAME}" "${parent_prefix_path}")
foreach(resource packages package_run_dependencies parent_prefix_path)
  install(FILES "${gen}/index/${resource}/${PROJECT_NAME}"
    DESTINATION "${index}/${resource}")
endforeach()

# --- setup files in the root of the install prefix
foreach(name
    local_setup.bash local_setup.fish.in local_setup.sh.in _local_setup_util.py
    local_setup.zsh setup.bash setup.fish setup.sh.in setup.zsh)
  _stage_file("${templates}/prefix_level/${name}" "${gen}/prefix" staged)
  install(FILES "${staged}" DESTINATION .)
endforeach()
