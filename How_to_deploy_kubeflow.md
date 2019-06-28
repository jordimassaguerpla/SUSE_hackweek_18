
    Disclaimer

    This is a hackweek project, so this is not ready for production use. It contains hacks and workarounds just to "make it work".

    This has been tested with SUSE CaaSP Beta 3 (public beta). I used that "kubernetes distribution" because given I am involved on that project, I am familiar with it (and I love it :) )

    However, instructions here should be also valid openSUSE Kubic and in general for any kubernetes+cri-o distribution


# Introduction

This how to is about installing [KubeFlow](https://www.kubeflow.org) on a [SUSE CaaSP Beta 3 cluster](https://www.suse.com/betaprogram/caasp-beta/)
(kubernetes/cri-o cluster) with NVIDIA gpu acceleration installed following the instructions at https://github.com/jordimassaguerpla/SUSE_hackweek_18/blob/master/How_to_setup_SUSE_CaaSP_kubernetes_crio_GPU.md

> What is Kubeflow? From its website:
> The Kubeflow project is dedicated to making deployments of machine learning (ML) workflows on Kubernetes simple, portable and scalable. Our goal is not to recreate other services, but to provide a straightforward way to deploy best-of-breed open-source systems for ML to diverse infrastructures. Anywhere you are running Kubernetes, you should be able to run Kubeflow.

# Installing kubeflow

(Extracted from https://www.kubeflow.org/docs/started/getting-started-k8s/)

    Follow these steps to deploy Kubeflow:

    Download a kfctl release from the Kubeflow releases page.

    Unpack the tar ball:

    tar -xvf kfctl_<release tag>_<platform>.tar.gz

    Run the following commands to set up and deploy Kubeflow. The code below includes an optional command to add the binary kfctl to your path. If you don’t add the binary to your path, you must use the full path to the kfctl binary each time you run it.

    # The following command is optional, to make kfctl binary easier to use.
    export PATH=$PATH:<path to kfctl in your kubeflow installation>

    export KFAPP=<your choice of application directory name>
    # Default uses IAP.
    kfctl init ${KFAPP}
    cd ${KFAPP}
    kfctl generate all -V
    kfctl apply all -V

        ${KFAPP} - the name of a directory where you want Kubeflow configurations to be stored. This directory is created when you run kfctl init. If you want a custom deployment name, specify that name here. The value of this variable becomes the name of your deployment. The value of this variable cannot be greater than 25 characters. It must contain just the directory name, not the full path to the directory. The content of this directory is described in the next section.

    Check the resources deployed in namespace kubeflow:

    kubectl -n kubeflow get  all


# Access the kubeflow dashboard

Then, in our installation we don't have a Load Balancer, so we will use NodePort for accessing the services (https://kubernetes.io/docs/concepts/services-networking/service/#nodeport). 

To know which port to connect, run

     kubectl get svc -n kubeflow | grep ambassador | grep NodePort
     
In our setup, we had this output:

    ambassador                               NodePort    10.107.43.116    <none>        80:31440/TCP        18h

So we opened a firefox on http://192.168.1.190:31440, because 192.168.1.190 is the IP of the GPU node and the port 31440 is the port
on the node (NodePort).

More on ambassador at https://www.getambassador.io/

Now that we have the kubeflow dashboard, we are ready to create a notebook server.

# Create a notebook server

With the default kubeflow "deployment" you will already have the option to start a [Jupyter notebook](https://jupyter.org/) server.

> The Jupyter Notebook
> The Jupyter Notebook is an open-source web application that allows you to create and share documents that contain live code, equations, visualizations and narrative text. Uses include: data cleaning and transformation, numerical simulation, statistical modeling, data visualization, machine learning, and much more.

The Jupyter Notebook is the "de facto standard" in ML industry for defining and training models.

Jupyter Notebook server is accessible from the kubeflow dashboard on the left menu at "Notebooks". There you will see the "console"
for creating servers. For our example, we will create a server which we will name "testgpu" using the default values except that 
in the "Extra Resouces" section, we will add

    {"nvidia.com/gpu": 1}
  
This way, the notebook will be sheduled in the node with the GPU.

And then, we are ready for our first notebook.

# The HelloWorld Notebook

At the jupyter dashboard for our recently created notebook server, there is a button with the name "New"... you guess it! This is for
creating a new notebook. Just click there and you will have a notebook.

Notebooks are very handy because you can "combine" text and code. When you want a code to run, you need to press "Shift+Enter".

Our first notebook will be the Hello World. Just write in the notebook:

    print("Hello World!")
  
and press "Shift+Enter" and voila! you will see "Hello World!"




