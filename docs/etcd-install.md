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

## 2. Confirm it is installed and enabled

`sudo snap info etcd`

## 3. Configure etcd

   ```
   sudo cp /snap/etcd/current/etcd.conf.yml.sample /var/snap/etcd/common/etcd.conf.yml
   sudo vi /var/snap/etcd/common/etcd.conf.yml
   ```

   Make sure these properties are set

   listen-client-urls: http://127.0.0.1:2379
   advertise-client-urls: http://127.0.0.1:2379

## 4. Start the service & check the status

  ```
  sudo systemctl start snap.etcd.etcd.service
  sudo systemctl status snap.etcd.etcd.service
  ETCDCTL_API=3 etcdctl member list
  ```

## 5. Validate etcd works

  ```
  etcdctl put /message "Hello World"
  etcdctl put foo "Hello World!"

  etcdctl get /message
  etcdctl get foo
  ```

## 6. Troubleshoot (if needed)

`sudo journalctl -u snap.etcd.etcd.service`