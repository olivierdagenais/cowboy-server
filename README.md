# cowboy-server
Configuration files for a RancherOS deployment

## Installing RancherOS
Links
* https://rancher.com/docs/os/v1.x/en/installation/
* https://rancher.com/docs/os/v1.x/en/installation/server/install-to-disk/

First, we need a Cloud-Config file, mostly to bring in the SSH public key(s).
1. Create a `cloud-config.yml` file with these lines:
    ```
    #cloud-config
    ssh_authorized_keys:
    ```
1. For each workstation:
    1. Create SSH key:
        ```
        ssh-keygen -f ~/.ssh/cowboy_rsa -C $COMPUTERNAME@cowboy
        ```
    2. Add the following to the workstation's `~/.ssh/config` file (notice the second line is indented by 2 spaces):
        ```
        Host cowboy
          IdentityFile ~/.ssh/cowboy_rsa
        ```
    3. Copy the contents of the `/.ssh/cowboy_rsa.pub` file and paste it in on a new line in the `cloud-config.yml` file, prefixing with 2 spaces, a hyphen, and a space.  Example:
        ```
          - ssh-rsa AAA...
        ```
1. Configure the host name by inserting this as the 2nd line of `cloud-config.yml`:
    ```
    hostname: cowboy
    ```
1. Boot RancherOS (I had previously configured a USB storage device with [UNetbootin](https://unetbootin.github.io/))
1. Record what storage device is attached where; both of these commands will give you information about what storage devices are connected, how big they are and what partitions might already exist on them:
    ```
    sudo fdisk -l
    sudo parted -l
    ```
    ...this will inform what the destination parameter (`-d`) will take as a value for the upcoming `ros install` command.
1. Install!
    ```
    sudo ros install -c https://raw.githubusercontent.com/olivierdagenais/cowboy-server/main/cloud-config.yml -d /dev/sda
    ```
    If you are installing from USB-based storage and it fails right away with:
    ```
    ERRO[0000] Failed to get boot iso: stat /dev/sr0: no such file or directory
    There is no boot iso drive, terminate the task
    FATA[0000] Failed to run install
    err="stat /dev/sr0: no such file or directory"
    ```
    ...then you might be hitting [#2241](https://github.com/rancher/os/issues/2241) (affects version 1.5.0), so try this workaround:
    ```
    sudo mkdir /dev/sr0
    ```
    ...and run the install command again.
