# ETCD Cluster with `Ansible` and `Vagrant` 

Prerequisites :

- [vagrant](https://www.vagrantup.com/downloads.html)
- [ansible](https://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html)

# Issue the following command and go for a coffee it will be done shortly :
```
$ ansible-playbook -i hosts deploy-etcd.yaml
```
and boom, thats it!

<p align="center">
  <img width="460" height="300" src="simple-etcd-cluster.jpg">
</p>

# Manual Installation
First of all lets confirm what we have:

|Role|FQDN|IP|
|----|----|----|
|ETCD Cluster Node|etcd01.gwf.me|192.168.56.111|
|ETCD Cluster Node|etcd02.gwf.me|192.168.56.112|
|ETCD Cluster Node|etcd03.gwf.me|192.168.56.113|
> Well we have virtualbox so the IP range could be familiar to anyone 
## On all etcd nodes

> Perform all commands logged in as **root** / **sudo** user

##### Download etcd from github.com and put it on a proper folder
```
{
  ETCD_VER=$(git ls-remote --tags https://github.com/etcd-io/etcd/ | sort -t '/' -k 3 -V | grep -v "\^{}" | tail -1 | awk -F/ '{print $3}')
  cd /tmp && wget -q --show-progress "https://github.com/etcd-io/etcd/releases/download/${ETCD_VER}/etcd-${ETCD_VER}-linux-amd64.tar.gz"
  tar zxf etcd-${ETCD_VER}-linux-amd64.tar.gz
  mv etcd-${ETCD_VER}-linux-amd64/etcd* /usr/local/bin/
  rm -rf etcd-${ETCD_VER}-linux-amd64
}
```

##### Create systemd unit file for etcd service on each servers
> Set NODE_IP to the correct IP of the machine where you are running this
```
{
  NODE_IP=$(hostname -i)
  ETCD_HOSTNAME=$(hostname -s)
  ETCD1_IP="192.168.56.111"
  ETCD2_IP="192.168.56.112"
  ETCD3_IP="192.168.56.113"
}
cat <<EOF >/etc/systemd/system/etcd.service
[Unit]
Description=etcd
[Service]
Type=notify
ExecStart=/usr/local/bin/etcd \\
  --name ${ETCD_HOSTNAME} \\
  --initial-advertise-peer-urls http://${NODE_IP}:2380 \\
  --listen-peer-urls http://${NODE_IP}:2380 \\
  --advertise-client-urls http://${NODE_IP}:2379 \\
  --listen-client-urls http://${NODE_IP}:2379,http://127.0.0.1:2379 \\
  --initial-cluster-token etcd-cluster-1 \\
  --initial-cluster etcd1=http://${ETCD1_IP}:2380,etcd2=http://${ETCD2_IP}:2380,etcd3=http://${ETCD3_IP}:2380 \\
  --initial-cluster-state new
Restart=on-failure
RestartSec=5
[Install]
WantedBy=multi-user.target
EOF
```
> Note: Please mind the `\\` , that is for escapeing `\`
##### Enable and Start etcd service
```
{
  systemctl daemon-reload 
  systemctl enable --now etcd 
  systemctl status -l etcd.service
}
```

#### Verify ETCD cluster status
> Below command could be issued on any server of our cluster
```
ETCDCTL_API=3 etcdctl --endpoints=http://127.0.0.1:2379 member list
```
#### Create and verify some key-value pair
```
ETCDCTL_API=3 etcdctl put name1 Arash
ETCDCTL_API=3 etcdctl put name2 Giti
ETCDCTL_API=3 etcdctl put name3 Rostam
ETCDCTL_API=3 etcdctl put name4 Shirin
ETCDCTL_API=3 etcdctl put name5 Kourosh
```
now we can check it like below :
```
ETCDCTL_API=3 etcdctl get name3 # the value of key:name3 should be the output
ETCDCTL_API=3 etcdctl get name1 name4 # lists range name1 to name 4
ETCDCTL_API=3 etcdctl get --prefix name # lists all keys with name prefix
```
