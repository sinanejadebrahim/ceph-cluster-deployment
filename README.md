# ceph-cluster-deployment
first of all redhat's recommendation is at least 3 servers for your cluster, we're using 3 here too, but you can use 1 or 2 servers depending on your expectation from your cluster, for example if you use 1 server and that server goes down for any reason, you'll go down in history as the guy who used 1 server for storage backend and now that the server is down, the whole company is in crisis mode, so don't be that guy :D <br>
<br>
i'm using ubuntu for our servers and centos7 as our client which we later use.(we're gonna deploy and use ceph rbd block device too), but same steps could be followed for other distros to, not much is gonna change.<br>
first, we need some prerequisites on our servers: <br>
1. Python 3
```
apt install -y python3 
# repeat for all 3 servers
```
2. Systemd
```
# you already have this
```
3. Docker for running containers
```
apt-get install ca-certificates curl gnupg lsb-release
mkdir -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | tee /etc/apt/sources.list.d/docker.list > /dev/null
apt update
apt-get install docker-ce docker-ce-cli containerd.io docker-compose-plugin
# repeat for all 3 servers
```
4. Time synchronization (such as chrony or NTP)
```
apt install ntp -y
# repeat for all 3 servers
```
5. LVM2 for provisioning storage devices
```
# ubuntu20 - 22 already have this
```
now we will install cephadm
```
curl --silent --remote-name --location https://github.com/ceph/ceph/raw/quincy/src/cephadm/cephadm
chmod +x cephadm
./cephadm add-repo --release quincy
./cephadm install
# Confirm that cephadm is now in your PATH by running which
which cephadm
# A successful which cephadm command will return this:
/usr/sbin/cephadm
```
now it's time to bootstrap our cluster, but there is something you need to know.<br>
ceph needs alot of bandwith because it's constantly syncing in the background and clients are accessing their data at the same time, for this reason te recommendation is a 10G network for your frontend which will be used by MONs - MDSs and clients acessing OSDs and a 40G netowrk for your clusters backend which OSDs will use for heartbeat, object replication and recovery traffic. This may improve performance compared to using a single network.but you can skip this step if you don't need it.<br>
to bootstrap your cluster run these command: 
```
# 192.168.1.10 => our first nodes IP address
# 10.10.10.10 => our first nodes second IP which will be used as backend network
cephadm bootstrap --mon-ip 192.168.1.10 --cluster-network 10.10.10.10
# if you don't want a cluster network just run the command without that 
# if you used cluster network make sure all servers are accesible by both IPs (ping them from eachother)
# to this only if you used cluster network
add << cluster_network = {cluster-network/netmask} >> to /etc/ceph/ceph.conf
# cluster_network = {10.10.10.0/24}
```
wait for it to finish, at the end you'll get a address and a username and a password to access ceph dashboard, save that for later.<br>
to ENABLE CEPH CLI run 
```
cephadm shell
```
which will drop you into a container that you can use to manage your cluster<br>
to make sure everything is working fine run << ceph -s >> which will show you some info, we don't care about the output now.<br>
we use all 3 server as mons and hosts, so first thing you need to do is follow these steps:
```
ssh-keygen
ssh-copy-id 192.168.1.10
ssh-copy-id 192.168.1.11
ssh-copy-id 192.168.1.12
ssh-copy-id -f -i /etc/ceph/ceph.pub root@192.168.1.10
ssh-copy-id -f -i /etc/ceph/ceph.pub root@192.168.1.11
ssh-copy-id -f -i /etc/ceph/ceph.pub root@192.168.1.12
# repeat first 4 steps for all servers.

# Tell Ceph that the new node is part of the cluster: 
ceph orch host add host2 192.168.1.11  --labels _admin
# labels is not necessary (will copy client.admin and ceph.conf to this host), also repeat for your third node.
```
now if you run this command you should have all your servers.
```
root@ceph1:~# ceph orch host ls

HOST   ADDR           LABELS  STATUS  
hsot1  192.168.1.10  _admin          
host2  192.168.1.11  _admin             
host3  192.168.1.12
3 hosts in cluster
```
now that we have these working , we will add our additional MONs:
```
ceph orch daemon add mon host2:192.168.1.11
ceph orch daemon add mon host3:192.168.1.12
```
now it's time to add our OSDs:<br>
A storage device is considered available if all of the following conditions are met:<br>
The device must have no partitions.<br>
The device must not have any LVM state.<br>
The device must not be mounted.<br>
The device must not contain a file system.<br>
The device must not contain a Ceph BlueStore OSD.<br>
The device must be larger than 5 GB.<br>
Ceph will not provision an OSD on a device that is not available
```
ceph orch device ls
NAME                  HOST  DATA      DB  WAL
all-available-devices host1 /dev/vdb  -   -
all-available-devices host2 /dev/vdc  -   -
all-available-devices host3 /dev/vdd  -   -
# will show you all disk that are available for ceph to use as osd

ceph orch apply osd --all-available-devices
# to use all available devies as OSDs
```
after that if you run that command again, you'll have an output like this:
```
host1  /dev/sdb  ssd   LOGICAL_VOLUME_600508b1001cgfd46ac50017b93394e8  1000G             9m ago     Insufficient space (<10 extents) on vgs, LVM detected, locked  
host1  /dev/sdc  ssd   LOGICAL_VOLUME_600508b1001c3c4bsfge790f84745d69  1000G             9m ago     Insufficient space (<10 extents) on vgs, LVM detected, locked  
host2  /dev/sdb  ssd   LOGICAL_VOLUME_600508b1001c5desd156cb7abe18cf4d  1000G             9m ago     Insufficient space (<10 extents) on vgs, LVM detected, locked  
host2  /dev/sdc  ssd   LOGICAL_VOLUME_600508b1001c8e2fdef1d5eaad246084  1000G             9m ago     Insufficient space (<10 extents) on vgs, LVM detected, locked  
```
that's it, you have a working ceph cluster, now we are going to use ceph block storage and rbd to add some storage to a client:<br>
```
# first create a pool on your cluster
# ceph osd pool create {pool-name} {pgp-num} [replicated] || erasure
# we create a simple replicated pool with 50 PGs.

ceph osd pool create storage 50 # default is replicated mode

# then we initialzie the pool
rbd pool init <pool-name>

# now you can create a user or use admin keyring to connect to this pool, i'll use admin but use this link to create a user
# https://docs.ceph.com/en/quincy/rbd/rados-rbd-cmds/

# first you need to install ceph-common on your client
apt install ceph-common
# then copy our conf files to the client
scp -r /etc/ceph/ client:/etc/

# then we create a disk on our pool
rbd create storage --size 100G --image-feature layering -p storage
           # disk name                                    # pool name
# then we map this disk to the pool
rbd map storage --name client.admin -p storage

# now we create a DIR to mount this disk
mkdir /mnt/ceph

# format it
mkfs.ext4 -m0 /dev/rbd/storage/storage

# now just mount it and you'll have a 100G disk added your client
mout /dev/rbd/storage /mnt/ceph/

# check to be sure
$ df -h
/dev/rbd0        98G   10G   88G  11% /mnt/ceph

$ lsblk
rbd0   252:0    0   100G  0 disk /mnt/ceph
```
some usefull links for you:
```
https://docs.ceph.com/en/latest/cephadm/host-management/#cephadm-adding-hosts
https://docs.ceph.com/en/latest/rados/configuration/network-config-ref/#cluster-network
https://docs.ceph.com/en/latest/cephadm/services/mon/#deploy-additional-monitors
https://docs.ceph.com/en/latest/cephadm/services/osd/#cephadm-deploy-osds
https://docs.ceph.com/en/quincy/rados/operations/pools/#create-a-pool
https://docs.ceph.com/en/quincy/rbd/rados-rbd-cmds/
https://docs.ceph.com/en/quincy/rados/operations/user-management/#add-a-user
https://docs.ceph.com/en/quincy/start/quick-rbd/
```
