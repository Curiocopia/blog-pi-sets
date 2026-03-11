# 📦 Pi Sets

This is a demonstration of using vertical pod autoscaling when performing arithmetics to calculate $\pi$ to a large degree of accuracy (over 100M digits). Details are described in [Pi Sets].

## 🌟 Highlights

- Uses a StatefulSet to implement `workers` that compute iterative values of $P_k, Q_k, T_k$ to compute $a$ and $b$ where (Chudnovsky formaula as adapted by [Nick Craig-Wood])

$$
\pi = \frac{426880\sqrt{10005}}{13591409a + 545140134b}
$$
$$
a = \sum_{n=0}^{\infty} \frac{(-1)^n(6n)!}{(3n)!(n!)^3640320^{3n}}
$$ and
$$
b = \sum_{n=0}^{\infty} \frac{(-1)^n(6n)!n}{(3n)!(n!)^3640320^{3n}}
$$ 
 
- A `merger` Job that triggers the calculation of $P_k, Q_k, T_k$ to compute $a$ and $b$, collects the set of $P_k, Q_k, T_k$, merges them to compute $a, b$ and $\pi$. 

## ℹ️ Overview

Please refer to [Pi Sets] for the specific blog and the relevant repo(s). If you don't have time TL;DR:

This was an attempt to learn more about how to use VPA to adjust the resources of a StatefulSet. The `merger` Job use the `worker` StatefulSet pod indices to calculate the specific pod that calculates the set of $P_k, Q_k, T_k$ based on the binomial splicing. The `merger` Job communicates with the `workers` using a FastAPI. VPA is used to adjust the `cpu` and `memory` for the `workers`. 

### ✍️ Authors

All repos shared in [CurioCopia] are shared under Creative Commons license for others to adopt and use it as they wish.

## 🚀 Usage

1. Generate your own Docker image for [chudnovsky-worker], place it in your favorite registry.
2. Generate your own Docker image for [chudnovsky-merger], place it in your favorite registry.
3. Clone this repo, adjust the essential parameters in [demo], run kustomize to generate the resources.
4. Access the UI via browser and enjoy.

## ⬇️ Installation

Before you start the rest of the exercise, make sure you install VPA. The instructions are in [VPA GitHub] repository.
In addition, the installation assumes you have a PersistentVolume with a proper `hostpath`. In the example, the volume accessMode is assumed to be of "ReadWriteOnce, ReadWriteMany". Adjust as you see fit.

Let's follow the typical Kustomize installation process.

Define a place to work:
```bash
TEST_HOME=$(mktemp -d)
```
### Establish the Base

```bash
BASE=$TEST_HOME/base
mkdir -p $BASE

CONTENT="https://raw.githubusercontent.com/Curiocopia/blog-pi-sets/refs/heads/main"

curl -s -o "$BASE/#1" "$CONTENT/base\
/{kustomization.yaml,pi-merger-job.yaml,pi-pvc.yaml,pi-sets.env,pi-worker-service.yaml,pi-worker-statefulset.yaml,pi-worker-vpa.yaml}"
```
Look at the directory:
```bash
tree $TEST_HOME
```
Expect something like:
```bash
/tmp/tmp.rKw8Rf4Cup
└── base
    ├── kustomization.yaml
    ├── pi-merger-job.yaml
    ├── pi-pvc.yaml
    ├── pi-sets.env
    ├── pi-worker-service.yaml
    ├── pi-worker-statefulset.yaml
    └── pi-worker-vpa.yaml
```
### The Base Customization

The base directory has a kustomization file:
```bash
more $BASE/kustomization.yaml
```
You can run kustomize on the base to emit customized resources to stdout and inspect:
```bash
kustomize build $BASE
```
If you are satisfied:
```bash
kubectl apply -k $BASE 
```
Once the resources are running, follow the job status
```bash
kubectl get jobs -w 
```
```bash
$ kubectl get jobs -w
NAME        STATUS    COMPLETIONS   DURATION   AGE
pi-merger   Running   0/1           2m38s      2m38s
pi-merger   Running   0/1           2m43s      2m43s
pi-merger   Running   0/1           2m44s      2m44s
pi-merger   Complete   1/1           2m44s      2m44s
```
Once the job is completed, check your `hostpath` directory to see the file. In my case, it is `/tmp`.
```bash
$ ls -l /tmp/pi.txt
-rw-r--r-- 1 systemd-network systemd-journal 100000003 Mar 10 17:28 /tmp/pi.txt
```

## Create Overlay

Create a `demo` overlay.
```bash
OVERLAYS=$TEST_HOME/overlays
mkdir -p $OVERLAYS/demo
```
## Demo Customization

```bash
curl -s -o "$OVERLAYS/demo/#1" "$CONTENT/overlays/demo\
/{kustomization.yaml,pi-sets-demo.env,pi-worker-statefulset-patch.yaml,pi-worker-vpa-patch.yaml}"
```
Adjust the parameters as you need. Set `namespace` for all resources and `chudnovsky-worker` and `chudnovsky-merger` `image` in `kustomization.yaml`:
```yaml
namespace: demo

images:
- name: merger
  newName: my-registry/chudnovsky-merger
  newTag: latest
- name: worker
  newName: my-registry/chudnovsky-worker
  newTag: latest
```
Adjust `pi-sets-demo.env` values for ConfigMap creation to use in various reources.

Adjust the values for the `spec.replicas` in the `pi-worker-statefulset-patch.yaml` to be identical to $WORKER_REPLICAS$ set in `pi-sets-demo.env`.

Adjust the values for the `spec.resourcePolicy` in the `pi-worker-vpa-patch.yaml`:
```yaml
spec:
  resourcePolicy:
    containerPolicies:
    - containerName: 'worker'
      minAllowed:
        cpu: 500m
        memory: "500Mi"
      maxAllowed:
        cpu: 500m
        memory: "1000Mi"
```
Inspect the values. If you are satisfied, apply the kustomization after you create the `demo` namespace.
```bash
kubectl apply -k $OVERLAYS/demo 
```
Once the resources are running, repeat the instruction for the base.

## 💭 Feedback and Contributing

If you have any other suggestions for improvements or corrections, please drop a note in Discussions.

[Pi Sets]: https://curiocopia.com/blog/pi-sets
[Nick Craig-Wood]: https://www.craig-wood.com/nick/articles/pi-chudnovsky/
[Curiocopia]: https://curiocopia.com
[chudnovsky-worker]: https://github.com/Curiocopia/chudnovsky-worker
[chudnovsky-merger]: https://github.com/Curiocopia/chudnovsky-merger
[demo]: overlays/demo/
[VPA GitHub]: https://github.com/kubernetes/autoscaler/tree/master/vertical-pod-autoscaler