# Openshift Pipeline Demo with Proxy

## Install Openshift Pipelines

```
helm upgrade --install openshift-pipelines openshift-pipelines
```

## Create project for demo

Create a project for the sample application that you will be using in this demo:

```
$ oc new-project pipelines-tutorial
```

NOTE: Be careful with your LimitRanges because the pods of the Tasks needed at least 1G per container in several steps (build mainly).

OpenShift Pipelines automatically adds and configures a ServiceAccount named pipeline that has sufficient permissions to build and push an image. This service account will be used later:

```
$ oc get serviceaccount pipeline
```

## Install Tasks

Install the apply-manifests, update-deployment, git-clone and buildah tasks from the repository using oc or kubectl, which you will need for creating a pipeline in the next section:

```
$ oc apply -f resources/01_apply_manifest_task.yaml
```

```
$ oc apply -f resources/02_update_deployment_task.yaml
```

```
$ tkn task ls
NAME                DESCRIPTION              AGE
apply-manifests                              20 minutes ago
buildah             Buildah task builds...   8 seconds ago
git-clone           These Tasks are Git...   9 minutes ago
update-deployment                            18 minutes ago

```

NOTE: we created the buildah, and the git-clone as local Tasks and not used ClusterTasks because we needed to modify the tasks and to support the HTTP Proxy parameters.

When a task starts running, it starts a pod and runs each step sequentially in a separate container on the same pod. 
Tasks can have multiple steps, and, since they run within the same pod, they have access to the same volumes in order to cache files, access configmaps, secrets, etc. You can specify volume using workspace. It is recommended that Tasks uses at most one writeable Workspace. Workspace can be secret, pvc, config or emptyDir.

In our case we need to provision a PVC to share the workspace between the different pods in the steps of the pipelines:

```
$ oc apply -f resources/05_persistent_volume_claim.yaml

$ oc get pvc
NAME         STATUS    VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS   AGE
source-pvc   Pending                                      gp2            21s
```

## Create Pipeline 

A pipeline defines a number of tasks that should be executed and how they interact with each other via their inputs and outputs.

This pipeline helps you to build and deploy backend/frontend, by configuring right resources to pipeline.

Pipeline Steps:

* Clones the source code of the application from a git repository by referring (git-url and git-revision param)
* Builds the container image of application using the buildah clustertask that uses Buildah to build the image
* The application image is pushed to an image registry by refering (image param)
* The new application image is deployed on OpenShift using the apply-manifests and update-deployment tasks.


Create the pipeline by running the following:

```
oc apply -f resources/06_pipeline.yaml
```

Check the list of pipelines you have created using the CLI:

```
$ tkn pipeline ls
NAME               AGE             LAST RUN   STARTED   DURATION   STATUS
build-and-deploy   3 minutes ago   ---        ---       ---        ---
```


## Trigger Pipeline

A PipelineRun is how you can start a pipeline and tie it to the persistentVolumeClaim and params that should be used for this specific invocation.

```
$ oc apply -f pipelinerun/01_build_deploy_api_pipelinerun.yaml
```

```
$ oc apply -f pipelinerun/02_build_deploy_ui_pipelinerun.yaml
```

```
 tkn pipeline ls
NAME               AGE          LAST RUN                       STARTED          DURATION   STATUS
build-and-deploy   1 hour ago   build-deploy-api-pipelinerun   2 minutes ago   1 minute   Succeeded
```

```
$ tkn pr ls
NAME                          STARTED         DURATION   STATUS
build-deploy-ui-pipelinerun   6 seconds ago   ---        Running
build-and-deploy-0mgs0g       4 minutes ago   1 minute   Succeeded
```

After a few minutes, the pipeline should finish successfully.

You can get the route of the application by executing the following command and access the application

```
$ oc get route vote-ui --template='http://{{.spec.host}}'
```