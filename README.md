Application Containers for KrakenSDR
---

## Prerequisites

### Removing USB buffer size limitation
Since for most linux distros `usbcore` module is builtin the kernel, we need to modify its parameters via the kernel arguments as
```bash
sudo grubby --args=usbcore.usbfs_memory_mb=0 --update-kernel=ALL
```
### Allowing user to request realtime process priorities
```bash
sudo echo "YOURUSERNAME -   rtprio  99" > /etc/security/limits.d/98-YOURUSERNAME.conf
```
### Allow realtime tasks to monopolize CPU
```bash
sudo echo "kernel.sched_rt_runtime_us=-1" >> /etc/sysctl.d/99-sysctl.conf
```

## Building container
```bash
podman build --tag kraken/doa -f ./Dockerfile.doa
```

## Running container
```bash
podman run -dt --rm --privileged --shm-size=0 --group-add keep-groups -p 8080:8080/tcp -p 8081:8081/tcp kraken/doa
```

## Sopping container
```bash
podman stop kraken/doa
```