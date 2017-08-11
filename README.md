
There is Portworx PX-Enterprise and Portworx PX-Dev, This demo uses the px-dev free option which allows me up to 3 nodes running px without running into licensing issues. Also this reference also uses Docker-CE 17.06.0-ce and uses a local etcd setup for running etcd keystore db.   

First I provisioned 2 nodes on AWS running Ubuntu Centos 7.3.    I used the link below to support my configuring node requirements  necessary to run PX-Dev.

https://docs.portworx.com/scheduler/docker/install.html

First get Docker installed.  Here are the steps

1. this step assumes your hosts are running Centos 7.x, and that there is no existing Docker Engine, etc installed. Also remove sudo if your running as root

# sudo yum install -y yum-utils device-mapper-persistent-data lvm2
# sudo yum-config-manager     --add-repo     https://download.docker.com/linux/centos/docker-ce.repo
# sudo yum-config-manager --enable docker-ce-edge
# sudo yum-config-manager --enable docker-ce-testing
# sudo yum makecache fast
# sudo yum install docker-ce -y
# sudo systemctl enable docker
# sudo systemctl start docker
# sudo usermod -aG docker $USER

# docker -v
Docker version 17.06.0-ce, build 02c1d87

2. Next, determine what etcd configuration you want.    If your into installing and maintaining etcd, then you should go for the local install. You can also run etcd within a container as shown in the example below.  You can also opt to run your etcd hosted, like up on compose.io.  They have a free 30 day trial, and its really simple and straightforward to setup.

Local etcd running in containers:  

docker run -d -v /usr/share/ca-certificates/:/etc/ssl/certs -p 4001:4001 -p 2380:2380 -p 2379:2379 \
 --name etcd quay.io/coreos/etcd:v2.3.8 \
 -name etcd1 \
 -advertise-client-urls http://10.0.0.1:2379,http://10.0.0.1:4001 \
 -listen-client-urls http://0.0.0.0:2379,http://0.0.0.0:4001 \
 -initial-advertise-peer-urls http://10.0.0.1:2380 \
 -listen-peer-urls http://0.0.0.0:2380 \
 -initial-cluster-token test-cluster \
 -initial-cluster etcd1=http://10.0.0.1:2380,etcd2=http://10.0.0.2:2380 \
 -initial-cluster-state new

And on Node 2

docker run -d -v /usr/share/ca-certificates/:/etc/ssl/certs -p 4001:4001 -p 2380:2380 -p 2379:2379 \
 --name etcd quay.io/coreos/etcd:v2.3.8 \
 -name etcd2 \
 -advertise-client-urls http://10.0.0.2:2379,http://10.0.0.2:4001 \
 -listen-client-urls http://0.0.0.0:2379,http://0.0.0.0:4001 \
 -initial-advertise-peer-urls http://10.0.0.2:2380 \
 -listen-peer-urls http://0.0.0.0:2380 \
 -initial-cluster-token test-cluster \
 -initial-cluster etcd1=http://10.0.0.1:2380,etcd2=http://10.0.0.2:2380 \
 -initial-cluster-state new




Once I had my 3 instances up and running the first step I performed was to create a 500MB 

root@node1:~# modprobe brd rd_size=500000

You should be able to see a device now named /dev/ram0 - Next prepare the ram drive for use

root@node1:~# mkfs /dev/ram0
mke2fs 1.42.13 (17-May-2015)
Discarding device blocks: failed - Input/output error
Creating filesystem with 499713 1k blocks and 125416 inodes
Filesystem UUID: b487573f-7234-4a22-96b1-429fbdb0cf5d
Superblock backups stored on blocks: 
	8193, 24577, 40961, 57345, 73729, 204801, 221185, 401409

Allocating group tables: done                            
Writing inode tables: done                            
Writing superblocks and filesystem accounting information: done 

Lets mount the prepared newly prepared device and lets see the ram drive as a useable File System in action

root@node1:~# mount /dev/ram0 /mnt/tmpfs
root@node1:~# df -h
Filesystem                  Size  Used Avail Use% Mounted on
.
.
/dev/ram0                   473M  2.3M  446M   1% /mnt/tmpfs


For comparison, lets try creating a file under our hosts /tmp directory and call it test2.  You'll notice the command 
generates the target (of=/tmp/test2) using the /dev/zero special file to create a blank output into the /tmp/test2 file using 1M block sizes 
looped 100 times and then asks dd to sync all the writes prior to exiting - here is a link explaining the offlag options - https://romanrm.net/dd-benchmark

root@node1:~# dd if=/dev/zero of=/tmp/test2 bs=1M count=100 oflag=dsync
100+0 records in
100+0 records out
104857600 bytes (105 MB, 100 MiB) copied, 0.822427 s, 127 MB/s

root@node1:~# ls -lh /tmp
total 101M
-rw-r--r-- 1 root root 100M Aug  1 11:58 test2

Not bad considering this is run on a macbook air virtual box VM ~ 127MB/s throughput when writing out that file.

Now lets try the same command on the ram drive we created earlier, except we will change the of= to be created on the mounted File System called /mnt/tmpfs and the file name will be test1

root@node1:~# dd if=/dev/zero of=/mnt/tmpfs/test1 bs=1M count=100 oflag=dsync
100+0 records in
100+0 records out
104857600 bytes (105 MB, 100 MiB) copied, 0.25881 s, 405 MB/s

root@node1:~# ls -lh /mnt/tmpfs/test1 
-rw-r--r-- 1 root root 100M Aug  1 11:57 /mnt/tmpfs/test1

Wow, thats a pretty good improvement ~3.5x throughput improvements !  405MB/s compared to 127MB/s

If you going to continue the steps in this lab, then you'll want to start over and clean /clear any storage configuration done in the earlier steps.

#umount /mnt/tmpfs
#rmmod brd

Re-make the ram disk any size you want, however, this is static memory, so you'll want to pick a size that leaves you enough remaining memory to be able keep your host running

# modprobe brd rd_size=500000   

NOTE:  DO NOT mkfs or mount the newly created ram drives as done earlier.   

The idea for this speed test was to find out if there are improvements that could be gained using a faster pluggable storage option for container workloads that require persitant backend data volumes. It seems there are possibilities to find speed improvements based on our tests results above, however, the ram drive solution does not persist data upon reboots, so thats an issue we'll have to solve later.

I chose to use Portworx PX to try and manage and present my ram drives as a pluggable storage mount point for use with containers across a cluster of hosts running Docker.  

Portworx PX provides an option for you to define your host storage devices into the PX storage cluster by adding an entry into a  config.json located under /etc/pwx on each host.  As you can see in my config.json example below, I have edited each of my hosts to include my ram drives on each host so they will be shared out to the entire storage cluster so that containers that need access to storage volumes, they will have access to some fast available storage to consume.  

A couple of configure notes to keep in mind before configuring PX for your disks.
- PX prepares devices for use, therefore you should not use mkfs, fdisk, etc. to try and prepare your storage devices.  Also you should not try to mount it via mount or via fstab
- Do not try to add in existing host mount points to px via the config.json or CLI, as PX prepares each storage device as if it were a new share, thus data loss is likely to occur

Here is my /etc/pwx/config.json for comparison. A couple of comments about my config example 
 	1. the 'clusterid' can be any string, but its typically assigned and known to PX.
 	2. the kvdb entries are created when installing etc during my preparation for installation of PX installation.  
	3. Also notice the entry created for my recently created /dev/ram0 device.

root@node1:~# cat /etc/pwx/config.json 
{
  "clusterid": "5ac2ed6f-7e4e-4e1d-8e8c-3a6df1fb61a5",
  "kvdb": [
      "etcd:http://etcd58.aws-us-east-1-memory.24.dblayer.com:2379",
      "etcd:http://etcd90.aws-us-east-1-memory.23.dblayer.com:2379",
      "etcd:http://etcd167.aws-us-east-1-memory.21.dblayer.com:2379"
    ],
  "storage": {
    "devices": [
      "/dev/xvdb"
    ]
  }
}

Now lets run PX in a container and see how the cluster creates the ram drive across the entire 3 node cluster.


6226dbcc34507de8 | started | etcd58.aws-us-east-1-memory.24.dblayer.com  | http://10.10.132.132:2380 | http://10.10.132.132:2379 |
| 931845a014aaec32 | started | etcd90.aws-us-east-1-memory.23.dblayer.com  | http://10.10.132.130:2380 | http://10.10.132.130:2379 |
| 9c91e289f86c3f4e | started | etcd167.aws-us-east-1-memory.21.dblayer.com | http://10.10.132.131:2380 | http://10.10.132.131:2379 |
