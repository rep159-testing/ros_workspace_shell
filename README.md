# ros_workspace_base

The setup files in the root of a ROS 2 install prefix (`setup.bash`,
`local_setup.sh` and the others) and the environment hooks for the whole
prefix, built with plain CMake. Neither building nor using this package
needs any ament package.

It is the part of `ros_workspace` that a distribution without ament packages
needs, for example a Gazebo distribution that ROS distributions extend
(REP 159). `ros_workspace` depends on this package and adds the ament part:
`AMENT_PREFIX_PATH` for the prefix and the package's CMake config files.

## What it installs

| Where | Files |
|---|---|
| root of the prefix | `setup.{sh,bash,zsh,fish}`, `local_setup.{sh,bash,zsh,fish}`, `_local_setup_util.py` |
| `share/ros_workspace_base/environment/` | `library_path`, `path`, `pythonpath` and, when the library directory is a multiarch one, `multiarch_library_paths`, each as `.sh` and `.dsv` |
| `share/ros_workspace_base/` | `local_setup.{sh,bash,zsh,fish,dsv}`, `package.dsv`, `package.xml` |
| `share/ament_index/resource_index/` | `packages`, `package_run_dependencies` and `parent_prefix_path` entries for `ros_workspace_base` |

The resource index entries are plain files that the setup files read to
find the packages of the prefix; writing them needs no ament code.

After sourcing `setup.bash`, the prefix is on `PATH` (if it has a `bin`
directory), `LD_LIBRARY_PATH` (`DYLD_LIBRARY_PATH` on macOS) and
`PYTHONPATH`. `AMENT_PREFIX_PATH` is set by `ros_workspace`, not by this
package.

## Templates

`templates/` holds unmodified copies of the ament_package 0.18.3 templates
and of the `path` hook of ament_cmake_core 2.8.8, from their `lyrical`
branches. The files this package generates from them are identical to the
ones ament_cmake generates for `ros_workspace` 1.0.4, apart from the package
name. To update them, copy the new versions over these.

## Parent distributions (REP 159)

A distribution that extends others with `binary_import` passes their install
prefixes, nearest first:

```
-DROS_WORKSPACE_PARENT_PREFIX_PATH="/opt/ros/parent_a;/opt/ros/parent_b"
```

The setup files then source each parent's `local_setup` first. A root
distribution leaves the list empty.

## Building

```
cmake -S . -B build -DCMAKE_INSTALL_PREFIX=/opt/ros/<distro>
cmake --build build
cmake --install build
```

The build needs a C compiler, used only to find the multiarch library
directory, and Python 3, whose path is written into the setup files.

## Limitations

Linux and macOS only: the Windows `.bat` setup files are not generated.
