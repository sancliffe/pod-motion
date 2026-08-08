# pod-motion

A containerized deployment of the [Motion](https://motion-project.github.io/) video surveillance software using Podman and Kubernetes-style YAML definitions. It turns a local webcam into a DVR: motion events are detected and recorded straight to the host filesystem, with a web control panel and live stream exposed on the host network.

- **Codec/resolution:** MJPG at 1280x720, 30 fps
- **Recording:** 60-second MKV chunks written directly to the host
- **Deployment model:** single `podman play kube` command, no manual `podman run` flags

## Prerequisites

- **Podman** installed and running on the host machine.
- **Hardware:** a V4L2-compatible webcam connected to the host (default device path: `/dev/video0`).
- **SELinux:** on an enforcing system (e.g. Fedora), the host directories need the `container_file_t` context or Motion won't be able to read its config or write recordings.

## Directory structure

Only `config/` and `container/` are tracked in this repo. `storage/` is runtime output and needs to be created locally — it holds your recordings, so you don't want it in version control.

```text
~/motion/
├── config/
│   └── motion.conf       # Motion configuration file
├── container/
│   └── motion-pod.yaml   # Kubernetes-style Podman deployment file
└── storage/              # Recordings land here (create this — not in the repo)
```

## Quick start

**1. Clone the repository**

```bash
git clone https://github.com/sancliffe/pod-motion.git ~/motion
cd ~/motion
mkdir -p storage
```

**2. Match the volume paths to your host**

`container/motion-pod.yaml` hard-codes its `hostPath` volumes to `/home/steve/motion/...`. If your home directory isn't `/home/steve`, update both `hostPath.path` entries (for `motion-storage-vol` and `motion-config-vol`) to match your actual clone location, e.g.:

```bash
sed -i "s#/home/steve/motion#${HOME}/motion#g" container/motion-pod.yaml
```

**3. Apply SELinux context (if applicable)**

```bash
chcon -Rt container_file_t ~/motion/config ~/motion/storage ~/motion/container
```

**4. Deploy the pod**

```bash
podman play kube ~/motion/container/motion-pod.yaml
```

This creates the pod (`motion-deployment-pod`), mounts the config/storage volumes, maps the ports, and pulls `docker.io/motionproject/motion:latest` if it isn't already present.

**5. Confirm it's running**

```bash
podman pod ps
podman logs motion-deployment-motion
```

Then open `http://<host-ip>:7999` for the web control panel and `http://<host-ip>:8081` for the live stream.

## Configuration reference

### Ports

| Port | Purpose |
|------|---------|
| `7999` | Web control panel (configuration and admin interface) |
| `8081` | Live camera stream |

`webcontrol_localhost` and `stream_localhost` are set to `off` in `motion.conf` — required for either port to be reachable from outside the pod's network namespace. If you only want local access, flip these back to `on` and drop the `hostPort` mappings.

### Volume mounts

| Host path | Container path | Purpose |
|-----------|-----------------|---------|
| `~/motion/storage` | `/var/lib/motion` | Recorded MKV output |
| `~/motion/config` | `/usr/local/etc/motion` | `motion.conf` |
| `/dev/video0` | `/dev/video0` | Webcam passthrough |

If your webcam isn't at `/dev/video0`, update both the `webcam-device` volume and `video_device` in `motion.conf` to match (check with `v4l2-ctl --list-devices` or `ls /dev/video*`).

## Teardown

```bash
podman pod rm -f motion-deployment-pod
```

This stops and removes the pod and its containers. Recordings in `~/motion/storage` are left untouched.

## Troubleshooting

- **`podman play kube` succeeds but the pod restarts / camera not found:** check the container has access to the device — you may need `--security-opt label=disable` or to confirm the `/dev/video0` path exists on the host before deploying.
- **Web control panel or stream unreachable from another machine:** double-check `webcontrol_localhost off` and `stream_localhost off` are actually set in `motion.conf` (a stale mount from an old path will silently use Motion's built-in defaults instead).
- **`Permission denied` writing recordings, or Motion can't read `motion.conf`:** almost always a missed `chcon -Rt container_file_t` on an SELinux-enforcing host — re-run step 3 above.
- **Port already in use:** something else on the host is bound to 7999 or 8081; change the `hostPort` values in `motion-pod.yaml` and the corresponding `Service` ports.

## License

MIT — see [LICENSE](./LICENSE).
