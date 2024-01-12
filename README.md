# isolet
Isolet is a framework to deploy linux wargames like [Bandit](https://overthewire.org/wargames/bandit/). It uses pre-configured templates to provide isolated instance using kubernetes pods for each user.

## Contents

- [Features](#features)
- [Tech Stack](#tech-stack)
- [Setup](#setup)
    - [Additional setup](#additional-setup)
- [Configuration](#configuration)
    - [General](#general)
    - [Instance](#instance)
    - [Secrets](#secrets)
- [API routes](#api-routes)

## Features

* Isolated access to each pod
* Kubernetes manages the pods and scaling
* Limited time for each instance
* Instances can be extended
* Run time variables to avoid rebuilding images
* Email verification for registration

## Tech Stack

* Go
* TypeScript
* ReactJS
* TailwindCSS
* Kubernetes

## Setup

1. Install [kubectl](https://kubernetes.io/docs/tasks/tools/) on your machine. 
2. Spin up a cluster on your favourite cloud provider or if you wish to test locally, install [minikube](https://minikube.sigs.k8s.io/docs/start/).

Here is a sample gcloud command line for the cluster

```sh
gcloud beta container --project <PROJECT_NAME> clusters create <CLUSTER_NAME> --no-enable-basic-auth --cluster-version "1.27.7-gke.1121000" --release-channel "regular" --machine-type "e2-medium" --image-type "COS_CONTAINERD" --disk-type "pd-balanced" --disk-size "30" --node-labels app=node --metadata disable-legacy-endpoints=true --scopes "https://www.googleapis.com/auth/devstorage.read_only","https://www.googleapis.com/auth/logging.write","https://www.googleapis.com/auth/monitoring","https://www.googleapis.com/auth/servicecontrol","https://www.googleapis.com/auth/service.management.readonly","https://www.googleapis.com/auth/trace.append" --num-nodes "1" --logging=SYSTEM,WORKLOAD --monitoring=SYSTEM --enable-ip-alias --network "projects/<PROJECT_NAME>/global/networks/default" --subnetwork "projects/<PROJECT_NAME>/regions/<PROJECT_REGION>/subnetworks/default" --no-enable-intra-node-visibility --default-max-pods-per-node "110" --security-posture=standard --workload-vulnerability-scanning=disabled --enable-dataplane-v2 --no-enable-master-authorized-networks --addons HorizontalPodAutoscaling,HttpLoadBalancing,GcePersistentDiskCsiDriver --enable-autoupgrade --enable-autorepair --max-surge-upgrade 1 --max-unavailable-upgrade 0 --binauthz-evaluation-mode=DISABLED --enable-managed-prometheus --enable-shielded-nodes --tags "isolet-node" --node-locations "us-central1-c"
```

Change the instance configuration of nodes as per your workload requirements.

> [!note]
> Check out [gcloud reference](https://cloud.google.com/sdk/gcloud/reference/container/) for more information.

3. Configure `kubectl` to access the cluster.

### Additional setup

This setup is specific for Standard [GKE](https://cloud.google.com/kubernetes-engine/) cluster. Check out the documentation of your service provider for specifics.

- `StorageClass`
Isolet by default uses `standard-rwo` which supports only `ReadWriteOnce` the volume can be mounted as read-write by a single node. To change the `StorageClass` check the options available in your cluster

```sh
kubectl get sc
```

Choose an appropriate one and replace `standard-rwo` it in the [db-volume.yml](./kubernetes/init/db-volume.yml) file.

- `NodePort`
Isolet uses service of type `NodePort` to expose pods for the user. You might need to configure your cloud services to allow traffic into the instances.

While creating node you need to add network tag to the node. Like in the sample above nodes are being added `isolet-node`.

```sh
gcloud compute firewall-rules create kube-node-port-fw-rule \
    --action allow \
    --target-tags isolet-node \
    --source-ranges 0.0.0.0/0 \
    --rules tcp:30000-32767 \
    --no-enable-logging
```

## Configuration
You can customize the application using environment variables passed to the deployments. All the options are available in [configuration](/kubernetes/configuration)

### General

* `WARGAME_NAME` Name of the wargame to be deployed

* `PUBLIC_URL` URL of the deployed application. Required for email verification

* `PROXY_SERVER_NAME` Domains and subdomains to be added under server_name directive in nginx proxy.

> [!note]
> Check out the nginx documentation for format [server_name](https://nginx.org/en/docs/http/server_names.html)

* `INSTANCE_HOSTNAME` Domain name for accessing the spawned instances

* `IMAGE_REGISTRY_PREFIX` Default registry for pulling challenge images. It is prefixed to `level` to get final repo url to be 

> [!important]
> If you specify your prefix to be `docker.io/thealpha16/` the final image path for level 1 will be `docker.io/thealpha16/level1`

* `DISCORD_FRONTEND` Boolean to determine whether API needs `/auth` routes to be setup. If it is true, API will not authenticate the request.

> [!warning]
> This option should be used only when the API is not exposed to the public and the request is being forwarded by some other application which is properly authenticating

* `KUBECONFIG_FILE_PATH` Path to the kubernetes config file to access cluster from outside

> [!note]
> for more information, check out [cluster access](https://kubernetes.io/docs/tasks/access-application-cluster/access-cluster/)

* `UI_URL`
host for frontend in case it exists. If kubernetes is being used for deployment, you can specify URL to be

```
<SERVICE_NAME_OF_UI>.<NAMESPACE>.svc.cluster.local
```

> [!note]
> for more information, head over to [dns for pods](https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/)

### Instance

* `INSTANCE_NAMESPACE` Namespace for deploying the user requested pods.

* `CONCURRENT_INSTANCES`  Number of concurrent pods that user can spawn.

* `TERMINATION_PERIOD` 
Time in seconds to be given to the pod for graceful shutdown.

* `INSTANCE_TIME` Time in minutes to be added in the pod annotations after which ripper will remove the instance

* `MAX_INSTANCE_TIME` Time in minutes the user can extend the instance

* `CPU_REQUEST` Number of cores to be reserved for the pod

* `CPU_LIMIT` Maximum number of cores the pod can consume

* `MEMORY_REQUEST` Amount of memory to be reserved for the pod

* `MEMORY_LIMIT` Maximum amount of memory the pod can use

* `DISK_REQUEST` Disk space to be reserved for the pod

* `DISK_LIMIT` Maximum disk space the pod can utilize

> [!note]
> for more information regarding kubernetes resources, check out [resources](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/)

### Secrets

> [!note]
> Secrets should be base64 encoded

* `SESSION_SECRET` Key used for signing jwt token after login

* `TOKEN_SECRET` Key used for signing verification token sent to mail

* `INSTANCE_NAME_SECRET` Key used for determining unique instance name for the pods

* `EMAIL_ID` Email ID to be used when sending verification mails

* `EMAIL_AUTH` Password for authenticating to use smtp service

* `DB_HOST` Host name of the database server

* `DB_USER` Username for the database

* `DB_PASSWORD` Password to authenticate to the database

* `DB_NAME` Name of the database

## API routes
| route | methods | parameters | response | sample |
|:---:|:---:|:---:|:---:|:---:|
| /api/challs | GET | NONE | challenges | [{"chall_id":0, "level":1, "name":"demo", "prompt":"solve it", "tags":["ssh", "cat"]}] |
| /api/launch | POST | chall_id, userid, level | status | {"status": "success", "message": "3b369c0b1fd5419b2f81da89cf5480d2 32747"} |
| /api/stop | POST | userid, level | status | {"status": "failure", "message": "User does not exist"} |
| /api/submit | POST | userid, level, flag | status | {"status": "failure", "message": "Flag copy detected. Incident reported!"} |
| /api/status | GET | NONE | instances | {"status": "success", "message": "[{"userid":123614343, "level":1, "password":"8f1ee93113affe32078c", "port":"32134"}]"}
