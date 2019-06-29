
    Disclaimer

    This is a hackweek project, so this is not ready for production use. It contains hacks and workarounds just to "make it work".

    This has been tested with SUSE CaaSP Beta 3 (public beta). I used that "kubernetes distribution" because given I am involved on that project, I am familiar with it (and I love it :) )

    However, instructions here should be also valid for openSUSE Kubic and in general for any kubernetes+cri-o distribution



If you have followed the previous steps [01](https://github.com/jordimassaguerpla/SUSE_hackweek_18/blob/master/01-How_to_setup_SUSE_CaaSP_kubernetes_crio_GPU.md)
[02](https://github.com/jordimassaguerpla/SUSE_hackweek_18/blob/master/02-How_to_deploy_kubeflow.md),
you will have now kubeflow deployed on SUSE CaaSP 4 with GPU acceleration.

We will now deploy an example ML "pipeline", meaning:

1- Load a Jupyter notebook with a definition of the model using Keras and Tensorflow
2- Run the Jupyter notebook to train a model
3- Extract the model
4- Deploy a microservice with the trainned model using Seldon

The example we will use is one for "github issue summarization". You can find more details on:

https://github.com/kubeflow/examples/tree/master/github_issue_summarization

We will follow the steps on that example but adapting them to our cluster.

In our case, we are skipping the disc configuration, not because it is not needed, but just to simplify and so be able to accomplish it within the hackweek time.


# Prerequisites: ksonnet and s2i

_instructions based on: https://github.com/kubeflow/examples/blob/master/github_issue_summarization/01_setup_a_kubeflow_cluster.md and on https://github.com/kubeflow/examples/blob/master/github_issue_summarization/03_serving_the_model.md _

In order to deploy our ML microservice, we will use a tool called [ksonnet](https://ksonnet.io) for managing the configuration (yes, another tool for "packaging" ...)

Let's download the latest release https://github.com/ksonnet/ksonnet/releases/download/v0.13.1/ks_0.13.1_linux_amd64.tar.gz and copy the binary to our path:

    tar zxvf ks_0.13.1_linux_amd64.tar.gz
    cp ks_0.13.1_linux_amd64/ks /usr/bin

> __Info__
> Jsonnet is a CLI-supported framework for extensible Kubernetes configurations
Built on the JSON templating language Jsonnet, ksonnet provides an organizational structure and specialized features for managing configurations across different clusters and environments.

>__Info__
> Jsonnet has been discontinued, so don't get too attached to it. You can read more about it [here](https://blogs.vmware.com/cloudnative/2019/02/05/welcoming-heptio-open-source-projects-to-vmware/)

For building images containing our model, we will use the tool "Source2Image". You can downloaded it from https://github.com/openshift/source-to-image/releases/download/v1.1.14/source-to-image-v1.1.14-874754de-linux-amd64.tar.gz and, as we did for the ksonnet, extract the binary and copied to your path.

>__Info__
> (from the s2i webpage)
Source-to-Image (S2I) is a toolkit and workflow for building reproducible container images from source code. S2I produces ready-to-run images by injecting source code into a container image and letting the container prepare that source code for execution. By creating self-assembling builder images, you can version and control your build environments exactly like you use container images to version your runtime environments.


# Load the example notebook

>__Warning__:
> Do NOT use the image mentioned in https://github.com/kubeflow/examples/blob/master/github_issue_summarization/01_setup_a_kubeflow_cluster.md. gcr.io/kubeflow-dev/issue-summarization-notebook-cpu:latest won't "work" in our cluster. I know, saying "won't work" is the worst bug report... but I didn't had the time to dig in on the reason. Instead, we will install the needed deps using the jupyter console (see below)

Before loading the example notebook, we will need to install some dependencies, download the code and do some directory setup. For this, go to the jupyter notebook server, and open a console. Then, run the following commands:

    pip3 install --upgrade pip
    pip3 install -U scikit-learn
    pip3 install -U ktext
    pip3 install -U IPython
    pip3 install -U annoy
    pip3 install -U tqdm
    pip3 install -U nltk
    pip3 install -U matplotlib
    pip3 install -U tensorflow
    pip3 install -U bernoulli
    pip3 install -U h5py
    git clone https://github.com/google/seq2seq.git
    pip3 install -e ./seq2seq/
    git clone https://github.com/kubeflow/examples.gif
    mkdir tmp/github-issues-data

Now our jupyter notebook has all the needed dependencies, setup and code, and we can procede to load the example.

You can use the web ui to load the examples/github_issue_summarization/notebook/Training.ipynb notebook

Then, given we had not setup external volumes for simplicity, you will need to replace the following code in the notebook

    %env DATA_DIR=/mnt/github-issues-data

by

    %env DATA_DIR=/tmp/github-issues-data

And we are ready to train! Let's do that in the next chapter.

# Training

_based on https://github.com/kubeflow/examples/blob/master/github_issue_summarization/02_training_the_model.md _

After loading the Training.ipynb notebook and doing some minor adaptations, we can start with the training.

The notebook contains a complete walk-through of loading the training data, preprocessing it, and training it, so all you have to do is run the notebook, and watch the output at each step to confirm that the resulting models produce sensible predictions.

After training completes, download the resulting files to your local machine. The following files are needed for serving results:

    seq2seq_model_tutorial.h5 - the keras model
    body_pp.dpkl - the serialized body preprocessor
    title_pp.dpkl - the serialized title preprocessor

You can download them from the jupyter file browser.


# Serving the model as a microservice

_based on https://github.com/kubeflow/examples/blob/master/github_issue_summarization/03_serving_the_model.md _

We are going to use [Seldon Core](https://www.seldon.io/open-source/) to serve the model. _IssueSummarization.py_ contains the code for this model. We will wrap this class into a seldon-core microservice which we can then deploy as a REST or GRPC API server.

We will build our image based on the files downloaded from the notebook on the previous chapter. In order to do that, you need to copy the files into the notebook directory and run

    s2i build -E environment_seldon_rest . seldonio/seldon-core-s2i-python36:0.4 docker.io/YOUR_USERNAME_IN_DOCKER_HUB/gh_issue_summarization:0.1

Then, we need to upload the image to a public registry, in our case we are using docker hub:

    docker login #this logs you into docker hub
    docker push docker.io/YOUR_USERNAME_IN_DOCKER_HUB/gh_issue_summarization:0.1

>> __Info__ You could use a different registry as long as it is available in your cluster
>> __Warning__: If you don't copy the model and preprocessed files, the docker image won't contain them!!!

And now you can use seldon core to deploy it:

    cd ks_app
    # Generate the seldon component and deploy it
    ks generate seldon seldon
    ks apply default -c seldon


Seldon Core should now be running on your cluster. You can verify it by running:

    kubectl get pods -n kubeflow

You should see two pods named seldon-seldon-cluster-manager-* and seldon-redis-*.

And now we can "deploy" our model:

    ks generate seldon-serve-simple-v1alpha2 issue-summarization-model \
      --name=issue-summarization \
      --image=docker.io/YOUR_USERNAME_IN_DOCKER_HUB/gh_issue_summarization:0.1 \
      --replicas=2
    ks apply default -c issue-summarization-model

> __Warning__:
> The model can take quite some time to become ready due to the loading times of the models and may be restarted if it fails the default liveness probe. If this happens you can add a custom livenessProbe to the issue-summarization.jsonnet file. Add the below to the container section:

    "livenessProbe": {
      "failureThreshold": 3,
      "initialDelaySeconds": 30,
      "periodSeconds": 5,
      "successThreshold": 1,
         "handler" : {
	    "tcpSocket": {
               "port": "http"
             }
      },


and that's it! But wait ... is it really running? Let's test it

# Testing it

Run:

    kubectl port-forward svc/ambassador -n kubeflow 8080:80

    curl -X POST -H 'Content-Type: application/json' -d '{"data":{"ndarray":[["issue overview add a new property to disable detection of image stream files those ended with -is.yml from target directory. expected behaviour by default cube should not process image stream files if user does not set it. current behaviour cube always try to execute -is.yml files which can cause some problems in most of cases, for example if you are using kuberentes instead of openshift or if you use together fabric8 maven plugin with cube"]]}}' http://localhost:8080/seldon/issue-summarization/api/v0.1/predictions


Expected output:

    {
      "meta": {
        "puid": "d8e0ljfmf0apl1b205vr7lddqs",
        "tags": {
         },
         "routing": {
         },
         "requestPath": {
           "issue-summarization": "docker.io/jordimassaguerpla/gh_issue_summarization:0.3"
          },
          "metrics": []
       },
       "data": {
         "names": ["t:0"],
         "ndarray": [["add a build from the the the error"]]
     }

> __TIP__
The app won't work until "everything" is running and available, so the above test may fail. In order to see that setup has finished, look for the "issue-summarization-" pods status:

    watch kubectl get pods -n kubeflow                                                                                                                                                                                                       
    linux-ru6c: Fri Jun 28 19:18:21 2019

    NAME                                                              READY   STATUS             RESTARTS   AGE                                                                                                                                                                    
    ambassador-784d4d757-7cwrd                                        1/1     Running            0          26h                                                                                                                                                                    
    ambassador-784d4d757-qzwm4                                        1/1     Running            0          26h                                                                                                                                                                    
    ambassador-784d4d757-xl6wf                                        1/1     Running            0          26h                                                                                                                                                                    
    argo-ui-77d64677b9-9fvb8                                          1/1     Running            0          26h                                                                                                                                                                    
    centraldashboard-dd96bb455-j7nrj                                  1/1     Running            0          26h                                                                                                                                                                    
    issue-summarization-issue-summarization-e61c3df-76dff9b5b99vmbg   2/2     Running            0          58m                                                                                                                                                                    
    issue-summarization-issue-summarization-e61c3df-76dff9b5b9s7k2f   2/2     Running            0          58m                                                                                                                                                                    
    jupyter-web-app-6f946577d5-kkr8b                                  1/1     Running            0          26h                                                                                                                                                                    
    katib-ui-54b8484d4b-kz2t4                                         1/1     Running            0          26h                                                                                                                                                                    
    metacontroller-0                                                  1/1     Running            0          26h                                                                                                                                                                    
    minio-5f6b9b5b7-tg9kt                                             0/1     Pending            0          26h                                                                                                                                                                    
    ml-pipeline-79474f7b99-jnb4n                                      1/1     Running            192        26h                                                                                                                                                                    
    ml-pipeline-persistenceagent-c56c89b98-hjwkl                      0/1     CrashLoopBackOff   212        26h                                                                                                                                                                    
    ml-pipeline-scheduledworkflow-855c759488-snvm9                    1/1     Running            0          26h                                                                                                                                                                    
    ml-pipeline-ui-5bd6b6dd4-lq2kj                                    1/1     Running            0          26h                                                                                                                                                                    
    ml-pipeline-viewer-controller-deployment-5b997b657d-gwd5l         1/1     Running            0          26h                                                                                                                                                                    
    mysql-657f87857d-7r4l5                                            0/1     Pending            0          26h 









