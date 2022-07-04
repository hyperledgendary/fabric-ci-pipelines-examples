# Using Tekton

This sample shows how Tekton tasks and pipelines can be configured to use the Ansible Collection for IBP. The tutorial network from the Ansible collection is used. It comprisies two endorsing ogranizations (org1/org2), along with a ordering organization. It is created in a series of playbooks, orchestrated by shell scripts. 

- Create the Ordering Organzation
- Create the (first) Endorsing organization 
- Create a channel
- Join the peer to the channel, and add as an anchor peer

- Create a (second) Endorsing ogranization
- Add this to the channel

- Deploy a chaincode

## Directory setup

The files here are (almost) identical copies of the ones from the test network. 

- ansible-vars
- ansible-templates

The variations are entirely down to the location of the variable files, and the use of secrets not varaible files for the api endpoint and key.

## Pre-reqs

Make sure you've access to a K8S cluster, and that Tekton is installed. Typically [installation of Tekton](https://tekton.dev/docs/getting-started/tasks/) is this command

```bash
kubectl apply --filename https://storage.googleapis.com/tekton-releases/pipeline/latest/release.yaml
```

Also install the [Tekton CLI](https://tekton.dev/docs/cli/)


You will need to have a created IBP Instance. This does not need to be in the same cluster as Tekton. A service credential needs to be created from the IBP Instance. Recommended to save this credential as a JSON file.

## Tekton Tasks and Pipelines


## Steps

### 1. Create K8S secrets

```
kubectl create secret generic ibp-creds --from-literal=api_endpoint=$(jq -r '.api_endpoint' service-creds.json) --from-literal=api_key=$(jq -r '.apikey' service-creds.json)
```

### 2. Create a PVC for storing working files

```
kubectl apply -f result-pvc.yaml
```

### 3. Create the organization variables

There are three organizations, "Ordering" "Org1" "Org2".  The tutorial splits these into separate files with a common file shared amongst all of them.   These can be loaded into K8S as Config Maps

```
kubectl create configmap ordering-org-vars --from-file=ansible-vars/common-vars.yml --from-file=org-vars.yml=ansible-vars/ordering-org-vars.yml
kubectl create configmap org1-vars --from-file=ansible-vars/common-vars.yml --from-file=org-vars.yml=ansible-vars/org1-vars.yml
kubectl create configmap org2-vars --from-file=ansible-vars/common-vars.yml --from-file=org-vars.yml=ansible-vars/org2-vars.yml
```

### 4. Templates
There are some templats used for creating policies, create these as a config map as well

```
kubectl create configmap common --from-file=ansible-templates
```

### 5. Build the nework.

```
kubectl apply -f build-network-task.yaml
```

Start the Tekton TaskRun
- using the persistence volume claim for outputs
- using the `ordering-org-vars` and `org1-vars` configuration configmaps
- using the `api-endpoint` and `api-key` stored in the secrets


```
tkn task start --use-param-defaults --workspace name=ansiblectx,claimName=ansible-results-pvc --param endorse-org-vars=org1-vars --param ordering-org-vars=ordering-org-vars --showlog build-network-task
```

