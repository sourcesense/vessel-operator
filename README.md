# Vessel operator

This repository contains all the bits that makes the user able to deploy the operator for the [Vessel project](https://github.com/sourcesense/vessel).

## Deploy the operator

Clone the repository:

```
$ git clone https://github.com/sourcesense/vessel-operator
```

Then if you already have access to your Kubernetes cluster (by having a .kube directory locally), you'll just have to do:

```
$ cd vessel-operator
$ make deploy
```

After you should be able to see the operator running:

```
$ kubectl -n vessel-operator-system get ingress,pods,services
NAME                                                     READY   STATUS    RESTARTS   AGE
pod/vessel-operator-controller-manager-55694999c-66rbr   2/2     Running   0          94s

NAME                                                         TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
service/vessel-operator-controller-manager-metrics-service   ClusterIP   10.106.210.54   <none>        8443/TCP   94s
```

## Create a Vessel custom resource

First create the token to access your cluster:

```
$ kubectl create serviceaccount k8sadmin -n kube-system
$ kubectl create clusterrolebinding k8sadmin --clusterrole=cluster-admin --serviceaccount=kube-system:k8sadmin
$ kubectl -n kube-system describe secret $(kubectl -n kube-system get secret | (grep k8sadmin || echo "$_") | awk '{print $1}') | grep token: | awk '{print $2}'
...
<OUTPUT IS YOUR TOKEN>
...
```

Then you can rely con the sample file to create a myvessel.yaml like this:

```yaml
apiVersion: cache.sourcesense/v1alpha1
kind: Vessel
metadata:
  name: vessel-sample
spec:
  vessel_nsname: 'myvessel'
  vessel_k8s_url: 'https://<YOUR CLUSTER IP>:<YOUR CLUSTER PORT>'
  vessel_k8s_token: '<YOUR TOKEN>'
```

After this it will be possible to actually create the resource:

```
$ kubectl create -f myvessel.yaml
```

And then, in short, you should see the created resources:

```
$ kubectl -n vessel-operator-system get ingress,pods,services
NAME         READY   STATUS    RESTARTS   AGE
pod/vessel   1/1     Running   0          73s

NAME                     TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
service/vessel-service   ClusterIP   10.96.57.183   <none>        80/TCP    72s
```

## Exposing the service

The operator automatically creates the vessel service, but it also supports ingress.
If you have an ingress-nginx configured in your cluster you can create a yaml like this:

```yaml
apiVersion: cache.sourcesense/v1alpha1
kind: Vessel
metadata:
  name: vessel-sample
spec:
  vessel_nsname: 'myvessel'
  vessel_k8s_url: 'https://<YOUR CLUSTER IP>:<YOUR CLUSTER PORT>'
  vessel_k8s_token: '<YOUR TOKEN>'
  vessel_ingress_enable: true
  vessel_ingress_host: 'vessel.sourcesense.com'
  vessel_ingress_class: 'nginx'
```

After creating the resource:

```
$ kubectl create -f myvessel.yaml
```

It will be possible to see the ingress:

```
NAME                                       CLASS    HOSTS                    ADDRESS          PORTS   AGE
ingress.networking.k8s.io/vessel-ingress   <none>   vessel.sourcesense.com   192.168.122.34   80      2m46s

NAME         READY   STATUS    RESTARTS   AGE
pod/vessel   1/1     Running   0          2m48s

NAME                     TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
service/vessel-service   ClusterIP   10.96.57.183   <none>        80/TCP    2m47s
```

And queries will be available at the declared address:

```
$ curl -s http://vessel.sourcesense.com/query?kind=deployment | jq
{
  "latest_run": {
    "VesselLinter": "2021-12-23 16:42"
  },
  "size": 20,
  "count": 20,
  "page": 1,
  "result": [
    {
    ...
    ...
    {
      "id": 2211,
      "name": "calico-kube-controllers",
      "namespace": "kube-system",
      "kind": "deployment",
      "issue": "MISSING_REQUESTS",
      "issue_metadata": "",
      "task": "VesselLinter",
      "created_at": "2021-12-23 16:42"
    }
  ]
}

```
