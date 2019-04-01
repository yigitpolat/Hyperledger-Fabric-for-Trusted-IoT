# Hyperledger Fabric for Trusted IoT

## Architecture Flow

## Included Components

* [IBM Cloud Kubernetes Service](https://www.ibm.com/cloud/container-service) delivers powerful tools by combining Docker containers, the Kubernetes technology, an intuitive user experience, and built-in security and isolation to automate the deployment, operation, scaling, and monitoring of containerized apps in a cluster of compute hosts.
* [Node-RED](https://nodered.org/) is a programming tool for wiring together hardware devices, APIs and online services in new and interesting ways.

## Featured Technologies

* [Hyperledger Fabric v1.4](https://hyperledger-fabric.readthedocs.io/en/release-1.4/) is a platform for distributed ledger solutions underpinned by a modular architecture delivering high degrees of confidentiality, resiliency, flexibility, and scalability.
* [GoLang](https://golang.org) is an open source programming language that makes it easy to build simple, reliable, and efficient software.
* [Node.js](https://nodejs.org/en/) is an open source, cross-platform runtime environment for developing server-side and networking applications.

## Running the application

### Prerequisites

* [IBM Cloud Account](https://cloud.ibm.com/)
* [IBM Cloud CLI](https://cloud.ibm.com/docs/cli)
* [Docker](https://docs.docker.com/install/)
* [Texas Instruments SensorTag](http://www.ti.com/tools-software/sensortag.html#)

> Note: If you are not able to provide SensorTag, it is possible to generate dummy data in the later steps.

### Steps

1. [Check installation prerequisites and clone the repo](#1-check-installation-prerequisites-and-clone-the-repos)
2. [Create and Access IBM Cloud Kubernetes Cluster](#2-create-and-access-ibm-cloud-kubernetes-cluster)
3. [Deploy Hyperledger Fabric](#3-deploy-hyperledger-fabric)
4. [Deploy Hyperledger Fabric SDK for Node.js](#4-deploy-hyperledger-fabric-sdk-for-nodejs)
5. [Deploy Node-RED](#5-deploy-node-red)

### 1. Check installation prerequisites and clone the repo

Open a new Terminal window and execute following commands to be sure you have installed prerequisites.

```bash
    docker version
    ibmcloud --version
```

Clone this repository in a folder your choice:

```bash
    git clone https://...
    cd ..
```

### 2. Create and Access IBM Cloud Kubernetes Cluster

As Hyperledger Fabric is a network consist of several components, we use microservice architecture on IBM Cloud Kubernetes Service.

#### 2.1. Create a Kubernetes Cluster on IBM Cloud

* Create a Kubernetes Cluster from [here](https://cloud.ibm.com/kubernetes/catalog/cluster/create).

> Note1: Depending on your IBM Cloud Account type you can either create Free or Standard Cluster.
>
> Note2: If you chose Standard Cluster type, it is recommended to create your cluster in North America, where the worker zone is Dallas.
>
> Note3: It can take up to 15 minutes for the cluster to be set up and provisioned.

#### 2.2. Gain access to your Cluster

Once your cluster provisioned (status set to normal), perform the “Gain Access to your Cluster” steps.

* Execute the following command to verify that the kubectl commands run properly.

```bash
    kubectl get nodes
```

<p align="center"><img src="docs/screen1.gif"></p>

### 3. Deploy Hyperledger Fabric

#### 3.1. Create PV and PVC

A PersistentVolume (PV) is a piece of storage in the cluster that has been provisioned by an administrator. A PersistentVolumeClaim (PVC) is a request for storage by a user. We will use this storage to store our configuration files and chaincode.

* If you have Free Cluster use the following command.

```bash
    kubectl create -f createPVandPVC.yaml
```

* If you have Standard Cluster, first change the region and zone variables inside the createPVC.yaml according to your cluster location and use the following command.

```bash
    kubectl create -f createPVC.yaml
```

> Note: You have to wait until the PVC will be bound to a storage.

<p align="center"><img src="docs/screen2.png"></p>

#### 3.2. Copy Artifacts

Once our PV and PVC are created, you are able to copy the local files to the storage on the cloud.

```bash
    cd jobs
    kubectl apply -f copyArtifactsJob.yaml
    pod=$(kubectl get pods --selector=job-name=copyartifacts --output=jsonpath={.items..metadata.name})
    kubectl cp ../artifacts $pod:/shared/
    kubectl get pods -w
```

After your copyartifacts pod become completed, you can continue with the following step.

<p align="center"><img src="docs/screen3.png"></p>

#### 3.3. Generating Hyperledger Fabric key materials and channel config related artifacts

Cryptogen is an utility for generating Hyperledger Fabric key material. It is provided as a means of preconfiguring a network for testing purposes. The configtxgen command allows users to create and inspect channel config related artifacts.

> Note: In the following steps, wait your jobs status to become completed to prevent any conflicts. You can see your pod status by executing "kubectl get pods"

* The following command will generate MSPs (Membership Service Providers)

```bash
    kubectl apply -f generateCryptoConfig.yaml
```

* The following command will generate 'genesis.block' which will be used to deploy Orderer.

```bash
    kubectl apply -f generateGenesisBlock.yaml
```

* The following command will generate 'channel1.tx' which will be used to create channel.

```bash
    kubectl apply -f generateChanneltx.yaml
```

* The following command will generate 'Org1MSPanchors.tx' and 'Org2MSPanchors.tx' which will be used to set the Anchor Peers in the network.

```bash
    kubectl apply -f generateAnchorPeerMSPs.yaml
```

> Note: A peer node on a channel that all other peers can discover and communicate with. Each Member on a channel has an anchor peer (or multiple anchor peers to prevent single point of failure), allowing for peers belonging to different Members to discover all existing peers on a channel.

<p align="center"><img src="docs/screen4.png"></p>

#### 3.4. Network Deployment

You have completed prerequired steps for the network deployment. Now, you will deploy the Hyperledger Fabric components, Certificate Authority, Orderer, and Peers to your cluster.

> Note: In the following step, wait your deployments status to become running to prevent any conflicts. You can see your pod status by executing "kubectl get pods"

```bash
    cd ../network-deployment
    sh deployAll.sh
```

<p align="center"><img src="docs/screen5.png"></p>

#### 3.5. Network Configuration

You should have your Hyperledger Fabric components are running. In the following steps, we will configure these components according to our use case.

> Note: In the following steps, wait your jobs status to become completed to prevent any conflicts. You can see your pod status by executing "kubectl get pods"

* The following command will create a channel named 'channel1'

```bash
    cd ../jobs
    kubectl apply -f create_channel.yaml
```

* The following command will join all the peers to the 'channel1'

```bash
    kubectl apply -f join_channel.yaml
```

<p align="center"><img src="docs/screen6.png"></p>

* The following command will install chaincode to peers (org1peer2, org2peer2). These peers will be your endorser peer.

```bash
    kubectl apply -f chaincode_install.yaml
```

* The following command will instantiate the installed chaincode to the channel. Besides, sets the endorsement policy as requests 1 signature from each of the two organizations.

```bash
    kubectl apply -f chaincode_instantaite.yaml
```

* The following command will update the channel and will set peers (org1peer1, org2peer1) as Anchor Peers.

```bash
    kubectl apply -f updateAnchorPeers.yaml
```

<p align="center"><img src="docs/screen7.png"></p>

### 4. Deploy Hyperledger Fabric SDK for Node.js

Up until now, you have developed Hyperledger Fabric Network which might be a backend for an application. However, we need a middleware in order to connect the back-end and the front-end. For this purpose you will use Hyperledger Fabric Client SDK for Node.js which makes it possible to use APIs to interact with a Hyperledger Fabric blockchain.

> Note: For the following steps, you must have 'DockerHub Account' in order to push and pull your container images. You can create an account from [here](https://hub.docker.com).

* The following commands will create a container image and push it to your container registery.

```bash
    cd ../API
    docker build . -t <your_account_name>/rest-api
    docker push <your_account_name>/rest-api
```

<p align="center"><img src="docs/screen8.png"></p>
<p align="center"><img src="docs/screen9.png"></p>

* The following commands will first pull the container image from your repository and create a deployment named "rest-api", then create a Kubernetes Service which exposes this deployment

```bash
    cd ..
    kubectl run rest-api --image=<your_account_name>/rest-api --port=3000
    kubectl apply -f rest-api-svc.yaml
```

<p align="center"><img src="docs/screen10.png"></p>

### 5. Deploy Node-RED

Node-RED dashboard will be your front-end. You will be able to see incoming sensor data and the history of the ledger from this dashboard. Besides, all the HTTP requests will be execute via this tool.

> Note: There is Node-RED service in the IBM Cloud Catalog. However, in this pattern you will use Node-RED inside a container.

* The following commands will first pull the container image from DockerHub and create a deployment named "nodered", then create a Kubernetes Service which exposes this deployment

```bash
    TODO: change the image user names, sensor id
    kubectl run nodered --image=yigitpolat/kworks-nodered --port=1880
    kubectl apply -f node-red-svc.yaml
```

## Extending Code Pattern
TODO: chaincode, sdk, ui
