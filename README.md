# ROS2 Docker Compose Dev Setup

A two-stage Docker image for ROS2 (`jazzy` by default) plus a Compose file
pair for building and running it — no devcontainers, no Makefile.

## Layout

- **`Dockerfile`** — multi-stage build:
  - `base` stage: ROS2 install, non-root user matching your host UID/GID,
    and this project's ROS package deps. This is the "real" image — it runs
    fine on its own.
  - `dev` stage: built on top of `base`, adds editor/shell/build tooling
    (neovim, git, zsh, gdb, etc). Language servers, formatters, linters and
    debug adapters are installed by Mason when neovim starts, so the image does
    not need to track your host neovim config. Only needed for development.

- **`docker-compose.yml`** *(committed)* — builds the `base` stage and wires
  up everything needed to actually run it: host networking, IPC, GPU access,
  X11 forwarding, and the workspace bind mount.

- **`docker-compose.override.yml`** *(committed)* — adds the `dev` stage
  tooling on top: builds `target: dev`, mounts your local neovim config,
  persists Mason packages in a named `nvim-data` volume, enables
  `SYS_PTRACE`/`pid: host` for debugger attach, and keeps the container alive
  with `sleep infinity` so you can shell into it.

Compose automatically merges both files when you run `docker compose up` /
`build` — no `-f` flags needed.

## Usage

**Everyday development** (base + dev tooling):
```bash
docker compose up -d --build
docker compose exec ros2 zsh   # or bash
```

**Build/run just the base+deps image** (e.g. CI, or a real deployment —
ignore the override by pointing Compose only at the committed file):
```bash
docker compose -f docker-compose.yml up -d --build
```
Note: without the override's `command: sleep infinity`, the container falls
back to the Dockerfile's `CMD ["bash"]` and will exit immediately unless you
pass a real `command:` (e.g. a `ros2 launch ...`) in `docker-compose.yml`.

## First-time setup

1. `docker-compose.override.yml` is committed in this template. If you want to
   keep project-specific overrides private, copy it to
   `docker-compose.override.yml` (Compose picks it up automatically) and add
   your copy to `.gitignore`.
2. Optionally create a local `.env` (also gitignored) to pin your host UID/GID:
   ```
   USER_UID=1000
   USER_GID=1000
   ROS_DISTRO=jazzy
   ```
3. GPU support requires the NVIDIA Container Toolkit on the host and Docker
   Compose v2. If you don't need GPU/rendering, delete the
   `deploy.resources.reservations.devices` block in `docker-compose.yml`.

## Customizing per project

- ROS packages: edit the `apt-get install` list in the `base` stage of the
  `Dockerfile` (the block that installs `ros-${ROS_DISTRO}-*` packages).
- Dev tooling: edit the `dev` stage. Mason-managed packages are stored in the
  named `nvim-data` volume, so they survive container recreates.
- Workspace source lives at `/home/ros/ws` inside the container, bind-mounted
  from the repo root.
