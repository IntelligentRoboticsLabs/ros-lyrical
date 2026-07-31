# EasyNav (EasyNavigation) on ROS 2 lyrical

This documentation covers two related workflows:

1. **For users**: how to set up a Pixi-based ROS workspace that consumes EasyNav/NavMap binaries and builds your own code with `colcon`.
2. **For package creators**: how to build EasyNav/NavMap packages in this repository (`ros-lyrical/`) and publish them to prefix.dev.

Project websites:

- EasyNavigation (EasyNav): https://easynavigation.github.io/
- NavMap: https://easynavigation.github.io/ (NavMap is part of the EasyNavigation ecosystem)

Pixi background and common troubleshooting lives in:

- `irl-docs/pixi.md`

---

## 1) What is this?

EasyNav is a navigation stack developed by IRL (and collaborators) for ROS 2.

In this repo we build EasyNav-related ROS packages as Conda artifacts (`.conda`) and publish them to prefix.dev (channel `irl-lyrical`).

Note: EasyNav/NavMap are not yet part of the official `lyrical` rosdistro snapshot bundled with vinca, even though `release/lyrical/*` tags already exist upstream for all of them. The `easynav_subset`/`navmap_subset` configs in this repo select exactly the packages needed; see `rosdistro_snapshot.yaml` for how these are resolved.

---

## 2) For users: workspace with Pixi + colcon

This folder contains a ready-to-use workspace template:

- `irl-docs/easynav/pixi.toml`
- `irl-docs/easynav/activate.sh`

It is aligned with the workspace used in this repo (`easynav_plugins_ws/`).

### 2.1 Create a new workspace

Create a new folder for your workspace and copy the template files:

```bash
mkdir -p ~/easynav_plugins_ws
cd ~/easynav_plugins_ws

cp /path/to/pixi-buildfarm/ros-lyrical/irl-docs/easynav/pixi.toml ./pixi.toml
cp /path/to/pixi-buildfarm/ros-lyrical/irl-docs/easynav/activate.sh ./activate.sh
chmod +x ./activate.sh
```

Then update the local file-based channel path if needed:

- `file:///.../ros-lyrical/output`

Important: use the **channel root**, not `.../output/linux-64`. See `irl-docs/pixi.md`.

What the template does by default:

- Prefers Zenoh (`RMW_IMPLEMENTATION=rmw_zenoh_cpp`).
- Pins the Eigen ABI (`eigen-abi-devel`) to avoid CMake falling back to system Eigen via PCL.
- Enables bash completion for `ros2` (via `argcomplete`) in interactive shells.

### 2.2 Install dependencies

```bash
pixi install
```

### 2.3 Build your ROS workspace

Add your ROS packages under `src/` (as in a normal colcon workspace) and build:

```bash
pixi run build
```

After the first build, the activation script will source `install/setup.bash` automatically.

### 2.4 Running EasyNav

Exact commands depend on your workspace and launch files.

As a rule of thumb:

- Use `ros2 launch ...` for launching systems.
- Use `ros2 run ...` for single executables.

If you need a curated set of binaries from `irl-lyrical`, keep the channel order in `pixi.toml` so local artifacts override remote ones.

### 2.5 Worked example: indoor testcase + Kobuki + Summit playgrounds

This is the workflow we use for jazzy/kilted/rolling to add content, build, and run. At the time of writing, the demo repos below don't have a `lyrical` branch yet (only `jazzy`, `kilted`, `rolling`); until it lands, `kilted` or `rolling` is the closest substitute:

Repositories and branches:

- https://github.com/EasyNavigation/easynav_indoor_testcase.git (branch: `main`)
- https://github.com/EasyNavigation/easynav_playground_kobuki.git (branch: `kilted` or `rolling`, pending a `lyrical` branch)
- https://github.com/EasyNavigation/easynav_playground_summit.git (branch: `kilted` or `rolling`, pending a `lyrical` branch)

Fetch the workspace content:

```bash
cd ~/easynav_plugins_ws
mkdir -p src

git clone -b main https://github.com/EasyNavigation/easynav_indoor_testcase.git src/easynav_indoor_testcase
git clone -b kilted https://github.com/EasyNavigation/easynav_playground_kobuki.git src/easynav_playground_kobuki
git clone -b kilted https://github.com/EasyNavigation/easynav_playground_summit.git src/easynav_playground_summit
```

Import third-party repositories referenced by the playgrounds:

```bash
cd ~/easynav_plugins_ws

vcs-import src < src/easynav_playground_kobuki/thirdparty.repos
vcs-import src < src/easynav_playground_summit/thirdparty.repos
```

Build:

```bash
cd ~/easynav_plugins_ws
pixi run build
```

Run (multiple terminals):

1) Open each terminal and enter the environment:

```bash
cd ~/easynav_plugins_ws
pixi shell
```

2) Terminal A (Zenoh daemon, start this before any nodes):

```bash
ros2 run rmw_zenoh_cpp rmw_zenohd
```

3) Terminal B (Kobuki playground):

```bash
ros2 launch easynav_playground_kobuki playground_kobuki.launch.py
```

4) Terminal C (indoor testcase + costmap):

```bash
ros2 launch easynav_indoor_testcase easynav_costmap_serest.launch.py
```

5) Optional: terminal D (TUI):

```bash
ros2 run easynav_tools tui
```

---

## 3) For package creators: buildfarm in this repo

All buildfarm work happens under `ros-lyrical/`.

### 3.1 Quick path (Pixi tasks)

```bash
cd ros-lyrical
pixi install
pixi run easynav-all
```

This runs, in order: clean → generate recipes → build → index.

### 3.2 How the subset is defined

The EasyNav subset is defined in:

- `ros-lyrical/easynav_subset/vinca.yaml`

Notes:

- Vinca reads `vinca.yaml` from the directory passed via `-d`.
- Generated recipes always go into `ros-lyrical/recipes/`.

### 3.3 Manual flow (if you need it)

Generate recipes:

```bash
cd ros-lyrical
pixi run remove-recipes
pixi run -v vinca -d ./easynav_subset --platform linux-64 -m
```

Build `.conda` artifacts:

```bash
pixi run rattler-build build \
  --recipe-dir ./recipes \
  --target-platform linux-64 \
  -m ./conda_build_config.yaml \
  -c https://prefix.dev/robostack-lyrical \
  -c https://repo.prefix.dev/conda-forge \
  --skip-existing
```

Index the local channel:

```bash
pixi run rattler-index fs ./output --force
```

### 3.4 Upload to prefix.dev

Typical upload to `irl-lyrical`:

```bash
cd ros-lyrical
export PREFIX_API_KEY="..."

pixi run rattler-build upload prefix \
  --channel irl-lyrical \
  --force \
  output/linux-64/*.conda
```

If consumers can't see newly-published packages, clear repodata cache on the consumer side:

```bash
pixi clean cache --repodata -y
```

If uploads get rate-limited, retry with backoff, one file per request:

```bash
cd ros-lyrical
export PREFIX_API_KEY="..."

for pkg in output/linux-64/*.conda; do
  for attempt in 1 2 3 4 5; do
    if pixi run rattler-build upload prefix --channel irl-lyrical --force "$pkg"; then
      break
    fi
    sleep $((attempt * 2))
  done
done
```

---

## Eigen: detect and avoid system contamination

If a consumer workspace fails with Eigen-related errors, a common cause is mixing include trees:

- System Eigen: `/usr/include/eigen3`
- Pixi/conda Eigen: `$CONDA_PREFIX/include/eigen3`

`ros-lyrical-easynav-common` and `ros-lyrical-navmap-ros` are patched (`patch/ros-lyrical-easynav-common.patch`, `patch/ros-lyrical-navmap-ros.patch`) to resolve Eigen3 from the conda prefix explicitly and to strip `/usr/include/eigen3` out of the `PCL_INCLUDE_DIRS` list injected by `PCLConfig.cmake`. `patch/dependencies.yaml` also pins `eigen-abi-devel` as a host dependency for both packages, matching how `pcl_conversions` is pinned in this repo.

Quick verification if you still hit this:

- Search for `/usr/include/eigen3` in `build/**/flags.make`.
- Create a small CMake project that does `find_package(PCL CONFIG REQUIRED)` and verify `Eigen3::Eigen` points at `$CONDA_PREFIX/include/eigen3`.

---

## Status

The `easynav_subset` and `navmap_subset` recipe generation has been validated (`pixi run easynav-generate-recipes`, `pixi run navmap-generate-recipes`) against the packages listed above. Full `.conda` builds and the `irl-lyrical` publish step have not been run yet from this repo.
