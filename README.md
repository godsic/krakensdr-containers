Application Containers for [KrakenSDR](https://krakenrf.com)
---

## Prerequisites

### Containers manager

Either [podman](https://podman.io/), [Docker](https://www.docker.com/) or [Moby](https://mobyproject.org/) would work. Please refer to your Linux distribution documentation for instructions on how to install either of those projects. The author prefers podman for its daemonless and rootless operation, thus you would see it in examples hereafter. Feel free to replace `podman` with `docker`, if you prefer Docker or Moby projects.

### Blacklisting `dvb_usb_rtl28xu`

`librtlsdr` should be able to directly access KrakenSDR recievers, so we need to blacklist kernel modules that might take over otherwise by

```bash
sudo echo "blacklist dvb_usb_rtl28xxu` > /etc/modprobe.d/blacklist-dvb_usb_rtl28xxu.conf
```

### Removing USB buffer size limitation

Since for most Linux distros `usbcore` module is builtin the kernel, we need to modify its parameters via the kernel arguments as

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

### Remote operation

If you are planning on using apps remotely, you might need to open ports `8080` and `8081` in the Linux distro firewall on the host machine. Please refer to your distro documentation for specific instructions. For those using [firewalld](https://firewalld.org/), e.g, Fedora, CentOS Stream, etc., the ports can be opened with the following commands:

```bash
sudo firewall-cmd --permanent --zone=public --add-port=8080-8081/tcp
sudo firewall-cmd --reload
```

### Apply changes

```
sudo reboot
```

## [KrakenSDR DoA DSP](https://github.com/krakenrf/krakensdr_doa)

### Building container

```bash
podman build --tag kraken/doa -f ./Dockerfile.doa
```

### Running container

```bash
podman run -dt --rm --privileged --shm-size=0 --group-add keep-groups -p 8080:8080/tcp -p 8081:8081/tcp kraken/doa
```

### Accessing web interface

Navigate to `localhost:8080` in your favorit browser.

### Sopping container

```bash
podman stop kraken/doa
```

## Advanced host settings

### [`tuned`](https://tuned-project.org/)

A number of Linux distributions provide `tuned` service and `tuned-adm` app that can be used to tune various system settings based on either included or custom profiles. If these are avaliable on your host, then please invoke:

```bash
sudo tuned-adm profile accelerator-performance
sudo tuned-adm profile accelerator-performance intel-sst
```

that would disable all CPU power saving technologies and tune your system to increase its responsiveness.

### Memory reservation

To avoid possible spikes in processing latency, it might be beneficial to set aside certain amount of host memory for the container via the `--memory-reservation` flag. For example, according to `podman stats`,  DoA app consumes around `600 MB`, so it is reasonable to reserve `1 GB` of host memory for DoA container via

```bash
podman run -dt --rm --privileged --memory-reservation=1g --shm-size=0 --group-add keep-groups -p 8080:8080/tcp -p 8081:8081/tcp kraken/doa
```

### Restrict execution to particular CPUs

It might be beneficial to restrict Kraken application containers execution to specific CPU cores, e.g., to physical cores in SMT systems, or to prevent non-unifom memory access in multi-CPU configurations. This can be done with so-called `cpuset` cgroup controller, but it might not be allowed for regular users. One can check it with

```bash
cat /sys/fs/cgroup/user.slice/user-*.slice/user@*.service/cgroup.subtree_control
```

If output contains `cpuset` among other things, then it is allowed. If not, then additional, distribution-specific configuration should be done. For instance, on stock `Fedora 36`, one needs to modify `/usr/lib/systemd/system/user@.service.d/00-uresourced.conf`
to make sure its content looks like:

```conf
[Unit]
After=uresourced.service

[Service]
Delegate=cpuset cpu io memory
```

and then restart the system and, in some cases, run

```bash
podman system reset
```

Once all is set, one can use podman's `--cpuset-cpus` flag to restict execution to specific CPUs, e.g.,

```bash
podman run -dt --rm --privileged --memory-reservation=1g --cpuset-cpus=0,1 --shm-size=0 --group-add keep-groups -p 8080:8080/tcp -p 8081:8081/tcp kraken/doa
```
