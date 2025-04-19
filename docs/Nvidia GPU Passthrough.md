# Nvidia GPU Passthrough
Proxmox v8 guide for GPU passthrough into an LXC

## 1. Install NVIDIA drivers onto proxmox host
You need to first install the nvidia drivers onto the proxmox host so that the required devices (`/dev/nvidia*`) are present.

1. Add `non-free` and `non-free-firmware` to the `/etc/apt/sources.list` debian lines.  This enables the use of the closed-source nvidia drivers
2. `apt update` to update apt and add the new packages
3. `apt install nvidia-driver` to install the nvidia drivers.  Optionally to install a specific version instead of latest, use `apt install nvidia-tesla-XXX-driver`.  
	1. You might need to use an older version if your GPU is not compatible with the latest drivers. 
4. Reboot
5. Run `nvidia-smi` and check that all GPUs are listed.  If working, the nvidia drivers are now installed on proxmox host.

## 2. Prepare LXC for GPU passthrough
To pass the GPUs into the LXC, you need to pass all the devices created by the nvidia drivers & make sure they have the correct permissions.

1. In the host CLI run `usermod -aG render,video root` to add the root user to the render and video groups. 
2. Run `ls -al /dev/nvidia*` and check for the devices (`nvidia0`, `nvidia1`, ...) and universal groups (`nvidiactl`, `nvidia-uvm`, ...).  Also note the device major numbers (in the below example - `195`, `509`, and `234` for later).
```
root@prox1:~# ls -al /dev/nvidia*  
crw-rw-rw- 1 root root 195, 0 Nov 13 01:23 /dev/nvidia0  
crw-rw-rw- 1 root root 195, 255 Nov 13 01:23 /dev/nvidiactl  
crw-rw-rw- 1 root root 509, 0 Nov 13 01:23 /dev/nvidia-uvm  
crw-rw-rw- 1 root root 509, 1 Nov 13 01:23 /dev/nvidia-uvm-tools

/dev/nvidia-caps:  
total 0  
drwxr-xr-x 2 root root 80 Nov 13 01:23 .  
drwxr-xr-x 22 root root 6360 Nov 13 01:23 ..  
cr——– 1 root root 234, 1 Nov 13 01:23 nvidia-cap1  
cr–r–r– 1 root root 234, 2 Nov 13 01:23 nvidia-cap2
```

3. Open the LXC's config file with `nano /etc/pve/lxc/xxx.conf` and paste in the below lines.  Make sure that (a) the device major numbers are correct, and (b) that the `/dev/nvidia0` line corresponds to the device you want to passthrough.  If you want to passthrough multiple devices add multiple lines for each `/dev/nvidiaX` instance. 
```
lxc.cgroup2.devices.allow: c 195:* rwm
lxc.cgroup2.devices.allow: c 234:* rwm
lxc.cgroup2.devices.allow: c 509:* rwm
lxc.mount.entry: /dev/nvidia0 dev/nvidia0 none bind,optional,create=file
lxc.mount.entry: /dev/nvidiactl dev/nvidiactl none bind,optional,create=file
lxc.mount.entry: /dev/nvidia-modeset dev/nvidia-modeset none bind,optional,create=file
lxc.mount.entry: /dev/nvidia-uvm dev/nvidia-uvm none bind,optional,create=file
lxc.mount.entry: /dev/nvidia-uvm-tools dev/nvidia-uvm-tools none bind,optional,create=file
lxc.mount.entry: /dev/nvidia-caps/nvidia-cap1 dev/nvidia-caps/nvidia-cap1 none bind,optional,create=file
lxc.mount.entry: /dev/nvidia-caps/nvidia-cap2 dev/nvidia-caps/nvidia-cap2 none bind,optional,create=file
```

4. Reboot the LXC, open the CLI, and run `ls -al /dev/nvidia*`, checking to make sure the devices are present.

## 3. Install drivers within the LXC
You now need to install the same drivers as are on the host into the LXC.  Make sure the drivers are the same version, otherwise it may not work.

1. Follow steps 1-5 from part 1 to install and verify the nvidia drivers.
2. Install nvidia container toolkit using [nvidia's official guide](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/install-guide.html).
3. Run `nvidia-ctk config --set nvidia-container-cli.no-cgroups --in-place`, then reboot the container.

## 4. Setup Docker with GPU Passthrough
Now that the GPU has successfully been passed through, I recommend setting up docker with container-toolkit to allow passing GPUs into containers.

1. Install docker using the [official guide](https://docs.docker.com/engine/install/debian/#installation-methods).
2. Enable the nvidia container runtime with `nvidia-ctk runtime configure --runtime=docker` and restart docker's system service or reboot.
3. Create a docker container using the nvidia runtime.

## 4.1 Setup Jellyfin with Transcoding
I'll use Jellyfin as an example program which can use transcoding.  These steps generally follow NvidiaJellyfin's Nvidia GPU [hardware acceleration guide](https://jellyfin.org/docs/general/administration/hardware-acceleration/nvidia).  To install it and setup GPU transcoding:

1. Create a docker-compose.yml file with the below contents:
```
services:
  jellyfin:
    image: jellyfin/jellyfin
    volumes:
      - ./config:/config
      - ./cache:/cache
      - ./media:/media
    ports:
      - 8096:8096
    runtime: nvidia
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: all
              capabilities: [gpu]
```

2. Run the docker-compose file and open Jellyfin in your browser.
3. Go through first-time setup, creating a library.
4. Go to admin settings, playback, and enable nvidia transcoding.
5. In the LXC's cli, download a test video (IE `wget https://download.blender.org/peach/bigbuckbunny_movies/big_buck_bunny_1080p_h264.mov`) to the library's folder.
6. Refresh the library in Jellyfin, begin playback of the test video, and change the quality settings.  If transcoding is working it should pause for a moment then continue playback.
7. To check that the GPU is being used, you can use `nvtop` to check the workload.  Transcoding a single 1080p stream might not be that intensive, so you may need to change the video quality a few times to notice an uptick.

# References
- https://digitalspaceport.com/proxmox-lxc-gpu-passthru-setup-guide/
- https://www.youtube.com/watch?v=lNGNRIJ708k&t=1121s
- Jellyfin's nvidia HWA guide: https://jellyfin.org/docs/general/administration/hardware-acceleration/nvidia