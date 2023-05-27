# Game streaming on Kubernetes

This repo is an example of how to set up a Kubernetes cluster and an example workload to stream a game from Kubernetes using [Sunshine](https://github.com/LizardByte/Sunshine).

Sunshine is an open source implementation of [NVIDIA GameStream](https://www.nvidia.com/en-gb/shield/support/shield-tv/gamestream/) and [Moonlight](https://moonlight-stream.org/) is an open source client for playing games.

This repo heavily builds on the work from [Games on Whales (GOW)](https://github.com/games-on-whales/gow).
They provided the main containers and an example implementation of a Kubernetes deployment.
The workload in this example folder has been changed but will be upstreamed at some point.

## Create cluster
You can start by creating a Kubernetes cluster.
I have an example cluster config in [cluster.yaml](./cluster.yaml) that uses EKS and [Karpenter](https://karpenter.sh) for node autoscaling.

You can use [`eksctl`](https://eksctl.io) to create the cluster or bring your own EKS cluster.

```
eksctl create cluster -f cluster.yaml
```

## Set up storage

We will be using an EBS volume for our game install storage mounted at /home/retro.
This will allow us to only install the games once.

This will create a storage class based on gp3 volumes and creates a PersistentVolumeClaim of 100Gb to start.

```
kubectl apply -f gp3.sc.yaml -f retro-home.pvc.yaml
```

## Set up Karpenter provisioner

Next we need a provisioner so Karpenter will know what instances to use and an AwsNodeTemplate that will configure our GPU node when it is created.

The node uses [tailscale](https://tailscale.com) for private networking so I don't have to expose any ports or pay for a load balancer.
You will need to create an auth key from your admin console https://login.tailscale.com/admin/settings/keys and add it to [gow.yaml](./gow.yaml) where it says `YOUR_AUTH_KEY`. The auth key should be reusable and devices should be ephemeral.

Make sure you do that before you create the provisioner.

```
kubectl apply -f provisioner.yaml
```

## Set up NVIDIA device plugin

Create a daemonset that will check the GPU drivers on each node and label the nodes with what GPUs they have available.
This DaemonSet comes [directly from NVIDIA](https://github.com/NVIDIA/k8s-device-plugin) so we'll install it from there.

```
kubectl create -f https://raw.githubusercontent.com/NVIDIA/k8s-device-plugin/v0.14.0/nvidia-device-plugin.yml
```

## Deploy GOW

Now we can deploy the workload!

There are 4 containers in the pod.

1. xorg
1. pulseaudio
1. sunshine
1. steam

The containers all communicate with volumes to share sockets for X and sound devices.
They need to start in order so X and sound are available before steam or sunshine are launched.

```
kubectl apply -f gow.yaml
```

X kept crashing for me with the defaut GOW scripts.
Instead I override the command to manually run X11 in the forground.

The sunshine container also updates sunshine to a nightly build and installs a missing package.

## Connect with moonlight

After the pod is running—it will take a few minutes because the node installs a lot of packages—you can connect to the node with Moonlight.

Find the node IP address with tailscale

```
sudo tailscale status
```

Launch Moonlight on your computer and click add computer and add it by IP address.

You'll need to put in the pin to authenticate your computer.
Port forward the sunshine interface to your computer.

```
kubectl port-forward deployment/gow 47990
```

Now open your browser to https://localhost:47990/pin
Log in with the default username/password of admin/admin and enter the pin.

You should be able to go back to moonlight and click on the computer and then launch desktop.

Now you can log in to steam and install your games.
You can launch games from here or add them as apps in the sunshine web interface.