# CNUT Demos

## etcd Demo

### View the health of the cluster

```
etcdctl cluster-health
```

### Working with key/value pairs

```
etcdctl set location us
```

```
etcdctl get location
```

---

### Watching for changes

```
etcdctl exec-watch /location -- sh -c "env | grep ETCD"
```

```
etcdctl set location china
```

### Keys that expire

```
etcdctl set location china --ttl 10
```

```
etcdctl exec-watch /location -- sh -c "env | grep ETCD"
```

### Automatic fail over

Simulate a node failure by rebooting the current etcd leader:

```
gcloud compute ssh -A controller
```

Get the current leader using the `etcd-leader` tool:

```
etcd-leader
```

Remotely reboot the etcd leader machine:

```
ssh node0 "sudo reboot"
```

The remaining etcd members will automatically elect a new leader when the current leader goes down.
Get the new etcd leader:

```
etcd-leader
```

---

## Kubernetes Demo

### Create a replication controller

Using kubectl, the Kubernetes CLI client, create the inspector replication controller:

```
kubectl run inspector \
  --labels="app=inspector,track=stable" \
  --image=b.gcr.io/kuar/inspector:1.0.0
```

List the pods created by the inspector replication controller:

```
kubectl get pods
```

Use the describe command to get more info:

```
kubectl describe pods inspector
```

### Self-healing

Replication controllers ensure the desired number of pods are always running.
If we delete the current inspector pod:

```
kubectl get pods
```

```
kubectl delete pods inspector-<id>
```

The inspector replication controller will replace it:

```
kubectl get pods
```

### Scale the inspector replication controller

Scale the number of inspector pods to 5:

#### Terminal 2

```
kubectl scale rc inspector --replicas=5
```

#### Terminal 1

```
kubectl get pods --watch-only
```

### Expose the inspector service

Expose the inspector service to the outside world by creating a service:

```
kubectl create -f configs/inspector-svc.yaml
```

List the services:

```
kubectl get services
```

Get more information about the service using the describe command:

```
kubectl describe service inspector
```

Visit the inspector service in your web browser:

http://104.155.195.1:36000

### The canary pattern

Now that the inspector service is up and running on version 1.0.0. We can use the canary pattern to roll out a new version.

First create a new replication controller to manage a new set of inspector pods running the new version:

```
kubectl run inspector-canary \
  --labels="app=inspector,track=canary" \
  --replicas=1 \
  --image=b.gcr.io/kuar/inspector:2.0.0
```

Use the curl command to locate the new instance of the inspector pod:

```
while true; do curl -s http://104.155.195.1:36000/ | \
  grep -o -e 'Version: Inspector [0-9].[0-9].[0-9]'; sleep .5; done
```

Notice version 2.0.0 is mixed in with version 1.0.0?

### Rolling update

One you are happy with the new version of the app, we can replace version 1.0.0 with version 2.0.0 across the cluster using the rolling update command.

First remove the canary controller:

```
kubectl delete rc inspector-canary
```

#### Terminal 1

Watch for pod events:

```
kubectl get pods --watch-only
```

#### Terminal 2

Make an HTTP request to the inspector service every .5 seconds:

```
while true; do curl -s http://104.155.195.1:36000/ | \
  grep -o -e 'Version: Inspector [0-9].[0-9].[0-9]'; sleep .5; done
```

#### Terminal 3

Start the upgrade process from version 1.0.0 to version 2.0.0:

```
kubectl rolling-update inspector --update-period=3s --image=b.gcr.io/kuar/inspector:2.0.0
```

Interrupt the upgrade process with ctrl-c:

```
ctrl-c
```

Rollback to version 1.0.0:

```
kubectl rolling-update inspector --rollback=true --update-period="2s" --image=b.gcr.io/kuar/inspector:1.0.0
```

Start the upgrade process again and let it complete:

```
kubectl rolling-update inspector --update-period=3s --image=b.gcr.io/kuar/inspector:2.0.0
```
