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

Because Snap overwrites the AppArmor file `/var/lib/snapd/apparmor/profiles/snap.etcd.etcd` on refresh, you will need to create a script that inserts your custom rules just before the final `}` in that file and reload the AppArmor profile. This will ensure that the necessary changes aren't lost when applying updates to Ubuntu.

In order to automate this process, we will use a `systemd Path` unit to watch for changes and run this script whenever changes to the `/var/lib/snapd/apparmor/profiles/snap.etcd.etcd` are detected. The script will need to modify the default AppArmor profile for the etcd snap to allow it to Read/Write in the non-standard directories.

In my deployment, I have two separate mount points, that I want to use. The first `/mnt/ssd-raid` points to a RAID Array
of SSD Drives that I will use for the etcd **data** directory. The other is `/mnt/nvme-raid` that points to a RAID Array of 
NVMe drives that I will use for the etcd write-ahead-log. Therefore to allow these directories to be used, the following permissions need to be added to the default AppArmor profile.

**NOTE**: Adjust the directories to match your mount point configuration.

```
profile "snap.etcd.etcd" (attach_disconnected,mediate_deleted) {
  #include <abstractions/base>
  #include <abstractions/consoles>
  #include <abstractions/openssl>
  ...
  #### ADD THE FOLLOWING LINES TO THE END #### 
  # Grant permissions on the RAID mount points  
  /mnt/ssd-raid/** rw,
  /mnt/nvme-raid/** rw,
  /mnt/ssd-raid/etcd/data/member/snap/db wk,
  /mnt/nvme-raid/etcd/** wk,
```

To automate the process requires the following steps:

### Step 1: Create the patch script
Create a script, e.g. `/usr/local/sbin/patch-snap-etcd-apparmor.sh`:

```
#!/bin/bash
PROFILE="/var/lib/snapd/apparmor/profiles/snap.etcd.etcd"
BACKUP="/var/lib/snapd/apparmor/profiles/snap.etcd.etcd.bak"

# Backup original if not exists
if [ ! -f "$BACKUP" ]; then
  cp "$PROFILE" "$BACKUP"
fi

# Insert custom rules just before the last closing brace
# Remove existing custom lines if any, then add fresh
sed -i '/# CUSTOM OVERRIDE START/,/# CUSTOM OVERRIDE END/d' "$PROFILE"


# Insert new custom rules *before* the final closing brace
sed -i -e '$i\
  # CUSTOM OVERRIDE START\
  /mnt/ssd-raid/** rw,\
  /mnt/nvme-raid/** rw,\
  /mnt/ssd-raid/etcd/data/member/snap/db wk,\
  /mnt/nvme-raid/etcd/** wk,\
  # CUSTOM OVERRIDE END' "$PROFILE"

# Reload AppArmor profile
apparmor_parser -r "$PROFILE"
```

Make it executable: `sudo chmod +x /usr/local/sbin/patch-snap-etcd-apparmor.sh`

### Step 2: Create systemd Path unit to watch profile changes
Create `/etc/systemd/system/patch-snap-etcd-apparmor.path` with the following contents:

```
[Unit]
Description=Watch snap.etcd.etcd AppArmor profile for changes

[Path]
PathChanged=/var/lib/snapd/apparmor/profiles/snap.etcd.etcd

[Install]
WantedBy=multi-user.target
```


### Step 3: Create systemd service to run the patch
Create `/etc/systemd/system/patch-snap-etcd-apparmor.service` with the following contents:

```
[Unit]
Description=Patch snap.etcd.etcd AppArmor profile with custom rules

[Service]
Type=oneshot
ExecStart=/usr/local/sbin/patch-snap-etcd-apparmor.sh
```

### Step 4: Enable and start the path watcher

```
sudo systemctl daemon-reload
sudo systemctl enable --now patch-snap-etcd-apparmor.path
```

### Step 5: Manually run once to patch current profile

```
sudo /usr/local/sbin/patch-snap-etcd-apparmor.sh
```

How it works
  - When Snap updates or rewrites the AppArmor profile, systemd sees the file changed.
  - It runs your patch script automatically.
  - The script cleans previous custom rules and injects your additions safely.
  - The profile is reloaded, keeping etcd working with your required permissions.

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
