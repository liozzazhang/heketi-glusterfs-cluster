# glusterfs-kubernetes

[![Build Status](https://travis-ci.org/gluster/gluster-kubernetes.svg?branch=master)](https://travis-ci.org/gluster/gluster-kubernetes)

## Preparation
   - Install glusterfs-client on each node: `yum install glusterfs-client`
   - Load kernel mode: `modprobe dm_thin_pool`
   - At least 3 nodes with at least one available block disk. such as /dev/xda, /dev/xdb .
   - Check All available disk can be formatted via `lvcreate` command. You can run `lvcreate /dev/xda` to verify!
   - Label glusterfs node specified in `templetes/glusterfs-deploy.yaml` _nodeSelector_ section.
   
## Installation
This installation is based on helm, so if you don't have a helm environment, please check [here](https://github.com/helm/helm).
### Modify Glusterfs and Heketi Charts
```bash
$ cd templetes/
# label glusterfs node in 'glusterfs-deploy.yaml'
$ kubectl label node server-node-01 storagenode=glusterfs
$ kubectl label node server-node-02 storagenode=glusterfs
$ kubectl label node server-node-03 storagenode=glusterfs
# keep heketi server replica num in 'heketi-deployment.yaml'
kind: Deployment
apiVersion: extensions/v1beta1
metadata:
  name: heketi
  labels:
    glusterfs: heketi-deployment
    heritage: {{.Release.Service | quote }}
    release: {{.Release.Name | quote }}
    chart: "{{.Chart.Name}}-{{.Chart.Version}}"
  namespace: "{{.Release.Namespace}}"
  annotations:
    description: Defines how to deploy Heketi
spec:
  replicas: 1 
...
# if you want change service port, modify in 'heketi-service.yaml'
...
spec:
  selector:
    name: heketi
  ports:
    - name: heketi
      port: 8080
      targetPort: 8080
...

```
### Helm Install Glusterfs and Heketi
```bash
$ cd project root 
$ cd kubernetes-glusterfs-cluster
$ helm install glusterfs --name glusterfs
```
Check pods.
```bash
$ kubectl get pods
NAME                                               READY   STATUS    RESTARTS   AGE
glusterfs-7z8wz                                    1/1     Running   0          23h
glusterfs-tmf9q                                    1/1     Running   0          23h
glusterfs-wcpj7                                    1/1     Running   0          23h
heketi-866cfc978-kv85n                             1/1     Running   0          20h
```
Describe heketi service 
```bash
$ kubectl describe svc heketi
Name:              heketi
Namespace:         library
Labels:            chart=glusterfs-0.1.0
                   deploy-heketi=support
                   glusterfs=heketi-service
                   heritage=Tiller
                   release=glusterfs
Annotations:       description: Exposes Heketi Service
Selector:          name=heketi
Type:              ClusterIP
IP:                10.68.0.80
Port:              heketi  8080/TCP
TargetPort:        8080/TCP
Endpoints:         172.20.3.116:8080
Session Affinity:  None
Events:            <none>

```
Check heketi server status with service clusterIP.
```bash
$ curl http://10.68.0.80:8080/hello
Hello from Heketi
```
Load heketi topology in heketi pod
```bash
$ kubectl exec -ti heketi-866cfc978-kv85n bash
[root@heketi-866cfc978-kv85n /]# heketi-cli topology load --json=/etc/heketi-topology/topology.json
#
# This step is very important, here you may face an issue with key word *NO SPACE*
#
# Solution:
# 1. check your glusterfs node/pod if the number is more than 3. Sometimes, thought you have more than three glusterfs nodes, but only one or two nodes join in the heketi cluster.
$ heketi-cli node list
Id:30e15967ab375bf39ee02a470c4bad7a     Cluster:3b5b82af683b55d7f4f13308935c803c
Id:71887bf22d3bdb6eb2c202d02a718b0c     Cluster:3b5b82af683b55d7f4f13308935c803c
# then delete all node with 'heketi-cli node delete $node_id' then reload topology config
[root@heketi-866cfc978-kv85n /]# heketi-cli topology load --json=/etc/heketi-topology/topology.json
#
# 2. Adding device /dev/xda timeout. Check your glusterfs node device if it can be formatted via 'pvcreate'
# attach to each glusterfs pod to run 'pvcreate'
$ kubectl exec -ti glusterfs-7z8wz sh
sh-4.2# pvcreate /dev/xda
# If all 'pvcreate' command run smoothly, then reload topology config.
[root@heketi-866cfc978-kv85n /]# heketi-cli topology load --json=/etc/heketi-topology/topology.json
Handling connection for 57598
    Found node server-node-01 on cluster 3b5b82af683b55d7f4f13308935c803c
        Adding device /dev/xda ... OK
    Found node server-node-02 on cluster 3b5b82af683b55d7f4f13308935c803c
        Adding device /dev/xda ... OK
    Found node server-node-03 on cluster 3b5b82af683b55d7f4f13308935c803c
        Adding device /dev/xda ... OK
```
Create Heketi Storage to persis data
```bash
[root@heketi-866cfc978-kv85n /]# heketi-cli setup-openshift-heketi-storage
# This command will generate a json file in current directory
$ ls heketi-storage.json
heketi-storage.json
```
Copy Heketi Storage json to helm node.
```bash
$ kubectl cp default/heketi-866cfc978-kv85n:/heketi-storage.json /data/glusterfs/
$ kubectl create -f /data/glusterfs/heketi-storage.json
$ kubectl get pods
NAME                                        READY     STATUS
heketi-storage-copy-job-j9x09               0/1       ContainerCreating
```
When heketi storage job finished, recreate heketi server to persis data.
```bash
$ kubectl delete all,service,jobs,deployment,secret --selector="deploy-heketi"
$ helm upgrade glusterfs /data/glusterfs/
```
Attach heketi server pod to check status
```bash
$ kubectl exec -ti heketi-866cfc978-kv85n bash
[root@heketi-866cfc978-kv85n /]# heketi-cli cluster list
Clusters:
3b5b82af683b55d7f4f13308935c803c

[root@heketi-866cfc978-kv85n /]# heketi-cli volume list
Id:b18fcbeadca4a0f57ed29aa35396d41e    Cluster:3b5b82af683b55d7f4f13308935c803c    Name:heketidbstorage

[root@heketi-866cfc978-kv85n /]# heketi-cli cluster info 3b5b82af683b55d7f4f13308935c803c
Cluster id: 3b5b82af683b55d7f4f13308935c803c
Nodes:
30e15967ab375bf39ee02a470c4bad7a
71887bf22d3bdb6eb2c202d02a718b0c
85343873975b49928819f060b9d25fba
Volumes:
b18fcbeadca4a0f57ed29aa35396d41e

[root@heketi-866cfc978-kv85n /]# heketi-cli topology info
Cluster Id: 3b5b82af683b55d7f4f13308935c803c

    Volumes:

        Name: vol_8b28e377963415b426fcd1bbdb0479c5
        Size: 20
        Id: 8b28e377963415b426fcd1bbdb0479c5
        Cluster Id: 3b5b82af683b55d7f4f13308935c803c
        Mount: 10.18.32.113:vol_8b28e377963415b426fcd1bbdb0479c5
        Mount Options: backup-volfile-servers=10.18.32.114,10.18.32.145
        Durability Type: replicate
        Replica: 3
        Snapshot: Enabled
        Snapshot Factor: 1.00

                Bricks:
                        Id: 1202ae152b61aacf4477d2937d0068a4
                        Path: /var/lib/heketi/mounts/vg_5d5a2e50fcf0498bfcbed3793a68ca91/brick_1202ae152b61aacf4477d2937d0068a4/brick
                        Size (GiB): 20
                        Node: 71887bf22d3bdb6eb2c202d02a718b0c
                        Device: 5d5a2e50fcf0498bfcbed3793a68ca91

                        Id: 811715b86e22ec9db7ab29a70042ff48
                        Path: /var/lib/heketi/mounts/vg_a5bc501ab4266b1697b6eb3ebb72a367/brick_811715b86e22ec9db7ab29a70042ff48/brick
                        Size (GiB): 20
                        Node: 30e15967ab375bf39ee02a470c4bad7a
                        Device: a5bc501ab4266b1697b6eb3ebb72a367

                        Id: 815e9fcae49cc970481199b87b6e1c64
                        Path: /var/lib/heketi/mounts/vg_d233612da3e31fe5656ece6283287497/brick_815e9fcae49cc970481199b87b6e1c64/brick
                        Size (GiB): 20
                        Node: 85343873975b49928819f060b9d25fba
                        Device: d233612da3e31fe5656ece6283287497


        Name: heketidbstorage
        Size: 2
        Id: b18fcbeadca4a0f57ed29aa35396d41e
        Cluster Id: 3b5b82af683b55d7f4f13308935c803c
        Mount: 10.18.32.113:heketidbstorage
        Mount Options: backup-volfile-servers=10.18.32.114,10.18.32.145
        Durability Type: replicate
        Replica: 3
        Snapshot: Disabled
......

```
Create Storage Class and Persistent Volume Claims in Kubernetes
```bash
# 1. modify heketi server ip and port in 'gluster-storage-class.yaml'
# 2. create glusterfs storage class with kubectl
$ kubectl apply -f gluster-storage-class.yaml

# 3. modify pvc volume size in 'glusterfs-pvc.yaml'
$ kubectl apply -f glusterfs-pvc.yaml
$ ubectl get pvc -n monitoring
NAME             STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS               AGE
grafana          Bound    pvc-f24866d2-82ab-11e9-b18e-02000a122070   5Gi        RWO            prometheus-storage-class   20h
prometheus-pvc   Bound    pvc-880f03ea-82a7-11e9-b18e-02000a122070   20Gi       RWO            prometheus-storage-class   20h

```
## Reference
https://github.com/gluster/gluster-kubernetes/blob/master/README.md
