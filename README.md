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
