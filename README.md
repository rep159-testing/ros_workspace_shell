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

`templates/` holds unmodified copies of these upstream files, without the
Windows `.bat` templates:

| Files in `templates/` | Copied from | Version |
|---|---|---|
| `prefix_level/*`, `package_level/*`, `environment_hook/library_path.sh`, `environment_hook/pythonpath.sh.in` | [ament/ament_package](https://github.com/ament/ament_package), `ament_package/template/` | tag `0.18.3`, commit `98089783110f7606a90e8ef9786719c3d0cd32f9` |
| `environment_hook/path.sh` | [ament/ament_cmake](https://github.com/ament/ament_cmake), `ament_cmake_core/cmake/environment_hooks/environment/path.sh` | tag `2.8.8`, commit `46a63275c1c2ff2383f32d0c378784aafd70e6e2` |

The `path` hook comes from ament_cmake_core and not from ament_package,
which has a different `path.sh`: ament_cmake installs the ament_cmake_core
one as the default `path` hook of every package, so `ros_workspace`
installed that one.

The files this package generates from them are identical to the ones
ament_cmake generates for `ros_workspace` 1.0.4, apart from the package
name. To update them, copy the new versions over these and update the
table.

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
