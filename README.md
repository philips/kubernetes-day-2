# Kubernetes Day 2

These are notes to accompany my [KubeCon EU 2017 talk](https://cloudnativeeu2017.sched.com/event/9Tcw/kubernetes-day-2-cluster-operations-i-brandon-philips-coreos). The slides [are available as well](https://docs.google.com/presentation/d/1LpiWAGbK77Ha8mOxyw01VlZ18zdMSNimhsrb35sUFv8/edit?usp=sharing). The video is [available from Youtube](https://www.youtube.com/watch?v=U1zR0eDQRYQ).

How do you keep a Kubernetes cluster running long term? Just like any other service, you need a combination of monitoring, alerting, backup, upgrade, and infrastructure management strategies to make it happen. This talk will walk through and demonstrate the best practices for each of these questions and show off the latest tooling that makes it possible. The takeaway will be lessons and considerations that will influence the way you operate your own Kubernetes clusters.

## WARNING

These are notes for a conference talk. Much of this may become out of date very quickly. My goal is to turn much of this into docs overtime.

## Cluster Setup

All of the demos in this talk were done with a self-hosted cluster deployed with the [Tectonic Installer](https://github.com/coreos/tectonic-installer#tectonic-installer) on AWS.

This cluster was also deployed using the self-hosted etcd option which at the time of this writing [isn't merged into the Tectonic Installer](https://github.com/coreos/tectonic-installer/pull/135) quite yet.

## Failing a Scheduler

Scale it down to remove all schedulers

```
kubectl scale -n kube-system deployment kube-scheduler --replicas=0
```

OH NO, scale it back up

```
kubectl scale -n kube-system deployment kube-scheduler --replicas=1
```

Unfortunately, it is too late. Everything is ruined?!?!

```
kubectl get pods -l k8s-app=kube-scheduler -n kube-system
NAME                              READY     STATUS    RESTARTS   AGE
kube-scheduler-3027616201-53jfh   0/1       Pending   0          52s
```

Get the current kubernetes deployment

```
kubectl get -n kube-system deployment -o yaml kube-scheduler > sched.yaml
```

Pick a node name from this list at random

```
kubectl get nodes -l master=true
```

Edit the sched.yaml to use just the pod spec and set the metadata.nodename field to one to the selected node above. Something like this:

```
kind: Pod
metadata:
  labels:
    k8s-app: kube-scheduler
  name: kube-scheduler
  namespace: kube-system
spec:
  nodeName: ip-10-0-37-115.us-west-2.compute.internal
  containers:
  - command:
    - ./hyperkube
    - scheduler
    - --leader-elect=true
    image: quay.io/coreos/hyperkube:v1.5.5_coreos.0
    imagePullPolicy: IfNotPresent
    name: kube-scheduler
    resources: {}
    terminationMessagePath: /dev/termination-log
  dnsPolicy: ClusterFirst
  nodeSelector:
    master: "true"
  restartPolicy: Always
  securityContext: {}
  terminationGracePeriodSeconds: 30
```

At this point the deployment scheduler should be ready and can take over

```
kubectl get pods -l k8s-app=kube-scheduler -n kube-system
```

Delete the temporary pod

```
kubectl delete pod -n kube-system kube-scheduler<Paste>
```

## Downgrade/Upgrade Scheduler

Edit the scheduler and downgrade a patch release.

```
kubectl edit -n kube-system deployment kube-scheduler
```

Now edit the scheduler and upgrade a patch release.

```
kubectl edit -n kube-system deployment kube-scheduler
```

Boom!

## kubectl drain and corden

```
$ kubectl get nodes
NAME                                        STATUS    AGE
ip-10-0-13-248.us-west-2.compute.internal   Ready     19h
```

To make a node unschedulable and remove all pods run the following

```
kubectl drain  ip-10-0-13-248.us-west-2.compute.internal 
```

## kubectl cordon and uncordon

To ensure a node doesn't get additional workloads you can cordon/uncordon a node. This is very useful to investigate an issue and to ensure a node doesn't change while debugging.

```
$ kubectl cordon ip-10-0-84-104.us-west-2.compute.internal
node "ip-10-0-84-104.us-west-2.compute.internal" cordoned
```

To undo run uncordon

```
$ kubectl uncordon ip-10-0-84-104.us-west-2.compute.internal
node "ip-10-0-84-104.us-west-2.compute.internal" uncordoned
```

## Monitoring

Using [contrib/kube-prometheus](https://github.com/coreos/prometheus-operator/tree/master/contrib/kube-prometheus) deployed in the self-hosted configuration.

Proxy to run queries against prometheus

```
while true; do kubectl port-forward -n monitoring prometheus-k8s-0 9090; don
```

NOTE: a few bugs were [found and filed](https://github.com/coreos/prometheus-operator/issues/created_by/philips) against this configuration

## Configure etcd backup

Note: S3 backup isn't working in the etcd Operator on self-hosted yet; hunting this down.

Setup AWS upload creds:

```
kubectl create secret generic aws-credential --from-file=$HOME/.aws/credentials -n kube-system
kubectl create configmap aws-config --from-file=$HOME/.aws/config-us-west-1 -n kube-system
```

```
kubectl edit deployment etcd-operator -n kube-system
```


```
      - command:
        - /usr/local/bin/etcd-operator
        - --backup-aws-secret
        - aws-credential
        - --backup-aws-config
        - aws-config
        - --backup-s3-bucket
        - tectonic-eo-etcd-backups
```

```
kubectl get cluster.etcd -n kube-system kube-etcd -o yaml > etcd
kubectl replace -f etcd  -n kube-system
```
