# Tekton Pipelines CiCD with Maven Caching Dependencies

This is an example to show how to use workspaces in Tekton Pipelines for caching Maven dependencies and reducing build time by avoiding to download dependencies on every pipeline run.


```
kubectl create -f pvc.yaml
kubectl create -f maven-task.yaml
kubectl create -f maven-pipeline.yaml
kubectl create -f maven-pipelinerun.yaml
```