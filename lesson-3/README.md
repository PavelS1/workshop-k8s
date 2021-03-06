# Kubernetes Operators

## Kubernetes Custom Resource Definition

- **A Resource** - is an endpoint that stores a collection of API objects of a
certain kind For example, the built-in pods resource contains a collection of Pod objects.

- **A Custom Resource** - is an object that extends the Kubernetes API.
It allows you to introduce your own API into a project or a cluster.

- **A Custom Resource Definitions (CRD)** - is a file that describes your own object
kinds and lets the Kubernetes API server handle the entire lifecycle. Deploying
a CRD into the cluster causes the Kubernetes API server to begin serving the specified custom resource.

Example:
```yaml
apiVersion: apiextensions.k8s.io/v1beta1
kind: CustomResourceDefinition
metadata:
  # name must match the spec fields below, and be in the form: <plural>.<group>
  name: crontabs.stable.example.com
spec:
  # group name to use for REST API: /apis/<group>/<version>
  group: stable.example.com
  # list of versions supported by this CustomResourceDefinition
  versions:
    - name: v1
      # Each version can be enabled/disabled by Served flag.
      served: true
      # One and only one version must be marked as the storage version.
      storage: true
  # either Namespaced or Cluster
  scope: Namespaced
  names:
    # plural name to be used in the URL: /apis/<group>/<version>/<plural>
    plural: crontabs
    # singular name to be used as an alias on the CLI and for display
    singular: crontab
    # kind is normally the CamelCased singular type. Your resource manifests use this.
    kind: CronTab
    # shortNames allow shorter string to match your resource on the CLI
    shortNames:
    - ct
```


## Deploy etcd cluster

We will now try to deploy etcd cluster into our Kubernetes. We will use etcd operator for it.
As we want to do it properly we want our operator to be managed by a Lifecycle Operator. Lifecycle
Operator enables management of operators inside Kuberentes clusters. It's mostly responsible for:

* Deploying Operators into namespaces
* Updating Operators
* Defines channels for operators delivery (Testing/Prod)
* Package Operators
* Operators inventory

You can read more about (Operator Lifecycle Manager)[https://github.com/prgcont/operator-lifecycle-manager/blob/master/Documentation/design/philosophy.md]

### Deploying Operator Lifecycle Manager
``` bash
git clone https://github.com/prgcont/operator-lifecycle-manager
cd operator-lifecycle-manager
kubectl apply -f deploy/upstream/manifests/0.5.0
```
Note: If previous apply coomand ends with error please run it again.

After should check that Operator Lifecycle Manager is up and running:

``` bash
kubectl -n kube-system get pods
```

You should see output similar to:

```
NAME                                    READY     STATUS    RESTARTS   AGE
alm-operator-8d6dc8b65-8cnrb            1/1       Running   0          1m
catalog-operator-f764ff46d-j8msc        1/1       Running   0          1m
...
```


### Deploying etcd operator
When Lifecycle Manager Operator is successfully deployed and is managing our cluster
we can deploy etcd operator to a newly created etc namespace. 


``` bash
kubectl create namespace etc
```

Note: If you want to operate on 'etc' namespace you need to pass namespace option to `kubectl`
so it will look like `kubectl -n etc get pods`

Deploy operator here by applying InstallPlan, this will assure that etcd
operator is deployed to our etc namespaces and is watch its crd object.

``` yaml
apiVersion: app.coreos.com/v1alpha1
kind: InstallPlan-v1
metadata:
  namespace: etc
  name: etcd
spec:
  clusterServiceVersionNames:
  - etcdoperator.v0.9.2
  approval: Automatic
```

Tasks:
* Check that etcd operator is deployed
* List all available crd


### Install etcd Cluster using etcd Operator

To install etcd cluster we need to apply following CRD object. 

``` yaml
apiVersion: "etcd.database.coreos.com/v1beta2"
kind: "EtcdCluster"
metadata:
  name: "example-etcd-cluster"
spec:
  size: 3
  version: "3.2.13"
  repository: "docker.io/prgcont/etcd"
```

Verify the state of deployed etcd cluster
```bash
kubectl -n etc  describe etcdcluster example-etcd-cluster
```

Tasks
* Check that etcdclusters.etcd.database.coreos.com CRD is available

#### Check the health of etcd Cluster ####

Exec into one etcd pod
```bash
# Get arbitrary pod name using 
kubectl -n etc get po -l etcd_cluster=example-etcd-cluster

# Exec into etcd pod
kubectl -n etc exec -it <POD_NAME> -- sh

# In container:
# Update env variable
export ETCDCTL_API=3

# List etcd members 
etcdctl member list

# Write and read record
etcdctl put /here test
etcdctl get /here
```

Tasks:
* Scale up Currently deployed etcd cluster and verify that record you made into the DB still exists,
* Deploy second etcd cluster in 'etc2' namespace
* Check that both clusters are independent (contains different data)

#### Note on Cluster wide operators ####

Note: This can lead to security issues and render you cluster to be hard to maintain

The above example created `etcd-operator` in `default` namespace and etcd Cluster in same namespace. 
By default etcd Operator reacts only on `etcdcluster` objects that are in same namespace. This behavior can be changed by passing
arg `-cluster-wide` to `etcd-operator` and creating `etcdcluster` object with annotation: `etcd.database.coreos.com/scope: clusterwide`. From our example: 

```yaml
apiVersion: "etcd.database.coreos.com/v1beta2"
kind: "EtcdCluster"
metadata:
  name: "example-etcd-cluster"
  annotations:
    etcd.database.coreos.com/scope: clusterwide
spec:
  size: 3
  version: "3.2.13"
  repository: "docker.io/prgcont/etcd"
```

Note: You need to update RBAC rules if you want etcd operator to manage resources across all kubernetes cluster. 




## Write simple dummy Operator in Python

We will create a very simple 'operator' in Python. It will be responsible for:
* monitoring changes in gordons.operator.prgcont.cz crd
* it will schedule and maintain pods according to replicas key in the crd
* it will register all the operated pods
* it will report which pods belongs to which gordon cluster (instance of crd)

We will start by defining crd which will be monitored by our operator.

```yaml
apiVersion: apiextensions.k8s.io/v1beta1
kind: CustomResourceDefinition
metadata:
  name: gordons.operator.prgcont.cz
spec:
  group: operator.prgcont.cz
  version: v1
  scope: Namespaced
  names:
    plural: gordons
    singular: gordon
    kind: Gordon
    shortNames:
    - gn
```

Before running the code below:

```bash
# create a virtual environment for python
virtualenv k8s
# load the virtual env
. k8s/bin/activate
# install the dependency
pip install kubernetes
```

Then we need to run following python code:

**Note:** this a daemon so use `&`, `tmux`, `screen` or another terminal

``` python
import threading
import time
import yaml


from kubernetes import client, config, watch

# Following line is sourcing your ~/.kube/config so you are authenticated same
# way as kubectl is
config.load_kube_config()
v1 = client.CoreV1Api()
crds = client.CustomObjectsApi()

crd_group = "operator.prgcont.cz" # group of crd to be vatched
namespace = 'default'

pod_template = yaml.safe_load("""
apiVersion: v1
kind: Pod
metadata:
  generateName: gordon-
spec:
  containers:
    - name: gordon
      image: prgcont/gordon:v1.0
""")


def main():
    # our simple watch loop for changes in our crd
    stream = watch.Watch().stream(crds.list_namespaced_custom_object,
                                  crd_group,
                                  'v1',
                                  namespace,
                                  'gordons')
    for event in stream:
        if event['type'] == 'ADDED':
            deploy(event['object'])
        elif event['type'] == 'MODIFIED':
            change(event['object'])
        elif event['type'] == 'DELETED':
            delete(event['object'])
        else:
            print('Unsupported change type: %s' % event['type'])


def deploy(crd):
    replicas = crd['spec']['gordon']['replicas']
    name = crd['metadata']['name']
    if 'state' in crd:
        print('[%s] Already exists!' % name)
        return
    else:
        crd['state'] = {}
        crd['state']['pods'] = []
    print('[%s] Deploying %s replicas of gordon.' %
          (name,
           replicas))
    i = 1
    while i <= replicas:
        resp = v1.create_namespaced_pod(namespace, pod_template)
        crd['state']['pods'].append(resp.metadata.name)
        print('[%s] Scheduled pod %s' % (name,
                                         resp.metadata.name))
        i += 1

    crd['state']['replicas'] = replicas
    crds.patch_namespaced_custom_object(crd_group,
                                        'v1',
                                        namespace,
                                        'gordons',
                                        name,
                                        crd)


def change(crd):
    replicas = crd['spec']['gordon']['replicas']
    name = crd['metadata']['name']
    print('[%s] Modifying.' % name)
    i = crd['state']['replicas']
    if i > replicas:
        while i > replicas:
            pod = crd['state']['pods'].pop()
            print('[%s] Removing pod %s .' % (name, pod))
            v1.delete_namespaced_pod(pod,
                                     namespace,
                                     client.V1DeleteOptions())
            i -= 1
    elif i < replicas:
        while i <= replicas:
            resp = v1.create_namespaced_pod(namespace, pod_template)
            crd['state']['pods'].append(resp.metadata.name)
            print('[%s] Scheduled pod %s' % (name,
                                             resp.metadata.name))
            i += 1
    crd['state']['replicas'] = i
    crds.patch_namespaced_custom_object(crd_group,
                                        'v1',
                                        namespace,
                                        'gordons',
                                        name,
                                        crd)


def delete(crd):
	pass


class Checker(threading.Thread):

    def run(self):
        while True:
            time.sleep(1)


if __name__ == "__main__":
    checker = Checker()
    checker.daemon = True
    checker.start()
    main()

```

After running the code create gordon cluster by applying following object:
``` yaml
apiVersion: "operator.prgcont.cz/v1"
kind: Gordon
metadata:
  name: gordoncluster
spec:
  gordon: 
    replicas: 3
```

Then you should check that 3 replicas of gordon pods are running via:
``` bash
kubectl get pods
```

Tasks:
* Explain what operator is doing, identify all Kubernetes API Calls
* Implement delete() function which will stop all pods
* Modify Checker().run() function so it will check that managed pods are running and 
create new ones if any of them was terminated (hint, use ``get_namespaced_pod`` function 
and ``kubectl delete pod`` commands to test it).
* 

## Operator Framework

[Operator Framework](https://coreos.com/operators/) is set of tools that simplifies creation management of k8s operators.

The operators created by Operator Framework are using same primitives like k8s controller which can be found in this diagram:

![Operator internals](./pic/operator_sdk_internals.jpeg "source: https://itnext.io/under-the-hood-of-the-operator-sdk-eebc8fdeebbf")

This framework is really good choice if you are golang developer or your applications stack is golang based, its benefit for other apps
maybe is not good enough to learn is as operator can be created in almost any language and it can still be managed by Lifecycle Operator
Manager.

Operator SDK helps you a lot with:
* generating CRD for you
* monitoring changes in CRD/Kubernetes cluster (you can register watchers and handlers easily)
* package and deploy operator into cluster

Advance task:
* Try to follow the [tutorial](https://github.com/operator-framework/getting-started)

