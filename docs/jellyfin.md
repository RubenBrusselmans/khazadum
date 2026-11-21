# Jellyfin

Media server (linuxserver image, `latest@sha256:` pinned and updated by
Renovate, see [renovate.md](renovate.md)). Transcoding uses the host GPU via a
`/dev/dri` hostPath mount.

## `privileged: true` is required: removing it breaks GPU device access

The devices cgroup is the kernel's per-container list of device files a
process may open. containerd/k3s does not add hostPath device nodes
(`/dev/dri/*`, `/dev/kfd`) to that allowlist for unprivileged pods, so the
kernel refuses every `open()` with EPERM, no matter what groups, caps or
uid the container runs as.

Three de-privileging attempts, all reverted on 2026-08-22, hit the
devices-cgroup wall:

1. `supplementalGroups: [303, 26]` + drop ALL caps.
   (`supplementalGroups` is a docker-runtime pattern that kubernetes doesn't
   wire into the device allowlist.)
2. Bounded caps for linuxserver's s6 init. Learned along the way: s6
   must start as root, and the ROCm exporter (AMD's GPU userspace stack)
   needs root-writable `/etc/metrics`, `/var/log`, `/var/run`.
3. Root + zero caps.

The same applies to device-metrics-exporter (ROCm/GPU exporter, lives in the
gundabad repo).

Proper fix (roadmap): the amdgpu device plugin DaemonSet
(`rocm/k8s-device-plugin`). A device plugin advertises the GPU as a
schedulable resource (`amd.com/gpu`) and manages the device node access for
pods that request the resource. Jellyfin and device-metrics-exporter then
request it and drop `privileged` for good.

## Verification gotcha

ffmpeg listing `*_vaapi` encoders in the logs is build-time info, not
device access. Probe real device access in an exec shell:

```sh
kubectl -n jellyfin exec -it deployment/jellyfin -- sh -c ': < /dev/dri/renderD128 && echo OK'
```

## Other facts

- `PUID=1001` / `PGID=100` (linuxserver convention) for the media files on
  the hostPath volumes.
- Volumes: `/data/crypt/jellyfin` (config), `/data/crypt/media` (media,
  shared with nextcloud), `/dev/dri` (GPU).
- `strategy: Recreate` + `replicas: 1`.
