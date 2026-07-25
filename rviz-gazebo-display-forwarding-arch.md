# RViz & Gazebo GUI forwarding from Docker on Arch Linux (Wayland/XWayland)

How to forward the RViz and Gazebo GUIs from a Docker container to the host
display on an Arch Linux machine running a Wayland session with an NVIDIA GPU.

## Quick recipe

RViz and Gazebo are X11 applications built on Qt and OGRE. On a Wayland desktop
they render through **XWayland**, so the container needs plain X11 access plus
the NVIDIA runtime:

```yaml
services:
  sim:
    image: <YOUR_IMAGE_NAME>
    environment:
      - DISPLAY                              # :0 (XWayland display)
      - QT_X11_NO_MITSHM=1                   # Qt shared-memory workaround
      - XAUTHORITY=/root/.Xauthority         # where the cookie is INSIDE the container
      - NVIDIA_VISIBLE_DEVICES=all
      - NVIDIA_DRIVER_CAPABILITIES=all
    runtime: nvidia
    network_mode: host
    ipc: host
    privileged: true
    stdin_open: true
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: all
              capabilities: [gpu]
    volumes:
      - /tmp/.X11-unix:/tmp/.X11-unix        # X socket
      # On Wayland/XWayland the real X auth cookie lives at $XAUTHORITY,
      # NOT at ~/.Xauthority. Bind-mount whatever the host var points to:
      - "$XAUTHORITY:/root/.Xauthority:ro"
```

Every line of the GUI block is needed. The sections below explain each piece.

## The pieces and why each is needed

### 1. X socket mount + host networking

```yaml
volumes:
  - /tmp/.X11-unix:/tmp/.X11-unix
network_mode: host
ipc: host
```

- `/tmp/.X11-unix` is the Unix socket XWayland listens on. Without it the
  container cannot reach the display at all.
- `network_mode: host` means the container sees the same `:0` display address
  as the host. It also lets DDS traffic flow between containers without port
  mapping.
- `ipc: host` is needed for X11 MIT-SHM shared-memory images. Together with
  `QT_X11_NO_MITSHM=1` it stops Qt and RViz from crashing or showing broken
  windows when shared memory is not reachable across the namespace boundary.

### 2. The XAuthority cookie - the main problem on Arch/Wayland

This is the part that differs from every Ubuntu tutorial. X clients must
present an authentication cookie, and tutorials assume it lives at
`~/.Xauthority`. **Under a Wayland session it does not.** The cookie lives at
a runtime path stored in the host's `$XAUTHORITY` variable. Under GDM this is
`/run/user/1000/gdm/Xauthority`, where `1000` is your user ID. Other display
managers use paths like `/run/user/1000/xauth_XXXXXX`.

The fix is to bind-mount the *host's* `$XAUTHORITY` file to a fixed path in the
container and point the container's `XAUTHORITY` at that path:

```yaml
environment:
  - XAUTHORITY=/root/.Xauthority
volumes:
  - "$XAUTHORITY:/root/.Xauthority:ro"
```

Notes:

- `$XAUTHORITY` is expanded by docker compose from the shell you run
  `docker compose up` in.
- Run compose from a graphical terminal session where `echo $XAUTHORITY`
  prints a path. From SSH or a bare TTY the variable is empty, and the mount
  points to the wrong file without showing any error.
- Hardcoding the GDM path `/run/user/1000/gdm/Xauthority` also works, but the
  path changes with the display manager and session, so it breaks easily.
  Prefer the `$XAUTHORITY` variable.
- Alternative if the cookie mount does not work: run `xhost +local:` on the
  host. This allows all local connections and skips the cookie check
  completely.
  It is less safe because it opens the X server to every local process, so
  prefer the cookie mount and use `xhost` only for debugging.

### 3. DISPLAY

```
DISPLAY=:0
```

Set this in a `docker/.env` file instead of passing it from the shell.
XWayland is `:0` on a normal desktop machine, and with the fixed value
`docker compose up` works the same no matter where you start it from.
`DISPLAY=$DISPLAY` pass-through also works when compose runs from a graphical
terminal.

### 4. NVIDIA GPU - hardware OpenGL for Gazebo and RViz rendering

Without GPU access, Gazebo falls back to llvmpipe software rendering. That is
slow and sometimes fails on OGRE shaders. With the proprietary NVIDIA driver
on Arch the host Mesa/DRI devices are not enough. The container needs the
NVIDIA container runtime:

```yaml
runtime: nvidia
environment:
  - NVIDIA_VISIBLE_DEVICES=all
  - NVIDIA_DRIVER_CAPABILITIES=all     # must include "graphics", "all" is simplest
deploy:
  resources:
    reservations:
      devices:
        - driver: nvidia
          count: all
          capabilities: [gpu]
```

Host prerequisites on Arch:

```bash
sudo pacman -S nvidia-container-toolkit
sudo nvidia-ctk runtime configure --runtime=docker   # writes /etc/docker/daemon.json
sudo systemctl restart docker
```

`NVIDIA_DRIVER_CAPABILITIES=all` is important. The common `compute,utility`
default gives CUDA but **no GLX/OpenGL**, and RViz and Gazebo then still use
software rendering.

### 5. `privileged: true`

This avoids device-access problems inside the sim containers, for example with
input devices and DRI nodes. It gives the container more access than strictly
needed, but that is fine for local single-user simulation.

### 6. `.env` centralization

A `docker/.env` file holds all shared values in one place:

```
DISPLAY=:0
ROS_DOMAIN_ID=<YOUR_DOMAIN_ID>
QT_X11_NO_MITSHM=1
NVIDIA_VISIBLE_DEVICES=all
NVIDIA_DRIVER_CAPABILITIES=all
GAZEBO_RESOURCE_PATH=/usr/share/gazebo-11
GAZEBO_MODEL_PATH=<YOUR_MODEL_PATH>
ROS_LOCALHOST_ONLY=1
CYCLONEDDS_URI=file://<YOUR_CYCLONEDDS_XML_PATH>   # only if you use a CycloneDDS config
```

The compose services then list bare variable names like `- DISPLAY` so compose
reads them from `.env`. Only **one** container per scenario needs the GUI/GPU
block, the service that runs Gazebo and RViz. Other containers can run
headless with just `network_mode: host` + `ipc: host` and still communicate
over DDS. There is no reason to give every container X access.

## Gazebo-specific env

These are not display settings, but Gazebo needs them to start without
problems:

```yaml
- GAZEBO_RESOURCE_PATH=/usr/share/gazebo-11
- GAZEBO_MODEL_PATH=/opt/ros/humble/share/turtlebot3_gazebo/models   # or your model path
- GAZEBO_MODEL_DATABASE_URI=          # blank → stops Gazebo hanging on online model DB fetch
- TURTLEBOT3_MODEL=burger             # TB3 setups only
```

## Troubleshooting checklist

| Symptom | Cause / fix |
|---|---|
| `Could not connect to display :0` or `Authorization required` | Cookie mount is wrong. Check that `echo $XAUTHORITY` on the host prints a path in the shell running compose. Fallback is `xhost +local:` |
| RViz window opens but is black, broken, or crashes with X errors | `QT_X11_NO_MITSHM=1` and `ipc: host` are missing |
| Gazebo runs at ~2 fps and `glxinfo` in the container shows llvmpipe | NVIDIA runtime is not active. Check `runtime: nvidia`, that `NVIDIA_DRIVER_CAPABILITIES` includes graphics, and that `nvidia-ctk runtime configure` was run |
| Works from a desktop terminal, fails from SSH | `$DISPLAY` and `$XAUTHORITY` are empty in that shell. Set `DISPLAY=:0` in `.env` and run from a graphical session |
| Gazebo hangs for minutes at startup | Set `GAZEBO_MODEL_DATABASE_URI=` to an empty value to disable the online model database |
