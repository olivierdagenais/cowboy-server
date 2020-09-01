# cowboy-server

Configuration files for a RancherOS deployment

## Installing RancherOS

### Links

* [Installing and Running RancherOS](https://rancher.com/docs/os/v1.x/en/installation/)
* [Installing to Disk](https://rancher.com/docs/os/v1.x/en/installation/server/install-to-disk/)

### Create the Cloud-Config file

First, we need a Cloud-Config file, mostly to bring in the SSH public key(s) for password-less authentication.

1. Create a `cloud-config.yml` file with these lines:

    ```yaml
    #cloud-config
    ssh_authorized_keys:
    ```

1. For each workstation that you'll want to use to connect to the server via SSH:
    1. Create SSH key:

        ```sh
        ssh-keygen -f ~/.ssh/cowboy_rsa -C $COMPUTERNAME@cowboy
        ```

    2. Add the following to the workstation's `~/.ssh/config` file (notice the second line is indented by 2 spaces):

        ```config
        Host cowboy
          IdentityFile ~/.ssh/cowboy_rsa
        ```

    3. Copy the contents of the `/.ssh/cowboy_rsa.pub` file and paste it in on a new line in the `cloud-config.yml` file, prefixing with 2 spaces, a hyphen, and a space.  Example:

        ```yaml
          - ssh-rsa AAA...
        ```

1. Configure the host name by inserting this as the 2nd line of `cloud-config.yml`:

    ```yaml
    hostname: cowboy
    ```

Now we have a `cloud-config.yml` that looks suspiciously like the one in this repository.

### Boot the RancherOS installer

You can download the `rancheros.iso` file from [the RancherOS releases page](https://github.com/rancher/os/releases/).  I configured a USB storage device with it using [UNetbootin](https://unetbootin.github.io/).

1. Boot from the USB storage device.
1. Record what storage device is attached where; both of these commands will give you information about what storage devices are connected, how big they are and what partitions might already exist on them:

    ```sh
    sudo fdisk -l
    sudo parted -l
    ```

    ...this will inform what the destination parameter (`-d`) will take as a value for the upcoming `ros install` command.
1. Install!  You'll notice that the URL that is provided to the `-c` parameter is that of the file from this repository, which needs to be retrievable anonymously.

    ```sh
    sudo ros install -c https://raw.githubusercontent.com/olivierdagenais/cowboy-server/main/cloud-config.yml -d /dev/sda
    ```

    If you are installing from USB-based storage and it fails right away with:

    ```log
    ERRO[0000] Failed to get boot iso: stat /dev/sr0: no such file or directory
    There is no boot iso drive, terminate the task
    FATA[0000] Failed to run install
    err="stat /dev/sr0: no such file or directory"
    ```

    ...then you might be hitting [#2241](https://github.com/rancher/os/issues/2241) (affects version 1.5.0), so try this workaround:

    ```sh
    sudo mkdir /dev/sr0
    ```

    ...and run the install command again.

    You'll be prompted to install and again to reboot.
1. Connect via SSH using one of your workstations:

    ```sh
    ssh rancher@cowboy
    ```

    ...unfortunately, as much as I tried to get `agetty` to include the public key fingerprints in the TTYs, I was not able to and thus you'll have to trust that your first connection is safe and accept the key fingerprint that's presented.

## Set up ZFS

1. Install the ZFS service as per [Using ZFS](https://rancher.com/docs/os/v1.x/en/installation/storage/using-zfs/):
    1. Download a container to compile ZFS for Linux:

        ```sh
        sudo ros service enable zfs
        ```

        ...this will print something like:

        ```log
        Pulling zfs (docker.io/rancher/os-zfs:v0.7.13-1)...
        (...)
        ```

        ...and take a minute or three.
    2. Install the kernel headers, download ZFS on Linux, build & install it:

        ```sh
        sudo ros service up zfs
        ```

        ...wait another few minutes.
1. Configure first ZFS pool, with inspiration from [ZFS Concepts and Tutorial](https://linuxhint.com/zfs-concepts-and-tutorial/):
    1. Cheat-sheet of commands to query the state while performing the following operations:

        ```sh
        sudo zpool list
        sudo zpool status
        sudo zfs list
        ```

    1. Create "internal" pool of type `raidz1` (needs 3 disks):

        ```sh
        sudo zpool create -m /mnt/internal internal raidz1 /dev/sdb /dev/sdc /dev/sdd
        ```

    1. Stop Docker and delete its old/current folder:

        ```sh
        sudo system-docker stop docker
        sudo rm -rf /var/lib/docker/*
        ```

    1. Create a file system for Docker:

        ```sh
        sudo zfs create internal/docker
        sudo zfs list -o name,mountpoint,mounted
        ```

    1. Configure Docker for ZFS:

        ```sh
        sudo ros config set rancher.docker.storage_driver 'zfs'
        ```

    1. Configure Docker to use the new mount point:

        ```sh
        sudo ros config set rancher.docker.graph /mnt/internal/docker
        ```

    1. Start Docker:

        ```sh
        sudo system-docker start docker
        ```

    1. Confirm it's working:

        ```sh
        docker info
        ```

        ... you should see something like:

        ```log
         Storage Driver: zfs
          Zpool: internal
          Zpool Health: ONLINE
          Parent Dataset: internal/docker
          Space Used By Parent: 53196
        ```

1. Configure second ZFS pool:
    1. Create "external" pool of type `mirror`:

        ```sh
        sudo zpool create -m /mnt/external external mirror /dev/sde /dev/sdf
        ```

    1. Confirm it's working:

        ```sh
        sudo touch /mnt/external/success
        ls -l /mnt/external
        ```
