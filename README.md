### RKE2 Terraform Install

This deploys an [rke2](https://docs.rke2.io/) cluster in Embassy Openstack

There are 3 directories for different install configurations

#### Requirements

##### GPU

The worker flavor must contain the following custom property to make sure the worker is scheduled on the GPU hypervisor

`pci_passthrough:alias='gpu-m10:4'`

##### SSD

There must be ample SSD Cinder storage quota to accommodate the control plane master nodes root disk. This enables etcd to run smoothly with low latency.

##### Keys

By default your local ssh key  `~/.ssh/id_rsa` will be used

##### Credentials

You will need to source [unrestricted application credentials](https://docs.embassy.ebi.ac.uk/userguide/Embassyv4.html#retrieving-credentials) to the target Openstack project

##### Image

You will need an image.
Any of [these](https://docs.rke2.io/install/requirements/#operating-systems) are supported.

For example

    wget --no-check-certificate https://cloud-images.ubuntu.com/daily/server/focal/current/focal-server-cloudimg-amd64.img
openstack image create --disk-format qcow2 --container-format bare --private --file focal-server-cloudimg-amd64.img focal-daily

#### Usage

Adjust settings in `variables.tf` then

- source unrestricted application credentials
- `terraform init`
- `terraform plan`
- `terraform apply`

The kubeconfig will be copied to your local dir (`rke2.yaml`)

##### GPU Operator

Please wait 5 minites for the Nvidia GPU operator to fully initialise and then make sure GPUs are visible

    kubectl exec -n gpu-operator  -it nvidia-driver-daemonset-g5ljd -- nvidia-smi

    Wed Dec 22 19:06:27 2021
    +-----------------------------------------------------------------------------+
    | NVIDIA-SMI 470.82.01    Driver Version: 470.82.01    CUDA Version: 11.4     |
    |-------------------------------+----------------------+----------------------+
    | GPU  Name        Persistence-M| Bus-Id        Disp.A | Volatile Uncorr. ECC |
    | Fan  Temp  Perf  Pwr:Usage/Cap|         Memory-Usage | GPU-Util  Compute M. |
    |                               |                      |               MIG M. |
    |===============================+======================+======================|
    |   0  Tesla M10           On   | 00000000:00:05.0 Off |                  N/A |
    | N/A   28C    P8     8W /  53W |      0MiB /  8129MiB |      0%      Default |
    |                               |                      |                  N/A |
    +-------------------------------+----------------------+----------------------+
    |   1  Tesla M10           On   | 00000000:00:06.0 Off |                  N/A |
    | N/A   27C    P8     8W /  53W |      0MiB /  8129MiB |      0%      Default |
    |                               |                      |                  N/A |
    +-------------------------------+----------------------+----------------------+
    |   2  Tesla M10           On   | 00000000:00:07.0 Off |                  N/A |
    | N/A   32C    P8     8W /  53W |      0MiB /  8129MiB |      0%      Default |
    |                               |                      |                  N/A |
    +-------------------------------+----------------------+----------------------+
    |   3  Tesla M10           On   | 00000000:00:08.0 Off |                  N/A |
    | N/A   31C    P8     8W /  53W |      0MiB /  8129MiB |      0%      Default |
    |                               |                      |                  N/A |
    +-------------------------------+----------------------+----------------------+

    +-----------------------------------------------------------------------------+
    | Processes:                                                                  |
    |  GPU   GI   CI        PID   Type   Process name                  GPU Memory |
    |        ID   ID                                                   Usage      |
    |=============================================================================|
    |  No running processes found                                                 |
    +-----------------------------------------------------------------------------+

#### Security

The kubelet API access is via a floating IP dirently on the Master node visible to the Internet. You will need to secure access to this using Openstack security groups on the Master instance

#### Openstack Cloud Provider

This is supported and installed.

##### LoadBalancer Type

This will create an Openstack load balancer in Octavia and should be secured following [Magnum documetation](http://docs.embassy.ebi.ac.uk/userguide/Embassyv4.html#securing-load-balancers-that-are-created-by-services-in-kubernetes).

##### Cinder

Storage class ready to use, but make sure you flag one as default before use

    $ kubectl patch storageclass csi-cinder-sc-retain -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'`

    kubectl get sc
    NAME                   PROVISIONER                RECLAIMPOLICY   VOLUMEBINDINGMODE   ALLOWVOLUMEEXPANSION   AGE
    csi-cinder-sc-delete   cinder.csi.openstack.org   Delete          Immediate           true                   23m
    csi-cinder-sc-retain   cinder.csi.openstack.org   Retain          Immediate           true                   23m

#### Ingress

By default Ingress is deployed in RKE2 with no LoadBalancer, so no external access.
To fix this you need to upgrade the chart with helmv3

    ./helm upgrade --install rke2-ingress-nginx ingress-nginx   --repo https://kubernetes.github.io/ingress-nginx   --namespace kube-system

You will need to be using the helmv3 binary to do this which can be downloaded from [here](https://helm.sh/docs/intro/install/#from-the-binary-releases)

You will then have a LB with a FIP attached for Ingress

    kubectl get svc -A
    NAMESPACE       NAME                                      TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)                      AGE
    calico-system   calico-kube-controllers-metrics           ClusterIP      10.43.188.73    <none>        9094/TCP                     10m
    calico-system   calico-typha                              ClusterIP      10.43.36.180    <none>        5473/TCP                     11m
    default         kubernetes                                ClusterIP      10.43.0.1       <none>        443/TCP                      12m
    kube-system     rke2-coredns-rke2-coredns                 ClusterIP      10.43.0.10      <none>        53/UDP,53/TCP                11m
    kube-system     rke2-ingress-nginx-controller             LoadBalancer   10.43.136.120   45.88.81.80   80:30639/TCP,443:30186/TCP   2m29s
    kube-system     rke2-ingress-nginx-controller-admission   ClusterIP      10.43.88.190    <none>        443/TCP                      10m
    kube-system     rke2-metrics-server                       ClusterIP      10.43.109.154   <none>        443/TCP                      10m

If you wish to install Ingress from scratch you can disable the installation of RKE2 ingress by adding this line to `server.yaml`

    `disable: rke2-ingress-nginx`

#### CNI

By default RKE2 ships with flannel. To change to calico you will need to add this line to `server.yaml`

`cni: calico`
