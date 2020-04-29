Deploy Kubernetes 1.16. & CentOS7 + GlusterFs in BareMetal
==========================================================

Requirements:
-------------
* 6 Virtual Machines or physical nodes. (Example)
  * master01.k8s.labs.vass.es
    * cpu: 4
    * memory:3024
    * disks:
      * sda 30 GiB
  * master02.k8s.labs.vass.es
    * cpu: 4
    * memory:3024
    * disks:
      * sda 30 GiB
  * master03.k8s.labs.vass.es
    * cpu: 4
    * memory:3024
    * disks:
      * sda 30 GiB
  * worker01.k8s.labs.vass.es
    * cpu: 4
    * memory:4000
    * disks:
      * sda 40 GiB
  * worker02.k8s.labs.vass.es
    * cpu: 4
    * memory:4000
    * disks:
      * sda 40 GiB
  * worker03.k8s.labs.vass.es
    * cpu: 4
    * memory:4000
    * disks:
      * sda 40 GiB
  * infra01.k8s.labs.vass.es
    * cpu: 4
    * memory:4024
    * disks:
      * sda 30 GiB
      * sdb 50 GiB
  * infra02.k8s.labs.vass.es
    * cpu: 4
    * memory:4024
    * disks:
      * sda 30 GiB
      * sdb 50 GiB
  * infra03.k8s.labs.vass.es
    * cpu: 4
    * memory:4024
    * disks:
      * sda 30 GiB
      * sdb 50 GiB  

Annotations:
-----------
* Calico as Network: calico-rr
* kube_proxy_mode: 'iptables'
  * You can change to ipvs changing all.yaml
* GlusterFS as Storage Solution

Diagram:
-------

![alt text](https://github.com/GIT-VASS/kubernetesSpray-v1.16.6-glusterfs/blob/master/InstallationOnBareMetal/DocOnBareMetal/RHV_Vass/img/Diagram.jpg)


* All the hostnames must be resolved by a DNS or set hostnames in the /etc/hosts of all the VMs. We will use a DNS

Prepare the node bastion:
------------------------------

Login in the bastion, in our case master01.k8s.labs.vass.es will be the bastion:
```
ssh root@master01.k8s.labs.vass.es
```

Install git:
```
yum install git -y
```

Clone our project from github in root directory:
```
cd /root
git clone https://github.com/GIT-VASS/kubernetesSpray-v1.16.6-glusterfs.git
```

Configure your inventory and bastion:

*In our case, we are going to install 6 workers, but  the first three will be infra nodes, so we call them infra nodes, the nodes where will deploy infraestructure apps (registry, ingress...something not cover in these guide). You can deploy just 3 master and 3 workers, just remember, delete this node references from "/kubernetesSpray-v1.16.6-glusterfs/InstallationOnBareMetal/ansible/inventories/rhvVass/inventory.ini.j2" and you must remember than in the workerone_k8s_hostname, the workertwo_k8s_hostname and the workerthree_k8s_hostname the glusterFS will be deployed, you can change this parameters changing topology.json*


Add your hostnames and IPs. In case you have not a DNS, you can use hostnames like "master-one.192.168.66.2.xip.io", check xip.io project.

```
cd /root/kubernetesSpray-v1.16.6-glusterfs/InstallationOnBareMetal/ansible/inventories/rhvVass/group_vars
cp allExample.yaml  all.yaml
vi  all.yaml
vi ../bastion
```

Execute bastion.sh to install some requirements in the bastion:
```
/root/kubernetesSpray-v1.16.6-glusterfs/InstallationOnBareMetal/InstallationTools/bastion.sh
```

Copy the SSH key to all the nodes change with your hostanames:
```
for host in master01.k8s.labs.vass.es \
            master02.k8s.labs.vass.es \
            master03.k8s.labs.vass.es \
            worker01.k8s.labs.vass.es \
            worker02.k8s.labs.vass.es \
            worker03.k8s.labs.vass.es \
            infra01.k8s.labs.vass.es \
            infra02.k8s.labs.vass.es \
            infra03.k8s.labs.vass.es;\
            do ssh-copy-id $host; \
            done
```

Launch the next ansible playbook to prepare the bastion:
```
cd /root/kubernetesSpray-v1.16.6-glusterfs/InstallationOnBareMetal/ansible/

ansible-playbook  -i inventories/rhvVass/bastion playbooks/preparebastion.yaml
```


Prepare the rest of nodes:
--------------------------
```
ansible-playbook  -i /root/kubernetes_installation/inventory/mycluster/inventory.ini playbooks/preparationnodes.yaml
```

Start kubernetes installation:
------------------------------
```
cd /root/kubernetes_installation
pip install --user -r requirements.txt
ansible-playbook -i inventory/mycluster/inventory.ini  cluster.yml
```

Details of the successfully installation:

![alt text](https://github.com/GIT-VASS/kubernetesSpray-v1.16.6-glusterfs/blob/master/InstallationOnBareMetal/DocOnBareMetal/RHV_Vass/img/InstallationAnsible.jpg)

Prepare dashboard for Kubernetes:
--------------------------------

Check your cluster info where you will find the URL of yor dashboard:
```
 kubectl cluster-info
```

Create a service-account, a clusterrolebinding and clusterrolebinding:
```
 kubectl create serviceaccount dashboard-admin-sa
 kubectl create clusterrolebinding dashboard-admin-sa
 kubectl create clusterrolebinding dashboard-admin-sa  --clusterrole=cluster-admin --serviceaccount=default:dashboard-admin-sa
```

Check the secret that you can use to login in your dashboard:
```
 kubectl get secrets
 kubectl describe secret dashboard-admin-sa-token-svhm2
```

Deploy GlusterFS in Kubernetes:
------------------------------
Check that all your nodes has glusterfs client installed:

```
cd /root/kubernetes_installation
ansible  -i inventory/mycluster/inventory.ini all  -a "glusterfs --version"
```
*Example exit: worker01.k8s.labs.vass.es | CHANGED | rc=0 >> glusterfs 7.2*

Create the topology file for glusterfs, where you should add the hostnames, IPS, and disk device for Storage, as it is automatically completed with the variables from the file all.yaml, just check that everything is correct. Maybe you have to change the disk (device reference), as it must not be the disk where you installed the OS, it must be a totally free disk not formated.
```
cd /root/glusterfs_installation/deploy/

vi topology.json

cat topology.json (Exmple)
{
  "clusters": [
    {
      "nodes": [
        {
          "node": {
            "hostnames": {
              "manage": [
                "worker01.k8s.labs.vass.es"
              ],
              "storage": [
                "X.X.X.X"
              ]
            },
            "zone": 1
          },
          "devices": [
            "/dev/sdb"
          ]
        },
        {
          "node": {
            "hostnames": {
              "manage": [
                "worker02.k8s.labs.vass.es"
              ],
              "storage": [
                "X.X.X.X"
              ]
            },
            "zone": 1
          },
          "devices": [
            "/dev/sdb"
          ]
        },
        {
          "node": {
            "hostnames": {
              "manage": [
                "worker03.k8s.labs.vass.es"
              ],
              "storage": [
                "X.X.X.X"
              ]
            },
            "zone": 1
          },
          "devices": [
            "/dev/sdb"
          ]
        }
      ]
    }
  ]
}
```

Deploy glusterfs:

***Before to deploy decide the user-key variable and the admin-key variable***

*user-key: Secret string for general heketi users. heketi users have access to only Volume APIs. Used in dynamic provisioning. This is a required argument.*

*admin-key:Secret string for heketi admin user. heketi admin has access to all APIs and commands. This is a required argument.*


./gk-deploy -g \
 --user-key <MyUserStrongKey> \
 --admin-key <MyAdminStrongKey> \
 -l /tmp/heketi_deployment.log \
 -v topology.json

```
./gk-deploy -g --user-key kubernetes --admin-key kubernetesadmin -l /tmp/heketi_deployment.log -v topology.json
...
Do you wish to proceed with deployment?
Yes
...
...
...Deployment complete!
```

Check HEKETI status:

```
export HEKETI_CLI_SERVER=$(kubectl get svc/heketi --template 'http://{{.spec.clusterIP}}:{{(index .spec.ports 0).port}}')
echo $HEKETI_CLI_SERVER
curl $HEKETI_CLI_SERVER/hello

...Hello from Heketi
```

Create a StorageClass:
-----------------------
Decode the ${ADMIN_KEY} used before:

SECRET_KEY=`echo -n "${ADMIN_KEY}" | base64`
```
SECRET_KEY=`echo -n "kubernetesadmin" | base64`
```

Create a secret in order to access to heketi from the StorageClass:
```
cat << EOF | kubectl apply -f -
apiVersion: v1
kind: Secret
metadata:
  name: heketi-secret
  namespace: default
data:
  # base64 encoded password. E.g.: echo -n "mypassword" | base64
  key: ${SECRET_KEY}
type: kubernetes.io/glusterfs
EOF

...
...
...
secret/heketi-secret created
```


Set HEKETI_CLI_SERVER variable before to create the StorageClass:
```
export HEKETI_CLI_SERVER=$(kubectl get svc/heketi --template 'http://{{.spec.clusterIP}}:{{(index .spec.ports 0).port}}')

echo $HEKETI_CLI_SERVER
...
http://10.233.60.171:8080
```
Create the StorageClass:

```
cat << EOF | kubectl apply -f -
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: glusterfs-storage
provisioner: kubernetes.io/glusterfs
parameters:
  resturl: "${HEKETI_CLI_SERVER}"
  restuser: "admin"
  secretNamespace: "default"
  secretName: "heketi-secret"
  volumetype: "replicate:3"
EOF

```



Configure heketi-client if you want to communicate with heketi(Optional):
-------------------------------------------------------------------------
Download heketi client:
```
mkdir -p /root/heketi-client
cd /root/heketi-client
yum install wget -y
curl -s https://api.github.com/repos/heketi/heketi/releases/latest   | grep browser_download_url   | grep linux.amd64   | cut -d '"' -f 4   | wget -qi -
for i in `ls | grep heketi | grep .tar.gz`; do tar xvf $i; done
cd heketi/
cp heketi-cli /usr/sbin
 heketi-cli --version
heketi-cli v9.0.0
```

Now you can communicate with your glsuter:

heketi-cli cluster list --user admin --secret <$ADMIN_KEY>

```
heketi-cli cluster list --user admin --secret kubernetesadmin

Clusters:
Id:b6aea255961a90ddc2c720e7466e224f [file][block]


heketi-cli cluster info  b6aea255961a90ddc2c720e7466e224f --user admin --secret kubernetesadmin
Cluster id: b6aea255961a90ddc2c720e7466e224f
Nodes:
5ca1630d7f0fc21685ca44c7961c4d3d
6ecaab134daa9af3604cfb50428ab302
f4dc9b76cf31fa9600457c97cdc251a6
Volumes:
a32c10d23f91ce94a991e6b15caa12c2
Block: true

File: true



heketi-cli topology info

Cluster Id: b6aea255961a90ddc2c720e7466e224f

    File:  true
    Block: true

    Volumes:

        Name: heketidbstorage
        Size: 2
        Id: a32c10d23f91ce94a991e6b15caa12c2
        Cluster Id: b6aea255961a90ddc2c720e7466e224f
        Mount: 10.0.5.34:heketidbstorage
        Mount Options: backup-volfile-servers=10.0.5.36,10.0.5.35
        Durability Type: replicate
        Replica: 3
        Snapshot: Disabled
```


Configure StorageClass glusterfs-storage as your default StorageClass:
----------------------------------------------------------------------


Check the name of your storageclass:
```
k get StorageClass
NAME                PROVISIONER               AGE
glusterfs-storage   kubernetes.io/glusterfs   10m
```

Patch the StorageClass as the default StorageClass:

*kubectl patch storageclass <your-class-name> -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'*

```
kubectl patch storageclass glusterfs-storage -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'

...
...
...
storageclass.storage.k8s.io/glusterfs-storage patched
```


Test your glusterfs Creating a pvc:
-----------------------------------
Create the pvc file yaml (testglusterfs.yaml):
```
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: testglusterfs
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 250Mi
  storageClassName: glusterfs-storage
```

Create the pvc in Kubernetes and check that the pvc bound a pv:
```
k create -f testglusterfs.yaml
k get pvc
...
...
...
NAME            STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS        AGE
testglusterfs   Bound    pvc-a7d23a03-c3b7-45cc-adc1-9974a68982c6   1Gi        RWX            glusterfs-storage   5d1h


k get pv
glusterfs-storage            3d22h
pvc-a7d23a03-c3b7-45cc-adc1-9974a68982c6   1Gi        RWX            Delete           Bound    default/testglusterfs
```
