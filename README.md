# Motion Podman Deployment

## Overview
A containerized deployment of the Motion video surveillance software using Podman and Kubernetes-style YAML definitions. This project simplifies deploying a local webcam stream and DVR setup by defining the container resources, port mappings, and volume mounts in a declarative format. 

The configuration optimizes performance by utilizing the MJPG codec to achieve 30 frames per second at a 1280x720 resolution. It records motion events in 60-second MKV chunks directly to the host filesystem[cite: 1].

## Prerequisites
* **Podman:** Installed and running on the host machine.
* **Hardware:** A V4L2 compatible webcam connected to the host (default: `/dev/video0`).
* **SELinux:** If running on an SELinux-enforcing system (like Fedora), the local directories must have the `container_file_t` context applied to allow the pod to read configurations and write video files.

## Directory Structure
Before deploying, ensure the following directory structure exists on your host machine to match the volume mounts:

```text
~/motion/
├── config/
│   └── motion.conf       # Motion configuration file
├── container/
│   └── motion-pod.yaml   # Kubernetes-style Podman deployment file
└── storage/              # Output directory for MKV recordings
```
## Quick Start
1.) Clone the repository:
```Bash
git clone [https://github.com/sancliffe/motion-deployment.git](https://github.com/sancliffe/motion-deployment.git) ~/motion
cd ~/motion
```
2.) Prepare SELinux (If applicable):
```Bash
chcon -Rt container_file_t ~/motion/config ~/motion/storage ~/motion/container
```
3.) Deploy the Pod:
Use Podman to play the Kubernetes YAML file, which creates the pod, mounts the volumes, and maps the ports:  
```Bash
podman play kube ~/motion/container/motion-pod.yaml
```
## Configuration Details
Port MappingsThe deployment automatically publishes the following ports to the host network:  
`7999`: Web Control Panel (Configuration and administrative interface).  
`8081`: Live Camera Stream.
Note: `webcontrol_localhost` and `stream_localhost` must be set to `off` in the motion.conf file to allow connections outside the pod's isolated network.

## Volume Mounts
The pod maps three crucial paths from the host to the container:  
`/home/steve/motion/storage -> /var/lib/motion (Video outputs).`
`/home/steve/motion/config -> /usr/local/etc/motion (Config files).`
`/dev/video0 -> /dev/video0 (Webcam passthrough).`

## Teardown
To completely stop and remove the deployment:
```Bash
podman pod rm -f motion-deployment-pod
```
