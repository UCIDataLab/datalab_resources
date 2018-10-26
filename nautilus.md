Nautilus Cluster Instructions
===
The Nautilus computing cluster is a high-performance computing cluster.
It provides a large amount of GPU computing resources for our group.
The cluster uses [Kubernetes](https://kubernetes.io/) to manage the distribution of resources to users.

Unfortunately, Kubernetes has quite a steep learning curve, and many tutorials can be difficult to understand due to the large amount of associated jargon. This guide is intended to provide quick and simple instructions for getting up and running with Kubernetes without going into too many details.

Creating an Account
---
- Start by clicking the *login* button at the top-right corner of the [Nautilus homepage](https://nautilus.optiputer.net).
- Look up *University of California, Irvine*.
- You should be redirected to UCI's authentication page. Use your UCI ID and password to login just as you would for any other campus resource.
- Next contact either Robby or Dimitris and ask them to:
    1. Authenticate your Nautilus account.
    2. Add your account to the *uci-datalab* namespace.

Kubernetes Setup
---
You will need the [kubectl](https://kubernetes.io/docs/reference/kubectl/kubectl) command line tool to use the cluster.
- Begin by installing kubectl by  following [these instructions](https://kubernetes.io/docs/tasks/tools/install-kubectl/).
- Next you will need to configure kubectl. This can be done by:
    1. Clicking the *Get Config* link in the top of the [Nautilus homepage](https://nautilus.optiputer.net) to download the `config` file.
    2. Saving the `config` file to the `.kube` folder in your home directory.
- Set your default namespace to *uci-datalab*. This can be done by running:

```{bash}
kubectl config set-context nautilus --namespace=uci-datalab
```

A Quick Overview of Relevant Terms
---
Kubernetes exists to help orchestrate the deployment of applications which are built up from many modular parts (which are stored in [containers](https://www.docker.com/resources/what-container)).
To accomodate the needs of large web applications (which need to dynamically scale the resources they have available), allocating resources using Kubernetes is a considerably complex topic.

As machine learning researchers, almost none of the details are relevant to our work.
Typically, we will only need the following things:
1. Command-line access to a server which is running the software we need.
2. A place to store data.
3. The required hardware resources (e.g. GPUs and RAM) for our models.

In Kubernetes lingo, our server is referred to as a *Pod*, and our data storage is referred to as a *PersistentVolume*.
The reason we treat storage seperately is that computing resources are intended to only be used temporarily (e.g. you should only request a GPU for as long as you need it), but typically we want to have the ability to access our data and results after our computing resources have been made available for other users.
This is why it referred to as a *persistent* volume.

Allocating and Using Resources
---
We highly recommend going through the tutorials provided on the [Nautilus homepage](https://www.docker.com/resources/what-container) as well as the [TensorFlow and Jupyter: Step by Step](https://wiki.nautilus.optiputer.net/wiki/Tensorflow_And_Jupyter_:_Step_By_Step) tutorial on the Nautilus wiki before reading this section.

The main goal here is to understand how to create a pod that runs software other than TensorFlow. For this example we'll create a pod that runs PyTorch.

Pods and PersistentVolumes are defined using a configuration file. For example, this is the configuration file used in the TensorFlow tutorial:

```{yaml}
apiVersion: v1
kind: Pod
metadata:
  name: gpu-pod-example
spec:
  containers:
  - name: gpu-container
    image: gitlab-registry.nautilus.optiputer.net/prp/jupyterlab:latest
    args: ["sleep", "infinity"]
    resources:
      limits:
        nvidia.com/gpu: 1
```

There are two important sections to be aware of. The first one is:
```{yaml}
  containers:
  - name: gpu-container
    image: gitlab-registry.nautilus.optiputer.net/prp/jupyterlab:latest
```
This says we're going to create a Pod named *gpu-container*, which will use prp's preconfigured *jupyterlab* image, which comes with TensorFlow and Jupyter.
The name is how we refer to the resource when we want to use it.
The image defines what operating system and software will be installed on the pod when it is loaded.

The second important section is:
```{yaml}
    resources:
      limits:
        nvidia.com/gpu: 1
```
This defines what physical resources the pos will have access to. In this case a single GTX 1080ti GPU.

### Example: Custom PyTorch Configuration w/ Storage

Depending on your application, you will probably need to use software other than TensorFlow + Jupyter.
Luckily, someone has probably already defined an image with the software you need!
[Docker Hub](hub.docker.com) is a public repository for images, all of which can be used in a Kubernetes configuration effortlessly.

#### Image
For example, suppose you wanted to make a pod with PyTorch installed.
A quick search for PyTorch on Docker Hub will yield the following top result:

[pytorch/pytorch](https://hub.docker.com/r/pytorch/pytorch)

If you click the *tags* tab you will see all of the possible versions of the image you can use.
Unless you need a previous version for a specific reason, it is almost always the case that you will want to go with **latest**.
To use this image in place of TensorFlow you need only replace the `image` argument in the configuration file to:
```{yaml}
    image: pytorch/pytorch:latest
```

#### Storage
Lastly, we need some storage for data / model checkpoints / analyses.
To do this we'll use the first block from the [volume-example.yaml](https://gitlab.com/ucsd-prp/prp_k8s_config/blob/master/volume-example.yaml) file provided on the Nautilus homepage.
*Note: I am also going to change the name to `data` since that seems more appropriate*
I recommend creating a seperate configuration file for the PersistentVolume since you will probably want to create and delete volumes independently from Pods (so you can keep your data while freeing GPU/RAM for other users).
This is what the adapted `volume-example.yaml` looks like:
```{yaml}
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
    name: data
spec:
    storageClassName: rook-block
    accessModes:
      - ReadWriteOnce
    resources:
        requests:
            storage: 100Gi
```
we'll also need to add the following `volumeMounts` section to our container:
```{yaml}
        volumeMounts:
          - mountPath: /data
            name: data
```
and an additional `volumes` section to `spec`:
```{yaml}
    volumes:
        - name: data
          persistentVolumeClaim:
              claimName: data
```

#### Final configuration

Putting everything together, our Pod configuration file [`pytorch-example.yaml`](pytorch-example.yaml) looks like:
```{yaml}
apiVersion: v1
kind: Pod
metadata:
    name: pytorch-example
spec:
    containers:
      - name: pytorch-container
        image: pytorch/pytorch:latest
        args: ["sleep", "infinity"]
        volumeMounts:
          - mountPath: /data
            name: data
        resources:
            limits:
                nvidia.com/gpu: 1
    restartPolicy: Never
    volumes:
        - name: data
          persistentVolumeClaim:
              claimName: data
```

And our PersistentVolume configuration file [`volume-example.yaml`](volume-example.yaml) looks like:
```{yaml}
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
    name: data
spec:
    storageClassName: rook-block
    accessModes:
      - ReadWriteOnce
    resources:
        requests:
            storage: 100Gi
```

#### Commands

To create the Pod/PersistentVolume:
```{bash}
kubectl create -f volume-example.yaml  # Create the PersistentVolume
kubectl create -f pytorch-example.yaml  # Create the Pod
```

To check on the status of your Pod/PersistentVolume:
```{bash}
kubectl get -f volume-example.yaml  # Check on the PersistentVolume
kubectl get -f pytorch-example.yaml  # Check on the Pod
```
The PersistentVolume is ready when its status is `Bound`.
The Pod is ready when its status is ``.

To log into the console run:
```
kubectl exec pytorch-example -it bash
```

To forward a port run:
```
kubectl port-forward pytorch-example 8888:8888
```
*Note: This will allow you to access Jupyter notebooks, TensorBoard, etc.*

To delete the Pod/PersistentVolume:
```{bash}
kubectl create -f pytorch-example.yaml  # Delete the Pod
kubectl create -f volume-example.yaml  # Delete the PersistentVolume
```
These commands can be run seperately if you want to delete only the Pod.
Also you should *always delete the Pod first* to prevent unwanted behaviour.


Final Words
---
Hopefully this tutorial has helped understand the details of Kubernetes.
If you have any further questions please contact the group admin.
Lastly, remember to be a good user:

**Always shut down Pods you aren't actively using!**