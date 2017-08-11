
There is Portworx PX-Enterprise and Portworx PX-Dev, This demo uses the px-dev free option which allows me up to 3 nodes running px without running into licensing issues. Also this reference also uses Docker-CE 17.06.0-ce and uses a local etcd setup for running etcd keystore db.   

First I provisioned 2 nodes on AWS running Ubuntu Centos 7.3.    I used the link below to support my configuring node requirements  necessary to run PX-Dev.

https://docs.portworx.com/scheduler/docker/install.html

First get Docker installed.  Here are the steps

1. this step assumes your hosts are running Centos 7.x, and that there is no existing Docker Engine, etc installed. Also remove sudo if your running as root

$ sudo yum install -y yum-utils device-mapper-persistent-data lvm2
$ sudo yum-config-manager     --add-repo     https://download.docker.com/linux/centos/docker-ce.repo
$ sudo yum-config-manager --enable docker-ce-edge
$ sudo yum-config-manager --enable docker-ce-testing
$ sudo yum makecache fast
$ sudo yum install docker-ce -y
$ sudo systemctl enable docker
$ sudo systemctl start docker
$ sudo usermod -aG docker $USER

$ docker -v
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

Try these commands on each AWS instnace to confirm that the etcd cluster is working properly

$ curl -L http://127.0.0.1:2379/health    -   you should see a 'healthy' response

$ curl -L http://127.0.0.1:2379/v2/members - you should see both of your etcd instances returned



Once I have etcd up and running, I can begin the installation of px-dev software

NOTE:  DO NOT mkfs or mount the newly created ram drives as done earlier.   


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
