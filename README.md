# Fleet Multi Cluster Management (Local K8s Environment)
This repository contains the source code for multi cluster management setup with Fleet in a local environment.

![Fleet Logo](Fleet.png)

## Requirements/Prerequisites
- [Rancher Kubernetes Engine (RKE)](https://rancher.com/docs/rke/latest/en/installation/)
- [Virtual Box](https://www.virtualbox.org/wiki/Downloads)
- [Vagrant](https://www.vagrantup.com/docs/installation)
- [Helm](https://helm.sh/docs/intro/install/)

## Create Virtual Machines (VMs)
To spin up the virtual machines, run the following command at the root level of the project directory:
```
vagrant up
```
Once the VMs are up and running, you can check their status with `vagrant status` or by connecting to anyone of them using `vagrant ssh [hostname]`. Once you've confirmed that all machines are running with no issues, copy over the generated SSH keys from your workstation/host to each guest machine with the following commands:
```
ssh-copy-id root@[relevant ip address]
```
When prompted, enter the root user password configured in the bootstrap node script.

## Provision/Create Kubernetes cluster with RKE
To provision the cluster on the VMs, run the `rke config` command. You will be presented with a series of questions to which the answers will be used to declare the cluster config in a generated `cluster.yml` file upon completion. Alternatively, you can create the cluster.yml file and populate it with your desired configuration. Once you have the `cluster.yml` file, run the following command:
```
rke up
```
When the cluster has been provisioned, the following files will be generated in the root directory:
- cluster.rkestate - the cluster state file 
- kube_config_cluster.yml - kube config file

To add the cluster to your context, copy the kube config file:
```
cp kube_config_cluster.yml ~/.kube/config
```
If you do not have a `./kube` directory on your machine you will have to create one. 

The last step will be to check that you can connect to your cluster:
```
kubectl cluster-info
```
or
```
kubectl config current-context
```
If you have a tool like [K9s](https://k9scli.io/), you can start it up to see the objects that are currently running in your cluster. 

## Fleet Controller Cluster & Downstream Cluster Registration
In order for your Fleet multi cluster management installation to properly work it is important the correct API server URL and CA certificates are configured properly. The Fleet agents will communicate to the Kubernetes API server URL. This means the Kubernetes API server must be accessible to the downstream clusters. You will also need to obtain the CA certificate of the API server.

Decode *certificate-authority-data* field from kubeconfig:
```
kubectl config view -o json --raw  | jq -r '.clusters[].cluster["certificate-authority-data"]' | base64 -d > ca.pem
```

### Setup the environment with your specific values
```
API_SERVER_URL="https://172.16.128.21:6443"
API_SERVER_CA="ca.pem"
```

### Install Fleet CustomResourcesDefintions
```
helm -n fleet-system install --create-namespace --wait fleet-crd https://github.com/rancher/fleet/releases/download/v0.3.3/fleet-crd-0.3.3.tgz
```

### Install Fleet Controllers
```
helm -n fleet-system install --create-namespace --wait \
    --set apiServerURL="${API_SERVER_URL}" \
    --set-file apiServerCA="${API_SERVER_CA}" \
    fleet https://github.com/rancher/fleet/releases/download/v0.3.3/fleet-0.3.3.tgz
```

### Create Registration Token
Cluster registration tokens are used to establish a new identity for a cluster. Create the token below on the cluster with the Fleet manager.
```
kind: ClusterRegistrationToken
apiVersion: "fleet.cattle.io/v1alpha1"
metadata:
  name: new-token
  namespace: clusters
spec:
  # A duration string for how long this token is valid for. A value <= 0 or null means infinite time.
  ttl: 240h
```

```
kubectl apply -f downstream-manifests/cluster-registration-token.yml
```

The token value is the contents of a values.yaml file that is expected to be passed to helm install to install the Fleet agent on your downstream cluster. 

The token is stored in a Kubernetes secret referenced by the status.secretName field on the newly created ClusterRegistrationToken.
```
kubectl -n clusters get secret new-token -o 'jsonpath={.data.values}' | base64 --decode > values.yaml
```

### Deploy Fleet Agent to Downstream Cluster
Change your kube context to the cluster where the agent will be deployed.
```
helm -n fleet-system install --create-namespace --wait \
    --values values.yaml \
    fleet-agent https://github.com/rancher/fleet/releases/download/v0.3.3/fleet-agent-0.3.3.tgz
```

The agent should now be deployed. You can check that status of the fleet pods by running the below commands.
```
kubectl -n fleet-system logs -l app=fleet-agent
kubectl -n fleet-system get pods -l app=fleet-agent
```

Change back to the kube context for your Fleet manager and run the following command:
```
kubectl -n clusters get clusters.fleet.cattle.io
```

## Mapping to Downstream Cluster
When deploying GitRepos to downstream clusters the clusters must be mapped to a target as shown below in the code block. Make sure the GitRepo objects are deployed to the same namespace used by the *ClusterRegistrationToken*.

```
kind: GitRepo
apiVersion: fleet.cattle.io/v1alpha1
metadata:
  name: setting-up-eks-cluster-dojo
  namespace: clusters
spec:
  repo: https://github.com/LukeMwila/setting-up-eks-cluster-dojo
  paths:
  - manifests
  targets:
  - name: cluster-name
    clusterSelector:
      matchLabels:
        fleet.cattle.io/cluster: cluster-name
```

Save this YAML in a manifest file and deploy it to your Fleet manager cluster. 
```
kubectl apply -f <git-repo-manifest>
```

Lastly, you can review your GitRepo deployment to check the status of Fleet cloning the repo and deploying the resources to the downstream cluster. Make sure the repo being cloned matches the [expected structure](http://fleet.rancher.io/gitrepo-structure/) for Fleet.