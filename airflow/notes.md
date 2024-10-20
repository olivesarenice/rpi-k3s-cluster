# Install

Use the community chart which is the stable version - other helm charts from guides may not work:
https://github.com/airflow-helm/charts/blob/main/charts/airflow/docs/guides/quickstart.md

    ## set the release-name & namespace
    export AIRFLOW_NAME="airflow-cluster"
    export AIRFLOW_NAMESPACE="airflow-cluster"

    ## create the namespace
    kubectl create ns "$AIRFLOW_NAMESPACE"

## K8s dependencies


First we need to pass the git ssh private key to K8s as a Secret. 


The corresponding public key will need to be added to the keys section in Github or whichever Git platform is being used.


Add the base64 encoded private key to the `airflow-ssh-secret.yaml` file

    cat  ~/.ssh/id_rsa | base64 -w 0

Copy it into `airflow-ssh-secret.yaml` github ssh key

Then apply:

    kubectl apply -f airflow-ssh-secret.yaml

## Setting up PVs for logs

Since I am using k3s, the `local-path` default location will be in `/var/lib/rancher/k3s/storage/...` of the master node (Rpi)

Note on how storage (PV) provisioning works: https://www.fadhil-blog.dev/blog/rancher-local-path-provisioner/

How this will work:

First we need to make sure that the `local-path-provisioner` pod that comes shipped with k3s is configured correctly. By default, it's ConfigMap tells it to use:

    {
        "nodePathMap": [
            {
            "node": "DEFAULT_PATH_FOR_NON_LISTED_NODES",
            "paths": ["/var/lib/rancher/k3s/storage"]
            }
        ],
    }

But this forces the storage volume to only be mounted to a single pod at a time. We want the same storage to be shared across all pods in the node, so we need to do:

    {
        "nodePathMap":[],
        "sharedFileSystemPath":"/var/lib/rancher/k3s/storage"
    }

By sharing the file system, we enable the ReadWriteMany option for PVC.
Issue: https://github.com/rancher/local-path-provisioner/issues/274

First create the PV and the PVC. This will create 

    kubectl apply -f pv-airflow-logs.yaml 
    kubectl apply -f pvc-airflow-logs.yaml 


    ### Note that the PV may still be bound to an old PVC that was deleted whose ID no longer exists. To release it, go to the PV's values and delete the `claimRef` section and it should make it Available

    ## grant write permissions to the logs folder that was mounted (since we created it manually)

    Since I am running a local k3s, the PVs write directly to the master node's local path i.e. inside the RPi, it just writes directly to /opt/airflow/logs.

    So, either go to the RPi and grant

    cd /opt/airflow
    chmod -R 777 logs/

    OR run a pod that mounts the volume, then add permissions from there.

    kubectl apply -f pv-permissions.yaml -n airflow-cluster
    shell into to the pod (K9s), run 

    cd /opt/airflow
    chmod -R 777 logs/
    

    ## install using helm 3
    helm install \
    "$AIRFLOW_NAME" \
    airflow-stable/airflow \
    --namespace "$AIRFLOW_NAMESPACE" \
    --version "8.X.X" \
    --values ./custom_values.yaml
  



# Troubleshooting

1. Web server container is restarting `exit code: 15`. This indicates that the liveness/ readiness probes are timing out too quickly - possibly caused by having weaker RAM/CPU and the Postgres db taking longer to run all setups. 

    This should was sufficient for the RPi to complete the setup:
        
        readinessProbe:
            enabled: true
            initialDelaySeconds: 10
            periodSeconds: 30
            timeoutSeconds: 30
            failureThreshold: 600

        livenessProbe:
            enabled: true
            initialDelaySeconds: 5
            periodSeconds: 30
            timeoutSeconds: 60
            failureThreshold: 600

        startupProbe:
            enabled: true
            initialDelaySeconds: 5
            periodSeconds: 10
            timeoutSeconds: 15
            failureThreshold: 600

    Different between the probes: https://kubernetes.io/docs/concepts/configuration/liveness-readiness-startup-probes/

    https://apache-airflow.slack.com/archives/CCV3FV9KL/p1724236899651329

2. If pod keeps getting scheduled on another pod (master) despite setting a nodeSelector, it could be that the PV that was previously created and trying to PVC to was created on that tainted node, so there is conflict.

# Homelab

https://dev.to/gjrdiesel/the-ultimate-kubernetes-homelab-setup-558f