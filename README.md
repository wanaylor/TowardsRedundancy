# Towards Redundancy
This is a set up guide for how to run Ubuntu Server 24.04 with it's root filesytem on zfs on a Pi 5

## Why?
I, like many folks with a homelab, find myself constantly wanting to expand what services I am running as I experiment with the latest and greatest. However, not all services are what I would consider critical and therefore do not need constant uptime. This has led me to divide my homelab into two segments, one that is considered critical and one that is for experimentation. I currently have everything on a single UPS which includes network gear, Konnected Alarm Pro, a Pi4, an old laptop, and an old airport for some bulk storage. Between the Pi 4 and laptop I run a variety of homelab services like HomeAssistant, ESPHome, Tailscale, Greylog, and a few others that I developed myself, like [Argus](https://github.com/wanaylor/Argus). I also have future plans for a Proxmox cluster or larger single server that I can virtualize with. This would be a great upgrade from my Pi4, which is relying on SD Card storage now, but at the cost of draining my UPS much more quickly. This led me to keep all critical services on the Pi with it on the UPS. But a Pi 4 with 1 SD card doesn't exactly scream high reliability or speed...

### Enter the Pi 5
With the release of the Pi 5 we got the ability to run dual (or even quad) nvme. This means we can do properly redundant storage. However, I don't just want redundant bulk storage, I want the OS drive to be redundant too. So how do we do this? There are a variety of options including mdadm, ceph, etc but zfs caught my eye because of some of the unique features it has like snapshotting.

### Into the unknown
Once I had settled on mirrored zfs drives for my Pi 5, now all that was left was to figure out how to actually do it. As it turns out, zfs on root has much better documentation and support for amd64 as opposed to arm. To add insult to injury Ubuntu allowed you to select zfs as an option during configuration but not for the arm installer! There is a guide in the OpenZFS docs for installing Ubuntu Server 22 on zfs on a Pi 4 and that's about as close as we can get. So here we go.

[zfs Ubuntu Server 22 on Pi4](https://openzfs.github.io/openzfs-docs/Getting%20Started/Ubuntu/Ubuntu%2022.04%20Root%20on%20ZFS%20for%20Raspberry%20Pi.html)


## Hardware
I will be using a Raspbery Pi 5 8GB with the Pimoroni nvme base duo and two WD Blue 1TB drives. The Pimoroni hat fits nearly under the Pi and even comes with rubber feat to keep it from sliding around.

![IMG_2096](https://github.com/user-attachments/assets/d70b0e99-fc87-4c0d-bc52-df1aed1ac503)

## How
Here I will post the modifications needed to the linked guide as I go through it.

### Disk Formatting
1. Change OS Image to Ubuntu Server 24.04

4. nvme partition labels are like nvmeXnYpZ

   ```
   DISK=/dev/nvme0n1
   
   export DISKP=${DISK}p
   ```

5. Also ensure zfs utils are installed

    ```
   sudo apt install zfsutils-linux
    ```

 ### Setup ZFS
 1. Replace drive names with nvme drives as before

### Setting Up The Mirror
My goal is to have mirrored drives so that if one drive fails I will have enough time to replace it. Unfortunately, zfs cannot mirror a vfat filesystem and the Raspbery Pi 5 requires a vfat filesystem for the boot partition. To work around that we will just simply copy the boot partition from the primary drive to the secondary drive and have the rpool mirrored across the larger partition on each drive. The boot partition will not update very often compared to the root filesystem so it is fine to copy it over every so often and let zfs mirror the rpool. Here's the steps to set up the mirror.
1. Follow the steps in the guide up to, and including, writing the image partition tables to the secondary drive. All we need is to have the partitions in place and we will let zfs copy all the data once added to the pool.
2. Boot with the primary drive in slot B of the Pimoroni dual hat (B being the primary drive is a strange quirk) and also have the secondary drive installed in the A slot.
3. Set `DISK` and `DISKP` for the secondary drive

```
DISK=/dev/nvme1n1
DISKP=/dev/nvme1n1p
```

4. Copy data from the boot partition on the primary drive to the boot partition on the secondary drive

```
sudo dd if=/dev/nvme0n1p1 of=${DISKP}1 bs=1M
```

5. Delete partition 3 off of the standby drive so that the partitions match the primary drive

```
sfdisk $DISK --delete 3
echo ", +" | sfdisk --no-reread -N 2 $DISK
```

6. Add the vdev to the rpool that was configured in the guide linked above
```
zpool status
zpool attach rpool /dev/nvme0n1p2 /dev/nvme1n1p2
```

### Now To Test
The purpose of this entire exercise was to be able to recover the sytem if the primary (or secondary) drive fails. So now we can test the worst case scenario, primary drive fail.
1. Power down the Pi
2. Remove the primary drive from slot B and set it aside
3. Remove the secondary drive from slot A and place it into slot B
4. Boot the system and hope

There will likely be a lot of errors on boot but the sytem should boot none the less. If you check the zpool status is will probably look something like this:

![IMG_2200](https://github.com/user-attachments/assets/451e15d2-e366-40bf-bf43-e55e1c1015d6)

This is to be expected though since one of the mirrors is removed. Once you are done looking around the system and proving that it is in fact intact power it back off and restore the drives to their original configuration.

Boot the sytem with the drives restored and check the zpool status. You will see errors in the status but the pool is otherwise online. Scrub the pool and clear the errors

```
zpool status
zpool scrub rpool
zpool clear rpool
zpool status
```


Everything should be back to normal now. Congrats, you now have a very low power device that can run your most important homelab services with the peace of mind knowing that you have redundant storage.
