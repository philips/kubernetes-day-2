# Kubernetes Day 2

These are notes to accompany my [KubeCon EU 2017 talk](https://cloudnativeeu2017.sched.com/event/9Tcw/kubernetes-day-2-cluster-operations-i-brandon-philips-coreos). The slides [are available as well](https://docs.google.com/presentation/d/1LpiWAGbK77Ha8mOxyw01VlZ18zdMSNimhsrb35sUFv8/edit?usp=sharing).

How do you keep a Kubernetes cluster running long term? Just like any other service, you need a combination of monitoring, alerting, backup, upgrade, and infrastructure management strategies to make it happen. This talk will walk through and demonstrate the best practices for each of these questions and show off the latest tooling that makes it possible. The takeaway will be lessons and considerations that will influence the way you operate your own Kubernetes clusters.

## WARNING

These are notes for a conference talk. Much of this may become out of date very quickly. My goal is to turn much of this into docs overtime.

## Cluster Setup

All of the demos in this talk were done with a self-hosted cluster deployed with the [Tectonic Installer](https://github.com/coreos/tectonic-installer#tectonic-installer) on AWS.

## Configure etcd backup

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
