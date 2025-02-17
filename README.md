# lkeluks
Automatically convert LKE worker nodes into a LUKS encrypted root filesystem with Tang/Clevis Unlocking

*Warning: This is an unofficial capability that is not currently a supported use case by Linode.  Use at your own risk*

Node pool must be provisioned with a large secondary "RAW" custom disk and the main boot disk at the minimum 15GB size.  This can only be provisioned usin the Linode API.  As this is a DaemonSet any recycled, manually scaled, or autoscaled nodes will get picked up and secured.  Example node pool creation via CLI :

```
$ linode-cli lke pool-create --count 1 --type g6-standard-4 --disks '[{"size": 148840,"type": "raw"}]' <LKE_clusterID>
```

To apply to your LKE cluster:

```
kubectl apply -f daemonset-lke-luks.yaml
```

DaemonSet will :
* Update base Debian 11 packages
* Changes TCP congestion control to BBR
* Increase eth0 NIC queue to max allowed
* One node at a time drain, convert to LUKS, reboot, uncordon, and then wipe original in the clear disk.
* Nodes are then labeled with luks=enabled allowing you to schedule work only on workers that have been secured.

luks-setup.sh script will perform operations in following manner:
* This script is a bash script that can be used to convert an existing Linode Kubernetes Engine (LKE) base Debian install into an encrypted root filesystem on `/dev/sdb`.

* The script first sets some environment variables, including the KUBECONFIG, PATH, and LUKS_KEY variables. It then checks that `/dev/sdb` is a `raw` disk and is not already being used for an encrypted filesystem.

* The script waits for all other nodes to leave drain status before moving the current node into maintenance mode and shutting down running pods. It then creates a disk layout for 2GB `/boot` and the remainder for the root filesystem. The script then upgrades the system and installs required packages, including `joe`, `net-tools`, `cryptsetup-initramfs`, and `rsync`.

* The script then formats `/dev/sdb2` and `/dev/sdb3` as `ext4` filesystems, and formats `/dev/sdb3` as a LUKS2 encrypted volume with the LUKS key specified in `/boot/keyfile`. It mounts `/dev/mapper/secure` on `/mnt` and syncs the contents of the root filesystem to the encrypted volume using rsync.

* The script then chroots into `/mnt` and updates `/etc/fstab` and `/etc/crypttab` to mount /dev/mapper/secure on / and to use the LUKS key specified in /boot/keyfile. It updates the grub configuration and installs the grub bootloader to /dev/sdb. Finally, it installs the Linode CLI, changes the configuration profile to direct disk boot, and reboots the system.

To determine which nodes have been secured :

```
$ kubectl get nodes -L luks
NAME                           STATUS   ROLES    AGE   VERSION   LUKS
lke75642-117585-63434c5cc0b2   Ready    <none>   43h   v1.23.6   enabled
lke75642-117585-6343753083ed   Ready    <none>   41h   v1.23.6   enabled
lke75642-117585-63437530ab29   Ready    <none>   41h   v1.23.6   enabled
lke75642-117619-63440cb6e434   Ready    <none>   30h   v1.23.6   enabled
lke75642-117619-6345a0727229   Ready    <none>   12m   v1.23.6   enabled
lke75642-117619-6345b080de99   Ready    <none>   48m   v1.23.6   enabled
lke75642-117619-6345b62e9f08   Ready    <none>   23m   v1.23.6   enabled
lke75642-117619-6345b62eca30   Ready    <none>   24m   v1.23.6   enabled
lke75642-117619-6345b62eef86   Ready    <none>   23m   v1.23.6   enabled
```

To change any of the behavior including the TANG server URL you must fork this repo, modify the setup script, and point the DaemonSet at the new URL with your customized version.  For a production environment you would want to self-host a TANG server which only accepts requests from your LKE worker node IPs.

Tested up through LKE with Kubernetes 1.24.
