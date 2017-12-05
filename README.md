

# Installing Kubernetes on vSphere 6.x

The document below describes the steps needed to get a Kubernetes cluster up and running on vSphere 6.x.

**Note:** This install guide is intended for a Lab/Dev environment installation, I would **NOT** recommend using this for production use.


I have broken up this deployment into a number of distinct parts, but in a nutshell, the steps can be summarized as follows:

Download “kubernetes-anywhere” container
Make configuration file for deploying Kubernetes on vSphere
Deploy Kubernetes on vSphere
Get access to the Kubernetes dashboard

### Installation Preparation

Setting up the install workstation/machine from where deployment will be done is a simple affair. The deployment machine in this guide is running OSX.

**Note:** Alternatively you could use Linux, other then how to install "Docker and kubectl" - the rest of the guide would apply as normal.

First we need to make sure that Docker is installed. Please follow the guide on docs.docker.com

[Guide to installing Docker on OSX](https://docs.docker.com/docker-for-mac/)

Next we need to install Kubernetes CLI interface kubectl. This will be needed to communicate with the Kubernetes cluster once installed.

on OSX this can be done easily via Homebrew

```
brew install kubectl
```

Next we need to install Kubernetes-Anywhere container. This will configure and deployment the Kubernetes cluster for us.

```
docker pull cnastorage/kubernetes-anywhere:latest
```

Now that the workstation is prepped, we can upload the Kubernetes template OS OVA image to vSphere. This image will be used by all the Kubernetes nodes.

You can upload the image using the vSphere Web Client. The steps are as follows:

1. Login to vSphere Client
2. Right-Click on the ESXi cluster where you would like to deploy the template
3. Select Deploy OVF template
4. Copy and paste the URL for the [Kubernetes OVA - vSphere 6.x](https://storage.googleapis.com/kubernetes-anywhere-for-vsphere-cna-storage/KubernetesAnywhereTemplatePhotonOS.ova)
5. Check the name of the VM created is **KubernetesAnywhereTemplatePhotonOS.ova**. This VM image will be used by the installer to create the cluster.
  * Once the VM instances are created (Master, Nodes) the default root password will be "kubernetes"

**Note:** Do **NOT** Power up the imported VM.

### Installation

Now we can launch the container on the deployment workstation and enter the container to configure and start the installation.

```
docker run -it --rm --env="PS1=[container]:\w> " --net=host cnastorage/kubernetes-anywhere:latest /bin/bash
```

Once inside the container, we need to create the configuration for the Kubernetes-Anywhere installer to use. Execute the following command

```
make config
```

You will now be asked a list of questions. I have included the questions below with example answers

**Note:** The default answer is in the square brackers []. If you are happy with this, just press return. Also in regards to the sizing (CPU, Memory and number of servers), this something you will need to adjust for your LAB/Dev environment

```
number of nodes (phase1.num_nodes) [4] (NEW)
kubernetes cluster name (phase1.cluster_name) [kubernetes] (NEW)
SSH user to login to OS for provisioning (phase1.ssh_user) [] (NEW)
cloud provider: gce, azure or vsphere (phase1.cloud_provider) [vsphere] (NEW)
vCenter URL Ex: 10.192.10.30 or myvcenter.io (without https://) (phase1.vSphere.url) [] (NEW) 192.168.10.253
vCenter port (phase1.vSphere.port) [443] (NEW)
vCenter username (phase1.vSphere.username) [] (NEW) administrator@lab.unixcraft.lcl
vCenter password (phase1.vSphere.password) [] (NEW) !MyP4ssW0rd!
Does host use self-signed cert (phase1.vSphere.insecure) [Y/n/?] (NEW)
Datacenter (phase1.vSphere.datacenter) [datacenter] (NEW) UnixCraft
Datastore (phase1.vSphere.datastore) [datastore] (NEW) VMW_SSD_NFS01
Deploy Kubernetes Cluster on 'host' or 'cluster' (phase1.vSphere.placement) [cluster] (NEW)
vsphere cluster name. Please make sure that all the hosts in the cluster are time-synchronized otherwise some of the nodes can remain in pending state for ever due to expired certificate (phase1.vSphere.cluster) [] (NEW) LAB
Do you want to use the existing resource pool on the host or cluster? [yes, no] (phase1.vSphere.useresourcepool) [no] (NEW)
VM Folder name or Path (e.g kubernetes, VMFolder1/dev-cluster, VMFolder1/Test Group1/test-cluster). Folder path will be created if not present (phase1.vSphere.vmfolderpath) [kubernetes] (NEW)
Number of vCPUs for each VM (phase1.vSphere.vcpu) [1] (NEW) 2
Memory for VM (phase1.vSphere.memory) [2048] (NEW) 8192
Network for VM (phase1.vSphere.network) [VM Network] (NEW)
Name of the template VM imported from OVA. If Template file is not available at the destination location specify vm path (phase1.vSphere.template) [KubernetesAnywhereTemplatePhotonOS.ova] (NEW)
Flannel Network (phase1.vSphere.flannel_net) [172.1.0.0/16] (NEW)

*
* Phase 2: Node Bootstrapping
*
kubernetes version (phase2.kubernetes_version) [v1.6.5] (NEW)
bootstrap provider (phase2.provider) [ignition] (NEW)
installer container (phase2.installer_container) [docker.io/cnastorage/k8s-ignition:v2] (NEW)
docker registry (phase2.docker_registry) [gcr.io/google-containers] (NEW)
*
* Phase 3: Deploying Addons
*
Run the addon manager? (phase3.run_addons) [Y/n/?] (NEW)
Run kube-proxy? (phase3.kube_proxy) [Y/n/?] (NEW)
Run the dashboard? (phase3.dashboard) [Y/n/?] (NEW)
Run heapster? (phase3.heapster) [Y/n/?] (NEW)
Run kube-dns? (phase3.kube_dns) [Y/n/?] (NEW)
Run weave-net? (phase3.weave_net) [N/y/?] (NEW)
#
# configuration written to .config
#
```

Once the configuration is created, you can now kick of the installation. Execute the following command.

```
make deploy
```

Once the installation is completed, we just need to get the save the Kubernetes configuration file, this will be used to provide the configuration details required by Kubectl.

```
cd /opt/kubernetes-anywhere
cat phase1/vsphere/kubernetes/kubeconfig.json
```

Copy the output of kubeconfig.json and store it in a file on your deployment workstation.

You can now exit the container.

On the Deployment workstation you need to export a environment variable and set the value to the configuration file you just created.

```
export KUBECONFIG=~/kubeconfig.json
```

Your installation is now complete. You just need to get the Kubernetes Dashboard url.

Execute the following commands. Once executed, the Kubernetes Dashboard address and port (URL) will be provided.

```
kube_dashboard_pod_name=$(kubectl get pods --namespace=kube-system| grep -i dashboard | awk '{print $1}')
kube_dash_ip=$(kubectl describe pod $kube_dashboard_pod_name --namespace=kube-system| grep Node: | awk -F "/" '{ print $2 }')
kube_dash_port=$(kubectl describe service kubernetes-dashboard --namespace=kube-system| grep -i NodePort: | awk '{print $3}' | awk -F "/" '{print $1}')
echo
echo "Dashboard Address: http://$kube_dash_ip:$kube_dash_port"
```

You can now visit the Kubernetes Dashboard URL
