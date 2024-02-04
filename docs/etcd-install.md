# Etcd Installation Guide for Ubuntu

Due to the following [issue](https://github.com/canonical/microk8s/issues/3227), the performance of microk8s degrades 
significantly over time. Leaving the system non-functional after approximately 2 weeks of uptime. To resolve this issue,
I have decided to replace the default key-value store used by microk8s (k8s-dqlite), with a more reliable alternative etcd.

## How to guide

I am trying to use snaps as the preferred package management software for Ubuntu, and fortunately, etcd is deployable 
via this mechanism. Unfortunately, documentation on the configuration process of etcd after the install is lacking, and 
the only relevant documentation I could find was [here](https://github.com/adamelliotfields/notes/blob/master/containers/2018-02-04-setting-up-an-etcd-cluster-with-ubuntu-snappy.md)

## 1. Install the snap

`sudo snap install etcd`

## 2. If you want to use non-standard directories for your data and write-ahead-logs (Optional)

You will need to modify the AppArmor profile for the etcd snap to allow it to Read/Write in the non-standard directories.
Before making any changes, create a backup of the existing AppArmor profile, then open the AppArmor profile for editing 
using a text editor and adjust the rules to allow ETCD to work.

```
sudo cp /var/lib/snapd/apparmor/profiles/snap.etcd.etcd /var/lib/snapd/apparmor/profiles/snap.etcd.etcd.bak
sudo vi /var/lib/snapd/apparmor/profiles/snap.etcd.etcd
```

In my deployment, I have two separate mount points, that I want to use. The first `/mnt/ssd-raid` points to a RAID Array
of SSD Drives that I will use for the etcd **data** directory. The other is `/mnt/nvme-raid` that points to a RAID Array of 
NVMe drives that I will use for the etcd write-ahead-log. Locate the section in the profile that defines the rules for 
file access, then add the following lines:

**NOTE**: Adjust the directories to match your mount point configuration.

```
profile "snap.etcd.etcd" (attach_disconnected,mediate_deleted) {
  #include <abstractions/base>
  #include <abstractions/consoles>
  #include <abstractions/openssl>

  #### ADD THE FOLLOWING LINES #### 
  # Grant permissions on the RAID mount points  
  /mnt/ssd-raid/** rw,
  /mnt/nvme-raid/** rw,
  /mnt/ssd-raid/etcd/data/member/snap/db wk,
  /mnt/nvme-raid/etcd/** wk,
```

After modifying the AppArmor profile, reload the profiles to apply the changes and restart the etcd snap to ensure that 
the changes take effect:

```
  sudo apparmor_parser -r /var/lib/snapd/apparmor/profiles/snap.etcd.etcd
  sudo snap restart etcd
  ```

## 3. Configure etcd

   ```
   sudo cp /snap/etcd/current/etcd.conf.yml.sample /var/snap/etcd/common/etcd.conf.yml
   sudo vi /var/snap/etcd/common/etcd.conf.yml
   ```

   Make sure the following properties are set

   ```
   # Human-readable name for this member.
   name: 'etcd-node00'

   # Path to the data directory.
   data-dir: /mnt/ssd-raid/etcd/data

   # Path to the dedicated wal directory.
   wal-dir: /mnt/nvme-raid/etcd/wal

   # List of comma separated URLs to listen on for peer traffic.
   listen-peer-urls: http://<NODE IP ADDRESS>:2380

   # List of comma separated URLs to listen on for client traffic.
   listen-client-urls: http://<NODE IP ADDRESS>:2379,http://localhost:2379

   # List of this member's peer URLs to advertise to the rest of the cluster.
   # The URLs needed to be a comma-separated list.
   initial-advertise-peer-urls: http://<NODE IP ADDRESS>:2380

   # List of this member's client URLs to advertise to the public.
   # The URLs needed to be a comma-separated list.
   advertise-client-urls: http://<NODE IP ADDRESS>:2379
   
   # Initial cluster configuration for bootstrapping.
   initial-cluster: etcd-node00=http://etcd-node00:2380,etcd-node01=http://etcd-node01:2380,etcd-node02=http://etcd-node02:2380

   # Reject reconfiguration requests that would cause quorum loss.
   strict-reconfig-check: true
   ```

## 4. Start the service & check the status

After you have finished make the necessary changes to the configuration file. You will need to restart the etcd service 
in order for the changes to take effect.

  ```
  sudo systemctl start snap.etcd.etcd.service
  sudo systemctl status snap.etcd.etcd.service
  ```

## 5. Validate etcd works

After you have made changes to all the etcd nodes and restarted them. You can verify the installation with the 
following commands. 

First, you will want to confirm that all the etcd nodes have joined the same quorum

 ```
 ETCDCTL_API=3 etcdctl member list -w="table"
+------------------+---------+-------------+-------------------------+--------------------------+
|        ID        | STATUS  |    NAME     |       PEER ADDRS        |       CLIENT ADDRS       |
+------------------+---------+-------------+-------------------------+--------------------------+
|  666724d0e31eb17 | started | etcd-node01 | http://etcd-node01:2380 | http://192.168.1.81:2379 |
| 146b6ccebdf8d52a | started | etcd-node02 | http://etcd-node02:2380 | http://192.168.1.82:2379 |
| 51adb0a92ca84265 | started | etcd-node00 | http://etcd-node00:2380 | http://192.168.1.80:2379 |
+------------------+---------+-------------+-------------------------+--------------------------+
 ```

Next, you can run some basic commands to store and retrieve data from etcd to confirm it is functioning as expected.

  ```
  etcdctl put /message "Hello World"
  etcdctl put foo "Hello World!"

  etcdctl get /message
  etcdctl get foo
  ```

## 6. Troubleshoot (if needed)

```
  sudo journalctl -u snap.etcd.etcd.service  # For basic stuff
  journalctl -xe | grep etcd  # For file permission / AppArmor issues
 ```
