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
