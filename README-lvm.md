# Umbrel with LVM /data partition and updated kernel

This repo holds patches to keep an updated umbrel install that has an LVM /data partition to support additional disk, and to use backported kernel for better h/w support.

Brief usage info below.

## Convert /data to LVM and add new disk to it

The strategy is:

- Initialize LVM using only the new disk consuming all of its capacity
- Stop existing apps and services to avoid writing to /data.
- Copy content of /data into the LVM volume
- Activate the new /data by replacing the old entry in fstab, and restarting the machine.
- Add the old /data partition to the LVM group

Assuming the main disk is `/dev/nvme1n1` and the new disk is `/dev/nvme0n1`, here are the steps to do this:

1. Initialize LVM using only the new disk consuming all of its capacity

    ```/bin/bash
    # Create a new LVM physical volume
    pvcreate /dev/nvme0n1

    # Create a volume group (VG) using the entire second disk
    vgcreate data_vg /dev/nvme0n1

    # Allocate a logical volume (LV) for `/data`
    lvcreate -l 100%FREE -n data_lv data_vg

    # Format the LV with a filesystem
    mkfs.ext4 /dev/data_vg/data_lv
    ```

2. Stop existing apps and services to avoid writing to /data.

    ```
    systemctl stop umbrel
    ```
3. Copy content of /data into the LVM volume

    ```/bin/bash
    mkdir /mnt/newdata
    mount /dev/data_vg/data_lv /mnt/newdata

    # Preserve permissions, symlinks, and special files
    rsync -aAXv /data/ /mnt/newdata/
    ```
4. Activate the new /data by replacing the old entry in fstab, and restarting the machine.

    The entry for /data in `/etc/fstab` is enforced by umbrel image at every startup, therefore we'll need to replace it by the custom image from this repo.

    ```/bin/bash
    # Clone this repo on a separate dev machine with at least 60 GB of free disk space
    git clone https://github.com/mmta/umbrel.git
    cd umbrel/packages/os

    # Set a custom root password at the end of initialize.sh script
    vi build-steps/initialize.sh 

    # Build the custom image, this will take several minutes
    ./build-image-update.sh

    # Upload the custom image to the umbrel box
    cd build/
    scp umbrelos-amd64.update umbrel.local:~/

    # ssh into umbrel and install the custom image
    ssh umbrel.local
    sudo mender install umbrelos-amd64.update
    
    # restart to activate the new image
    sudo reboot

    # when maintenance mode prompt comes up, enter the root password set in initialize.sh above, then remove extranous /data mount from /etc/fstab using this script
    /opt/umbrel-data-part-lvm/umbrel-data-part-lvm 

    # exit maintenance mode to continue
    exit

    # at the login prompt, login as umbrel, then use sudo to lock the root account
    sudo -s
    passwd -l root
    ```
5. Add the old /data partition to the LVM group

    The system should now run using LVM /data partition, which can be extended live without interruption:
    ```
    # Convert the old partition (in this case /dev/nvme1n1p4) to an LVM physical volume
    pvcreate /dev/nvme1n1p4

    # Extend the volume group to include the old partition
    vgextend data_vg /dev/nvme1n1p4

    # Extend the logical volume
    lvextend -l +100%FREE /dev/data_vg/data_lv

    # Resize the filesystem to utilize the new space
    resize2fs /dev/data_vg/data_lv
    ```
