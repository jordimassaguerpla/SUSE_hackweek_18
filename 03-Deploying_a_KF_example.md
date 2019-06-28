---- DRAFT ----

This is just random notes that need to be rewritten...

If you have followed the previous steps [01](https://github.com/jordimassaguerpla/SUSE_hackweek_18/blob/master/01-How_to_setup_SUSE_CaaSP_kubernetes_crio_GPU.md)
[02](https://github.com/jordimassaguerpla/SUSE_hackweek_18/blob/master/02-How_to_deploy_kubeflow.md),
you will have now kubeflow deployed on SUSE CaaSP 4 with GPU acceleration.

We will now deploy a ML "pipeline", meaning:

1- Load a Jupyter notebook with a definition of the model using Keras and terraform
2- Run the Jypyter notebook to train a model
3- Extract the model and use that to deploy a microservice using ....

Example that we will follow:

https://github.com/kubeflow/examples/tree/master/github_issue_summarization

# 1
based on: https://github.com/kubeflow/examples/blob/master/github_issue_summarization/01_setup_a_kubeflow_cluster.md

Install ksonet following instructions on ^

DO NOT USE THEIR IMAGE, is not available. Instead open the jupyter console and run

pip3 install --upgrade pip
pip3 install -U scikit-learn \
    && pip3 install -U ktext \
    && pip3 install -U IPython \
    && pip3 install -U annoy \
    && pip3 install -U tqdm \
    && pip3 install -U nltk \
    && pip3 install -U matplotlib \
    && pip3 install -U tensorflow \
    && pip3 install -U bernoulli \
    && pip3 install -U h5py \
    && git clone https://github.com/google/seq2seq.git \
&& pip3 install -e ./seq2seq/ 
git clone https://github.com/kubeflow/examples.gif
mkdir tmp/github-issues-data

Load the examples/github_issue_summarization/notebooks/Training.ipynb notebook

In the notebook, replace the 

%env DATA_DIR=/mnt/github-issues-data
by
%env DATA_DIR=/tmp/github-issues-data

because in our demo we don't have a persistent volume mounted in mnt

We have the notebook, let's train now!!

based on https://github.com/kubeflow/examples/blob/master/github_issue_summarization/02_training_the_model.md

Open the Training.ipynb notebook. This contains a complete walk-through of downloading the training data, preprocessing it, and training it.

Run the Training.ipynb notebook, viewing the output at each step to confirm that the resulting models produce sensible predictions.

After training completes, download the resulting files to your local machine. The following files are needed for serving results:

    seq2seq_model_tutorial.h5 - the keras model
    body_pp.dpkl - the serialized body preprocessor
    title_pp.dpkl - the serialized title preprocessor

(download from the jupyter file browser)


SERVING THE MODEL

based on https://github.com/kubeflow/examples/blob/master/github_issue_summarization/03_serving_the_model.md

We are going to use Seldon Core to serve the model. IssueSummarization.py contains the code for this model. We will wrap this class into a seldon-core microservice which we can then deploy as a REST or GRPC API server.

We will build our image based on the files downloaded from the notebook

We need to install s2i

cp the files into ..../notebooks

run

    s2i build -E environment_seldon_rest . seldonio/seldon-core-s2i-python36:0.4 docker.io/YOUR_USERNAME_IN_DOCKER_HUB/gh_issue_summarization:0.1

docker login #this logs you into docker hub
docker push docker.io/YOUR_USERNAME_IN_DOCKER_HUB/gh_issue_summarization:0.1

>> You could use a different registry ....
>> If you don't copy the files, the docker image won't contain them!!!

now you can use seldon core...

cd ks_app
# Generate the seldon component and deploy it
ks generate seldon seldon
ks apply default -c seldon


Seldon Core should now be running on your cluster. You can verify it by running kubectl get pods -n${NAMESPACE}. You should see two pods named seldon-seldon-cluster-manager-* and seldon-redis-*.


ks generate seldon-serve-simple-v1alpha2 issue-summarization-model \
  --name=issue-summarization \
  --image=docker.io/YOUR_USERNAME_IN_DOCKER_HUB/gh_issue_summarization:0.1 \
  --replicas=2
ks apply default -c issue-summarization-model


The model can take quite some time to become ready due to the loading times of the models and may be restarted if it fails the default liveness probe. If this happens you can add a custom livenessProbe to the issue-summarization.jsonnet file. Add the below to the container section:

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


kubectl port-forward svc/ambassador -n kubeflow 8080:80

jordi@linux-ru6c:~> curl -X POST -H 'Content-Type: application/json' -d '{"data":{"ndarray":[["issue overview add a new property to disable detection of image stream files those ended with -is.yml from target directory. expected behaviour by default cube should not process image stream files if user does not set it. current behaviour cube always try to execute -is.yml files which can cause some problems in most of cases, for example if you are using kuberentes instead of openshift or if you use together fabric8 maven plugin with cube"]]}}' http://10.99.240.24:9000/api/v0.1/predictions                                                                                                                                                                                                                              
<!DOCTYPE HTML PUBLIC "-//W3C//DTD HTML 3.2 Final//EN">
<title>405 Method Not Allowed</title>
<h1>Method Not Allowed</h1>
<p>The method is not allowed for the requested URL.</p>
jordi@linux-ru6c:~> curl -X POST -H 'Content-Type: application/json' -d '{"data":{"ndarray":[["issue overview add a new property to disable detection of image stream files those ended with -is.yml from target directory. expected behaviour by default cube should not process image stream files if user does not set it. current behaviour cube always try to execute -is.yml files which can cause some problems in most of cases, for example if you are using kuberentes instead of openshift or if you use together fabric8 maven plugin with cube"]]}}' http://localhost:8080/seldon/issue-summarization/api/v0.1/predictions
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


TIP:

See when the "issue-s...." are running before running the curl command

watch kubectl get pods -n kubeflow                                                                                                                                                                                                   linux-ru6c: Fri Jun 28 19:18:21 2019

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









