# VKS Deployment
## Requirements
### CLI Tools
* kubectl cli v1.34: https://dl.k8s.io/release/v1.34.0/bin/linux/amd64/kubectl
* vcf cli v9.0.1: https://packages.broadcom.com/artifactory/vcf-distro/vcf-cli/linux/amd64/v9.0.1/
* helm cli v3.19: https://get.helm.sh/helm-v3.19.0-linux-amd64.tar.gz

### Get files
```shell
git clone https://github.com/cleeistaken/bmc-k8s-testing.git
```

### Required vSphere Steps
1. In WCP, create a supervisor namespace **bmc**.
2. Add '**vsan-esa-default-policy-raid5**' storage policy to **bmc** supervisor namespace
3. Add VM classes to supervisor namespace 
   * Create and add a VM class named '**vks-8-32-class**' and add it to supervisor namespace
     * Define the CPU and RAM. This class will be used for the VKS worker nodes (e.g. 8 vCPU and 32GB RAM)
   * Add '**best-effort-medium**' VM class to supervisor namespace

**Note**: These steps required access to vCenter and typically done by the infrastructure administrator.

## Deployment Procedure

### 1. Set environment variables
To facilitate the creation of multiple deployments in different environments we create and use variables throughout this sample deployment procedure.
```shell
SUPERVISOR_IP="10.138.169.5"
SUPERVISOR_USERNAME="<username>"  # Update with correct value
SUPERVISOR_NAMESPACE_NAME="bmc"
SUPERVISOR_CONTEXT="bmc-ctx"
CLUSTER_NAME="bmc-vks"
CLUSTER_NAMESPACE_NAME="bmc-vks-ns"
```


### 2. Clean kubectl and vcf configs
Note: This step is not required but helps avoid issues related to stale contexts or collisions between environments using the same context names.
```shell
rm ~/.kube/config
rm -rf ~/.config/vcf/
```

### 3. Create context on supervisor
```shell
# Create a context named 'bmc-ctx'
vcf context create $SUPERVISOR_CONTEXT --endpoint $SUPERVISOR_IP --insecure-skip-tls-verify -u $SUPERVISOR_USERNAME 

# Expected output:
# [i] Some initialization of the CLI is required.
# [i] Let's set things up for you.  This will just take a few seconds.
# 
# [i] 
# [i] Initialization done!
# [i] ==
# [i] Auth type vSphere SSO detected. Proceeding for authentication...
# Provide Password: 
# 
# Logged in successfully.
#
# You have access to the following contexts:
#    bmc-ctx
#    bmc-ctx:bmc
#
# If the namespace context you wish to use is not in this list, you may need to
# refresh the context again, or contact your cluster administrator.
#
# To change context, use `vcf context use <context_name>`
# [ok] successfully created context: bmc-ctx
# [ok] successfully created context: bmc-ctx:bmc
```

### 4. Set supervisor context
```shell
# Set context
vcf context use $SUPERVISOR_CONTEXT:$SUPERVISOR_NAMESPACE_NAME

# Expected output
# [ok] Token is still active. Skipped the token refresh for context "bmc-ctx:bmc"
# [i] Successfully activated context 'bmc-ctx:bmc' (Type: kubernetes) 
# [i] Fetching recommended plugins for active context 'bmc-ctx:bmc'...
# [i] No image repository override information was found
# [ok] All recommended plugins are already installed and up-to-date. 
```

### 5. Create VKS cluster 
Note: The vks yaml file is set to use "cluster-vks" as the resource name. We use 'sed' to dynamically replace 'cluster-vks' with the vks cluster name set in the environment variables.
```shell
# Create a VKS cluster as defined in vks.yaml
sed "s/cluster-vks/$CLUSTER_NAME/g" vks.yaml  | kubectl apply -f -

# Expected output:
# cluster.cluster.x-k8s.io/bmc-vks created
# 
# or:
# cluster.cluster.x-k8s.io/bmc-vks configured
```

### 5. Wait for cluster creation
```shell
# Test commands
kubectl get cluster --watch

# Expected output:
# NAME         CLUSTERCLASS             AVAILABLE   CP DESIRED   CP AVAILABLE   CP UP-TO-DATE   W DESIRED   W AVAILABLE   W UP-TO-DATE   PHASE         AGE   VERSION
# bmc-vks   builtin-generic-v3.5.0   False       1            0              1               4           0             4              Provisioned   67s   v1.34.1+vmware.1
# [...]
# bmc-vks   builtin-generic-v3.5.0   False       1            1              1               4           3             4              Provisioned   3m18s   v1.34.1+vmware.1
# bmc-vks   builtin-generic-v3.5.0   True        1            1              1               4           3             4              Provisioned   3m18s   v1.34.1+vmware.1
#
# Note: wait until output shows "Available: True" 
```

### 6. Connect to VKS cluster
```shell
# Connect to the VKS cluster
vcf context create vks \
--endpoint $SUPERVISOR_IP \
-u $SUPERVISOR_USERNAME \
--workload-cluster-namespace=$SUPERVISOR_NAMESPACE_NAME \
--workload-cluster-name=$CLUSTER_NAME \
--insecure-skip-tls-verify

# Expected output:
# [i] Logging in to Kubernetes cluster (bmc-vks) (bmc)
# [i] Successfully logged in to Kubernetes cluster 10.138.216.200
#
# You have access to the following contexts:
#    vks
#    vks:bmc-vks
#
# If the namespace context you wish to use is not in this list, you may need to
# refresh the context again, or contact your cluster administrator.
# 
# To change context, use `vcf context use <context_name>`
# [ok] successfully created context: vks
# [ok] successfully created context: vks:bmc-vks
```

### 7. Use VKS cluster context
```shell

# Use context
vcf context use vks:$CLUSTER_NAME

# Expected output: 
# [ok] Token is still active. Skipped the token refresh for context "vks:bmc-vks"
# [i] Successfully activated context 'vks:bmc-vks' (Type: kubernetes) 
# [i] Fetching recommended plugins for active context

# Test Commands

# Get Nodes
kubectl get nodes -o wide

# Expected output: 
# NAME                                       STATUS   ROLES           AGE   VERSION            INTERNAL-IP   EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION     CONTAINER-RUNTIME
# bmc-vks-node-pool-1-8jzgf-bhrz2-vmnw7   Ready    <none>          36m   v1.34.1+vmware.1   172.26.0.5    <none>        Ubuntu 24.04.3 LTS   6.8.0-86-generic   containerd://2.1.4+vmware.3-fips
# bmc-vks-node-pool-2-zd45h-59jlx-mrr64   Ready    <none>          36m   v1.34.1+vmware.1   172.26.0.6    <none>        Ubuntu 24.04.3 LTS   6.8.0-86-generic   containerd://2.1.4+vmware.3-fips
# bmc-vks-node-pool-3-v8j28-bjm5t-w2h2q   Ready    <none>          36m   v1.34.1+vmware.1   172.26.0.4    <none>        Ubuntu 24.04.3 LTS   6.8.0-86-generic   containerd://2.1.4+vmware.3-fips
# bmc-vks-node-pool-4-btqp5-z4ts9-sx9bd   Ready    <none>          36m   v1.34.1+vmware.1   172.26.0.7    <none>        Ubuntu 24.04.3 LTS   6.8.0-86-generic   containerd://2.1.4+vmware.3-fips
# bmc-vks-trdx9-gmtbm                     Ready    control-plane   38m   v1.34.1+vmware.1   172.26.0.3    <none>        Ubuntu 24.04.3 LTS   6.8.0-86-generic   containerd://2.1.4+vmware.3-fips
```


### 8. Create VKS namespace
```shell
# Create a namespace on the VKS cluster
kubectl create namespace $CLUSTER_NAMESPACE_NAME

# Expected output:
# namespace/bmc-ns created
#
# or:
# Error from server (AlreadyExists): namespaces "bmc-ns" already exists

# Set context
kubectl config set-context --current --namespace=$CLUSTER_NAMESPACE_NAME

# Expected output:
# Context "vks:bmc-vks" modified.

# Test commands
kubectl get all
```

### 9. (Optional) Create Secret with Docker.io Credentials
May be required if the deployment hits errors about the site hitting image pull limits.
```shell
# Create secret with Docker login credentials in Kubernetes
kubectl create secret docker-registry regcred \
  --docker-server=docker.io \
  --docker-username=<docker_username> \
  --docker-password=<docker_password> \
  --docker-email=<docker_email> 
  --namespace=$CLUSTER_NAMESPACE_NAME

# Automatically use credentials for all pods in namespace 
kubectl patch serviceaccount default \
  -p '{"imagePullSecrets": [{"name": "regcred"}]}'
```


## Cleanup Procedure

```shell
#Delete the namespace
kubectl delete namespace $CLUSTER_NAMESPACE_NAME

# Switch to supervisor context 
vcf context use $SUPERVISOR_CONTEXT:$SUPERVISOR_NAMESPACE_NAME

# Delete VKS cluster as defined in vks.yaml
kubectl delete -f vks.yaml
```


## Troubleshooting

### Useful Commands
```shell
# Refresh contexts
vcf context refresh --insecure-skip-tls-verify $SUPERVISOR_CONTEXT:$SUPERVISOR_NAMESPACE_NAME 
vcf context refresh --insecure-skip-tls-verify vks:$CLUSTER_NAME

# List all
kubectl get all

# Get detailed pod information
kubectl get pods -o wide

# Get container logs
kubectl logs -f <container name>

# Get all services
kubectl get svc

# Expose a service
kubectl expose service <service_name>  --type=LoadBalancer --name=<service_name>-external

# Get the external IP
kubectl get svc <service_name>-external

# Get TKR releases
kubectl get tkr -l '!kubernetes.vmware.com/kubernetesrelease'

# Get TKE releaase specs
# e.g. kubectl get tkr 'v1.34.1---vmware.1-vkr.4' -o yaml
kubectl get tkr TKR_NAME -o yaml  

# Get all cluster classes
kubectl -n bmc get clusterclass -A

kubectl get secret 
kubectl get secret bmc-vks-no-cni-1-33-cc-ssh-password -o yaml
echo "" | base64 -d 

kubectl get secret bmc-calico-vks-ssh-password -o json | jq -r '.data."ssh-passwordkey"' | base64 -d


kubectl patch packageinstall <pkgi-name> -n <namespace> --type='merge' -p '{"spec":{"paused":true}}'
```
