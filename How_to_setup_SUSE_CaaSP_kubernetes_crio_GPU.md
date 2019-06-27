This is the result from my hackweek project:
https://hackweek.suse.com/projects/architecting-a-machine-learning-project-with-suse-caasp


 
> __Disclaimer__
>
> _This is a hackweek project, so this is not ready for production use. It contains hacks and workarounds just to "make it work"._
>
> _This has been tested with [SUSE CaaSP Beta 3 (public beta)](https://www.suse.com/betaprogram/caasp-beta/). I used that "kubernetes distribution" because given I am
> involved on that project, I am familiar with it (and I love it :) )_
>
>_However, instructions here should be also valid [openSUSE Kubic](https://kubic.opensuse.org/) and in general for any [kubernetes](https://kubernetes.io/)+[cri-o](https://cri-o.io/) distribution_  

# How to setup SUSE CaaSP 4.0 Beta3 to work with GPU

SUSE CaaSP 4.0 is build on top of Kubernetes and cri-o. What we need to do is start by installing SUSE CaaSP and then setup the nvidia parts that glue kubernetes, cri-o and NVIDIA drivers, and then setup a NVIDIA kubernetes device plugin, so pods can "expose" gpus as resources.

|-----------------------------------------------------|  
NVIDIA kubernetes device plugin  
|-----------------------------------------------------|  
          kubernetes  
|-----------------------------------------------------|  
           cri-o  
|-----------------------------------------------------|  
        cri-o OCI hook  
|-----------------------------------------------------|  
 nvidia-container-runtime-hook  
|-----------------------------------------------------|  
     nvidia-container-cli  
|-----------------------------------------------------|  
     libnvidia-container  
|-----------------------------------------------------|  
        nvidia-drivers  
      (kernel and CUDA)          
|-----------------------------------------------------|  
         NVIDIA GPU  
|-----------------------------------------------------|              
  

  
## Installing SUSE CaaSP

We need a cluster, that is obvious :), so let's start with installing two nodes with [SUSE CaaSP 4.0 Beta3](https://www.suse.com/betaprogram/caasp-beta/) powered by [SUSE Linux Enterprise Server 15 SP1](https://www.suse.com/products/server/), which will become our worker and master nodes:

* __gpu__: A bare metal workstation with [NVIDIA Quadro K2000 graphics card](https://www.nvidia.com/content/PDF/data-sheet/DS_NV_Quadro_K2000_OCT13_NV_US_LR.pdf)

* __master__:A virtual machine

You can do that by following the [SUSE CaaSP Beta 3 deployment instructions](https://susedoc.github.io/doc-caasp/beta/caasp-deployment/single-html/#_deployment_on_existing_sles_installation).


> __Tip__   
_Make sure you have a user "sles" which can run "sudo" without a password, and that both nodes have the other's public ssh keys in "~sles/.ssh/authorized_keys". Also, as stated in the deployment guide, that you have the ssh-agent running, and add the hostnames into /etc/hosts so they are both reachable by hostname_. Then, disable _firewalld_ and enable _sshd_.
  
> __Tip__
_Do not configure a swap partition and set the vm to use 2 CPUs. Otherwise, SUSE CaaSP will fail to install._
>
  
Finally, we need both machines to be in the same network. For this, I setup the vm to use a _macvtap host device_. You can find more info on _macvtap_ [here](https://virt.kernelnewbies.org/MacVTap). However, I just did that with _virt-manager_ on [openSUSE Leap 15.1](https://en.opensuse.org/Portal:15.1)*.

_(*): This is not precise. Actually, for whatever reason, I could not setup this with the virt-manager run as a "normal user" but I could if I started virt-manager from YaST (with root permissions... may that be the reason?)_

>__Tip__
_Do you want to know if the cluster is properly setup? Run the [k8s conformance tests](https://github.com/cncf/k8s-conformance/blob/master/instructions.md)._
>

Once we have a SUSE CaaSP cluster running, we can proceed to install the NVIDIA drivers.

## Installing nvidia graphics driver kernel module

So we have a workstation with NVIDIA GPU [compatible with](https://developer.nvidia.com/cuda-gpus) [CUDA](https://developer.nvidia.com/cuda-zone) (in our case Quadro K2000). Now is time to install the right drivers so we can use that.

Drivers can be installed from [NVIDIA download servers](https://download.nvidia.com/suse/sle15sp1/x86_64/):
  
    sudo zypper ar https://developer.nvidia.com/cuda-gpus nvidia
    sudo zypper ref
    sudo zypper install nvidia-gfxG05-kmp-default

You can check if drivers are loaded:
  
    lsmod | grep nvidia
  
Now that we have the drivers, we can install the NVIDIA driver for computing with GPUs using CUDA.
  
## Installing NVIDIA driver for computing with GPUs using CUDA
  
After installing nvidia drivers, we need to install the nvidia-computeG05. Given we setup the nvidia repo from the previous step, all we need to do is:
  
    sudo zypper install nvidia-computeG05

Then, add your user to the video group, and also to the root user:

    sudo usermod -G video -a sles
    sudo usermod -G video -a root

> __Important__: If you are not a member of the video group, you won't be able to access the nvidia devs at /dev/nvidia* . Later you will see that even running as root is not enough, that you need to **explicetely** run as **root:video**.

You can check if this is running by running
  
    nvidia-smi

and you should get this output

Wed Jun 26 14:30:59 2019  

+-----------------------------------------------------------------------------+  
| NVIDIA-SMI 430.26       Driver Version: 430.26       CUDA Version: 10.2     |  
|-------------------------------+----------------------+----------------------+   
| GPU  Name        Persistence-M| Bus-Id        Disp.A | Volatile Uncorr. ECC |  
| Fan  Temp  Perf  Pwr:Usage/Cap|         Memory-Usage | GPU-Util  Compute M. |  
|===========================================================================|  
|   0  Quadro K2000        Off  | 00000000:05:00.0 Off |                  N/A |  
| 31%   47C    P8    N/A /  N/A |      0MiB /  1998MiB |      0%      Default |  
+-------------------------------+----------------------+----------------------+  
                                                                                
+-----------------------------------------------------------------------------+  
| Processes:                                                       GPU Memory |  
|  GPU       PID   Type   Process name                             Usage      |  
|=============================================================================|  
|  No running processes found                                                 |  
+-----------------------------------------------------------------------------+  
  
 Congrats! We have drivers installed! Let's continue with installing the nvidia container libs :)

## Installing libnvidia-container

Ready for starting with the container specific parts? Let's start with libnvidia-container

What is [libnvidia-container](https://github.com/NVIDIA/libnvidia-container) ? See definition in the github project:
> This repository provides a library and a simple CLI utility to automatically configure GNU/Linux containers leveraging NVIDIA hardware.  
The implementation relies on kernel primitives and is designed to be agnostic of the container runtime.
>
Unfortunately, NVIDIA is not building packages for SUSE, so we will use the "distribution agnostic build", downloadable from the release page:

https://github.com/NVIDIA/libnvidia-container/releases/download/v1.0.0/libnvidia-container_1.0.0_x86_64.tar.xz

> __Info__
> The latest release is 1.0.2, but that one was built with a more recent glibc and so we cannot install it on SLE15SP1
> 

You can untar that package and copy the binaries in bin and libs to /usr/bin and /usr/lib64.

    tar xJf libnvidia-container_1.0.0_x86_64.tar.xz
    cp libnvidia-container_1.0.0/usr/local/bin/nvidia-container-cli /usr/bin
    cp libnvidia-container_1.0.0/usr/local/lib/libnvidia-container.so* /usr/lib64

> __Info__
> I guess you could also copy to /usr/local and set the PATH and the ldconfig paths

To test it, run

    nvidia-container-cli info

and you should get

    NVRM version:   430.26                                                          
    CUDA version:   10.2                                                            
                                                                                
    Device Index:   0                                                               
    Device Minor:   0                                                               
    Model:          Quadro K2000                                                    
    Brand:          Quadro                                                          
    GPU UUID:       GPU-6a04b812-c20e-aeb6-9047-6382930eef7d                        
    Bus Location:   00000000:05:00.0                                                
    Architecture:   3.0 

> __Warning__
> If you run
> _nvidia-container-cli info_
> as root
> you will get this error
> _nvidia-container-cli: initialization error: cuda error: no cuda-capable device is detected_
> Don't worry. We will fix this later by making root run this binary with the video group
> For now, run it as a user which belongs to the video group
    
And we have the libnvidia container installed! Now let's go for the hook, that will plug cri-o with the libnvidia-container.

## Installing nvidia-container-runtime-hook

Since we are using cri-o, all we need to do is to setup a [OCI hook](https://github.com/cri-o/cri-o#oci-hooks-support) that will run some custom binary _pre-starting_ a container. This binary mount host paths to make the nvidia-container-cli, libnvidia-container, and devices available inside the container.

This is the [nvidia-container-runtime-hook](https://github.com/NVIDIA/nvidia-container-runtime/tree/master/hook).

> _Info_
> Note we don't need to install any modified runc or cri-o. This is the difference with docker. If we were using docker, we would need to install a modified runtime (runc) or modified docker (nvidia-docker). With cri-o, we can use upstream cri-o and we don't have to maintain a forked runc.

Unfortunately, NVIDIA is not building this for SUSE and, alike the libnvidia-container, is neither building it distribution agnostic. I could have build it for SUSE, but this would have taken me the whole hackweek (at least), so instead I took the CentOS RPM, unpack it and copied the binaries over:

    docker run -ti -v$PWD:/var:/tmp centos:7
    DIST=$(. /etc/os-release; echo $ID$VERSION_ID)
    curl -s -L https://nvidia.github.io/nvidia-container-runtime/$DIST/nvidia-container-runtime.repo |    tee /etc/yum.repos.d/nvidia-container-runtime.repo
    yum install --downloadonly nvidia-container-runtime-hook
    cp /var/cache/yum/x86_64/7/nvidia-container-runtime/packages/nvidia-container-runtime-hook-1.4.0-2.x86_64.rpm /var/tmp
    exit
    urpm nvidia-container-runtime-hook-1.4.0-2.x86_64.rpm

now you can copy the contents of the rpm

* /etc/nvidia-container-runtime/config.toml 
* /usr/bin/nvidia-container-runtime-hook
* /usr/share/containers/oci/hooks.d/oci-nvidia-hook.json

And **very important** , edit the  /etc/nvidia-container-runtime/config.toml  and set

    user = "root:video"

> __Tip__
> Do you want to test if this is working?
> Install podman, comment the cni_default_network in /etc/containers/libpod.conf, and run
>  _sudo podman  run nvidia/cuda nvidia-smi_
>  and you should get the nvidia-smi output as you were running it on the host

Now, cri-o is able to run a container with GPU acceleration. Next step is kubernetes "setup".


## NVIDIA Device Plugin

The NVIDIA device plugin for Kubernetes is a _Daemonset_ that allows you to automatically:

-   Expose the number of GPUs on each nodes of your cluster
-   Keep track of the health of your GPUs
-   Run GPU enabled containers in your Kubernetes cluster.

This [repository](https://github.com/NVIDIA/k8s-device-plugin) contains NVIDIA's official implementation of the [Kubernetes device plugin](https://github.com/kubernetes/community/blob/master/contributors/design-proposals/resource-management/device-plugin.md).

Thus, let's "install" it:

    kubectl create -f https://raw.githubusercontent.com/NVIDIA/k8s-device-plugin/1.0.0-beta/nvidia-device-plugin.yml

And that is all!

See that the node is exposing the NVIDIA GPUs by running

    kubectl describe pod gpu

And you should get something like:
    
    Allocatable:                                                                    
    ...
    nvidia.com/gpu:     1                                                          
    Allocated resources:                                                            
      (Total limits may be over 100 percent, i.e., overcommitted.)                  
      Resource           Requests  Limits
      --------           --------  ------                                           
      cpu                0 (0%)    0 (0%)                                           
      memory             0 (0%)    0 (0%)                                           
      ephemeral-storage  0 (0%)    0 (0%)                                           
      nvidia.com/gpu     0         0                                                
      Events:              <none>   

See the **nvidia.com/gpu** ? :)

## Testing

We have all setup and running now, let's have some more fun and do some testing :)

Let's create a file named *cuda-vector-add.yaml*:  
                                                                                
    apiVersion: v1                                                                  
    kind: Pod                                                                       
    metadata:                                                                       
      name: cuda-vector-add                                                         
    spec:                                                                           
      restartPolicy: OnFailure                                                      
      containers:                                                                   
        - name: cuda-vector-add                                                     
          # https://github.com/kubernetes/kubernetes/blob/v1.7.11/test/images/nvidia-cuda/Dockerfile
          image: "k8s.gcr.io/cuda-vector-add:v0.1"                                  
          resources:                                                                
            limits:                                                                 
              nvidia.com/gpu: 1 # requesting 1 GPU 

"Run it":
    
    kubectl create -f cuda-vector-add.yaml                                          

and see the output

    kubectl logs cuda-vector-add                                       
    [Vector addition of 50000 elements]                                             
    Copy input data from the host memory to the CUDA device                         
    CUDA kernel launch with 196 blocks of 256 threads                               
    Copy output data from the CUDA device to the host memory                        
    Test PASSED                                                                     
    Done                                                                            
                                                                                
if we change the requested gput to 2
    
    nvidia.com/gpu: 2 # requesting **2** GPU                              

Then you will see how this does not get scheduled, because our workstation has only 1 GPU:
  
    kubectl get pods                                                   
    NAME              READY   STATUS    RESTARTS   AGE                              
    cuda-vector-add   0/1     Pending   0          50s                              
                                                                                
    kubectl describe pod cuda-vector-add                               
    Limits:                                                                     
      nvidia.com/gpu:  2                                                        
    Requests:                                                                   
      nvidia.com/gpu:  2                                                        
    Events:                                                                         
    Type     Reason            Age               From               Message       
     ----     ------            ----              ----               -------       
    Warning  FailedScheduling  7s (x3 over 85s)  default-scheduler  0/2 nodes are available: 2 Insufficient nvidia.com/gpu.
                                                                                

# TODOs

So we manage! We have SUSE CaaSP Cluster scheduling pods that need GPUs to a node that has an NVIDIA GPU. What is left? I am sure you can guess it .... **PACKAGING**

* Packaging libnvidia-containers
* Packaging nvidia-containers-runtime-hook

> __Warning__
> Pods need to mount some host paths to be able to access nvidia libs, command and devices. In SUSE CaaSP Beta3 there was no restriction but you could have some PSP and so not be able to do so. If that is the case, you could add a namespaced RoleBinding:

    ---
    apiVersion: v1
    kind: ServiceAccount
    metadata:
      name: nvidia-device-plugin
      namespace: kube-system
    ---
    apiVersion: rbac.authorization.k8s.io/v1
    kind: RoleBinding
    metadata:
      name: nvidia-device-plugin-psp-privileged
      namespace: kube-system
    roleRef:
      kind: ClusterRole
      name: suse![add-emoji](https://assets-cdn.github.com/images/icons/emoji/caasp.png)psp:privileged
      apiGroup: rbac.authorization.k8s.io
    subjects:
    - kind: ServiceAccount
      name: nvidia-device-plugin
      namespace: kube-system
    
> And then in your DaemonSet spec, add `serviceAccount: nvidia-device-plugin` .
    
> This creates the ServiceAccount+RoleBinding in the kube-system
namespace - if you're deploying into another NS, swap out `kube-system` 
for the namespace you're using.
    

