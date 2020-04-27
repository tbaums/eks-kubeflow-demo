# Kommander with EKS + Kubeflow demo

Built for VZ 5G MEC team.

Tested with:
- Konvoy v1.5.0-beta.0 deployed to AWS
- Kommander `configVersion: stable-1.16-1.1.0-beta.1`
- eksctl v0.17.0 (NB: eksctl does NOT have a `--version` flag. Check your version with `brew info eksctl`.)
- KUDO version 0.12.0 (Run `kubectl kudo --version` to check.)
- KUDO Kubeflow version 

## Prereqs

1. `eksctl` installed on your machine. (On MacOS: `brew install eksctl`)
1. `maws` authentication configured
1. Konvoy v1.5.0-beta.0 or above
1. Github SSH keys configured

## Install

- Clone this repo 
    - `git clone git@github.com:tbaums/eks-kubeflow-demo.git`
- CD into `eks-kubeflow-demo` directory
    - `cd eks-kubeflow-demo`
- Download Konvoy binaries or move existing binaries into local directory
- Modify the cluster.yaml to include your name and desired expiration tags
- Refresh AWS credentials 
    - `eval $(maws li 110465657741_Mesosphere-PowerUser)`
- Boot Konvoy cluster
    - `./konvoy up -y`
- Add Konvoy credentials to your local kubeconfig file
    - `./konvoy apply kubeconfig --force-overwrite`
- Confirm kubeconfig allows access to Konvoy cluster
    - `kubectl get no`
- Log in to your Kommander dashboard and add Kommander license (to allow for attachment of multiple EKS clusters)
- Create a new workspace named `USA-MEC`
- Create a new project named `mec-app-1`
- Set environment variables for your name and your desired expiration time
    - `export MY_NAME=<your name>`
    - `export EKS_EXPIRATION=<time, ex. "12h">`
- Run the command below to spin up EKS cluster #1: 
```
cat <<EOF | eksctl create cluster -f -
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig 
metadata:
    name: ${MY_NAME}-1
    region: us-east-1
    version: "1.15"
nodeGroups:
  - name: workers
    instanceType: m5.2xlarge
    desiredCapacity: 5
    volumeSize: 30
    labels: { owner: ${MY_NAME}, expiration: ${EKS_EXPIRATION} }
EOF
```
- Run the command below to spin up EKS cluster #2: 
```
cat <<EOF | eksctl create cluster -f -
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig 
metadata:
    name: ${MY_NAME}-2
    region: us-east-1
    version: "1.15"
nodeGroups:
  - name: workers
    instanceType: m5.2xlarge
    desiredCapacity: 5
    volumeSize: 30
    labels: { owner: ${MY_NAME}, expiration: ${EKS_EXPIRATION} }
EOF
```
- Run the command below to spin up EKS cluster #3: 
```
cat <<EOF | eksctl create cluster -f -
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig 
metadata:
    name: ${MY_NAME}-3
    region: us-east-1
    version: "1.15"
nodeGroups:
  - name: workers
    instanceType: m5.2xlarge
    desiredCapacity: 5
    volumeSize: 30
    labels: { owner: ${MY_NAME}, expiration: ${EKS_EXPIRATION} }
EOF
```
- Next, ensure you are connected to your first EKS cluster:
    - `kubectl config get-contexts`
    - `kubectl config use-context <context for first eks cluster>`
- Confirm your kubectl can access the EKS cluster  
    - `kubectl get no`
- Create a service account for Kommander on your EKS cluster
    - `kubectl -n kube-system create serviceaccount kommander-cluster-admin`
- Next, give your kommander-cluster-admin service account `cluster-admin` permissions
```
cat << EOF | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: kommander-cluster-admin
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: kommander-cluster-admin
  namespace: kube-system
EOF
```
- Next, set the following environment variables which will be used to create a kubeconfig file that is compatible with the Kommander UI.
```
export USER_TOKEN_NAME=$(kubectl -n kube-system get serviceaccount kommander-cluster-admin -o=jsonpath='{.secrets[0].name}')
export USER_TOKEN_VALUE=$(kubectl -n kube-system get secret/${USER_TOKEN_NAME} -o=go-template='{{.data.token}}' | base64 --decode)
export CURRENT_CONTEXT=$(kubectl config current-context)
export CURRENT_CLUSTER=$(kubectl config view --raw -o=go-template='{{range .contexts}}{{if eq .name "'''${CURRENT_CONTEXT}'''"}}{{ index .context "cluster" }}{{end}}{{end}}')
export CLUSTER_CA=$(kubectl config view --raw -o=go-template='{{range .clusters}}{{if eq .name "'''${CURRENT_CLUSTER}'''"}}{{ index .cluster "certificate-authority-data" }}{{end}}{{ end }}')
export CLUSTER_SERVER=$(kubectl config view --raw -o=go-template='{{range .clusters}}{{if eq .name "'''${CURRENT_CLUSTER}'''"}}{{ .cluster.server }}{{end}}{{ end }}')
```
- Confirm these variables have been set correctly
    - `env | grep CLUSTER`
- Now we are ready to create a new kubeconfig file that will be used in the Kommander UI. Run
```
cat << EOF > kommander-cluster-admin-config
apiVersion: v1
kind: Config
current-context: ${CURRENT_CONTEXT}
contexts:
- name: ${CURRENT_CONTEXT}
  context:
    cluster: ${CURRENT_CONTEXT}
    user: kommander-cluster-admin
    namespace: kube-system
clusters:
- name: ${CURRENT_CONTEXT}
  cluster:
    certificate-authority-data: ${CLUSTER_CA}
    server: ${CLUSTER_SERVER}
users:
- name: kommander-cluster-admin
  user:
    token: ${USER_TOKEN_VALUE}
EOF
```
- Verify that this kubeconfig file gives you access to the EKS cluster
    - `kubectl --kubeconfig $(pwd)/kommander-cluster-admin-config get all --all-namespaces`
- Copy `kommander-cluster-admin-config` to your clipboard
    - `cat kommander-cluster-admin-config | pbcopy`
- When adding to Kommander UI, change name to `eks-us-east-1-dev-1`. Add label `project: mec-app-1` and `env: dev`.
- Repeat for EKS clusters 2 and 3
- Edit the project to associate the 3 clusters with the project
- Deploy Zookeeper, Kafka, and Spark from catalog
    - Reserve Cassandra for live deployment during demo
- Create example secrets and configmaps to show federation
- Edit `custom-repo.yaml` to point to your project namespace
- Add custom app to the catalog to show extensibility
    - `kubectl apply -f custom-repo.yaml`


## Clean up
- MAKE SURE YOU DETACH YOUR EKS CLUSTERS IN THE KOMMANDER UI BEFORE DELETING
- Delete eks clusters
    - `eksctl delete cluster --name ${MY_NAME}-<number of cluster to delete>  --region us-east-1`


