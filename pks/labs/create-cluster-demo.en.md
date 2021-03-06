# Create Kubernetes Clusters

## Goal

Login to PKS to create a Kubernetes cluster and connect to it.

## Prerequisites

  - Deployed Pivotal Ops Manager (2.0+)
  - Deployed Pivotal Director Tile
  - Post-deploy scripts enabled
  - Deployed Pivotal Container Service Tile
  - Installed PKS CLI

## Determine PKS Management API IP

In order to reach the PKS Management API service we must ensure either our DNS or Load Balancer is sending us to the correct PKS Management API url.
We cannot utilize IPs due to the default security enforcement built into the PKS Management API.
**Skip this step if you deployed a load balancer and/or already registered the PKS Management API hostname with your DNS.**

1. Navigate to the Ops Manager FQDN (hostname).
1. Login with the Ops Manager credentials setup earlier.
1. Click on Pivotal Container Service Tile.
1. Click on Status.
1. Note the IP for the Pivotal Container Service job down.

## Login to PKS CLI

In order to create a Kubernetes Cluster with PKS we utilize the PKS CLI. Since PKS is backed by UAA for authorization, we use a preconfigured `admin` username and password stored with PKS UAA to log into PKS.
While external identity resources are supported for backing UAA this demo is focused on using local users stored in UAA.

  1. Navigate to the Ops Manager FQDN (hostname).
  1. Login with the Ops Manager credentials setup earlier.
  1. Click on Pivotal Container Service Tile.
  1. Click on Credentials.
  1. Find the **Uaa Admin Password** entry and click on Link to Credential.
  1. Note down the secret.
  1. Using the PKS CLI, login to the PKS API by typing `pks login -a <PKS-API-HOSTNAME> -u <UAA-USERNAME> -p <UAA-USER-PASSWORD> -k`
    - `PKS-API-HOSTNAME` comes from Step `Determine PKS Management API IP` above
    - `UAA-USERNAME` is `admin`
    - `UAA-USER-PASSWORD` is the secret looked up previously

  ```
  $ pks login -a api.pks.<domain-name> -u admin -p <admin_secret> -k
  API Endpoint: api.pks.<domain-name>
  User: admin
  ```

## Create a Kubernetes Cluster

  - Lookup the PKS plans configured in the PKS Tile using `pks plans`

    ```
    $ pks plans
    Name    ID                                    Description
    small   8A0E21A8-8072-4D80-B365-D1F502085560  Example: This plan will configure a lightweight kubernetes cluster. Not recommended for production workloads.
    medium  58375a45-17f7-4291-acf1-455bfdc8e371  Example: This plan will configure a medium sized kubernetes cluster, suitable for more pods.
    ```
  - Provision a Kubernetes Cluster by typing `pks create-cluster <CLUSTER-NAME> --external-hostname <cluster-name.domain-name> --plan <plan-name>`
    - The `CLUSTER-NAME` can be replaced with any name you choose
    - Note: The `--external-hostname` flag determines which hostname PKS will add to the certificate to authenticate to the Kubernetes cluster with. The hostname you place here will have to be resolved and load-balanced to the Kubernetes Master IP.

  ```
  $ pks create-cluster pks-wmt -e pks-wmt.pks.<domain-name> -p small
  Name:                     pks-wmt
  Plan Name:                small
  UUID:                     91c80e8c-7a9c-4bb8-bfb4-f4a54e2a4f72
  Last Action:              CREATE
  Last Action State:        in progress
  Last Action Description:  Creating cluster
  Kubernetes Master Host:   pks-wmt.pks.<domain-name>
  Kubernetes Master Port:   8443
  Worker Nodes:             3
  Kubernetes Master IP(s):  In Progress
  Network Profile Name:     

  Use 'pks cluster pks-wmt' to monitor the state of your cluster
  ```
  - Watch the status of our cluster. It can take up to 10 minutes to create the cluster.

  ```
  $ pks cluster pks-wmt
  Name:                     pks-wmt
  Plan Name:                small
  UUID:                     91c80e8c-7a9c-4bb8-bfb4-f4a54e2a4f72
  Last Action:              CREATE
  Last Action State:        succeeded
  Last Action Description:  Instance provisioning completed
  Kubernetes Master Host:   pks-wmt.pks.<domain-name>
  Kubernetes Master Port:   8443
  Worker Nodes:             3
  Kubernetes Master IP(s):  10.195.1.128
  Network Profile Name:  
  ```

## Access the Kubernetes Cluster

  - Once cluster if finished creating, you need to create a DNS entry for the `external-hostname` you indicated while creating the cluster to point to the Master IP shown in the status output
  - Finally, you can issue `pks get-credentials <CLUSTER-NAME>`, which creates an entry in your local `~/.kube/config` file to be able to interact with Kubernetes cluster
    - Enter the `UAA-USER-PASSWORD` that we looked up previously when prompted for Password.

  ```
  $ pks get-credentials pks-wmt
  Fetching credentials for cluster pks-wmt.
  Password: ******
  Context set for cluster pks-wmt.

  You can now switch between clusters by using:
  $kubectl config use-context <cluster-name>
  ```
  - Confirm `kubectl` is working by invoking a command such as `kubectl get nodes`

  ```
  $ kubectl get nodes -o wide
  NAME                                   STATUS   ROLES    AGE   VERSION   INTERNAL-IP   EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION      CONTAINER-RUNTIME
  38d045a2-2f3f-4ad1-a8aa-acc622e8ac99   Ready    <none>   1d    v1.11.3   172.24.0.3    172.24.0.3    Ubuntu 16.04.5 LTS   4.15.0-33-generic   docker://17.12.1-ce
  6663dac2-ff16-4a73-909f-4a1f10be3c4a   Ready    <none>   1d    v1.11.3   172.24.0.4    172.24.0.4    Ubuntu 16.04.5 LTS   4.15.0-33-generic   docker://17.12.1-ce
  b0b26691-62ac-42ea-84ed-234cce7deea4   Ready    <none>   1d    v1.11.3   172.24.0.5    172.24.0.5    Ubuntu 16.04.5 LTS   4.15.0-33-generic   docker://17.12.1-ce
  ```
