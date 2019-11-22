# AWS-to-minikube

# Setting up minikube Kubernetes, pulling images from a private repository and deploying pods, and using initContainers and volumeMounts.

### Setting up minikube

1. Initially, make sure docker is available, if not get it installed from https://docs.docker.com/docker-for-mac/install/
**Installing minikube**
2. To check if virtualization is supported on macOS, run the following command on your terminal.
```bash
sysctl -a | grep -E --color 'machdep.cpu.features|VMX' 
```
If you see VMX in the output (should be colored), the VT-x feature is enabled in your machine.
3. Install kubectl with Homebrew on macOS
```bash
brew install kubectl
kubectl version
```
4. Link kubernetes cli
```bash
brew link kubernetes-cli
```
5. Linking fails with macOS Catalina, in that case overwrite it.
```bash
brew link --overwrite kubernetes-cli
```
6. Install xcode-select. xcode-select is a command-line utility on OS X that facilitates switching between different sets of command line developer tools provided by Apple.
```bash
xcode-select --install (BEFORE MINIKUBE)
```
7. Install minikube with Homebrew
 ```bash
brew install minikube
```
8. minikube dashboard can be simply used by executing the following command
```bash
minikube dashboard
```

### Pulling images from private repositories (AWS, GCP, DockerHub)

1. Enable ingress
```bash
minikube addons enable ingress
```
2. Configure registry-creds
Upon executing the below command, you'll be asked to choose a repository. Say 'yes' to the relevant one and 'no' to the repositories from which you won't be pulling any image. The four important values are access key id, access key, account number and the location.
```bash
minikube addons configure registry-creds
```
3. Enable registry creds once the same has been configured.
```bash
minikube addons enable registry-creds
```

### Optional: setting up credential helper in-case registry-creds doesn't work
```bash
brew install docker-credential-helper-ecr
~/.docker/config.json
```
```json
{
	"credHelpers": {
		"aws_account_id.dkr.ecr.region.amazonaws.com": "ecr-login"
	}
}
```
Define the credentials here
```bash
nano ~/.aws/credentials
```

### Designing deployment files and creating (deploying) them.

##### Different yaml 'kind' used in this project
- Service: This is the networking part where you define port, targetPort (optional), protocol, ip address (optional) along with metadata and labels (optional).
- Pod, and Deployment: Both Pod and Deployment are full-fledged objects in the Kubernetes API. Deployment manages creating Pods by means of ReplicaSets. What it boils down to is that Deployment will create Pods with spec taken from the template. It is rather unlikely that you will ever need to create Pods directly for a production use-case. In this project deployment kind is used.

Both, service and deployment yaml specification can be deployed in a single file by keeping both of them separate with a line made up of three hyphens (- - -)
```yaml
apiVersion: v1
kind: Service
metadata:
  labels:
spec:
  selector:
---    
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  labels:
```
**1. Design of the Service kind**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: drt-darp-solver-frontend # NAME OF THE SERVICE
  labels:
    k8s-app: drt-darp-solver-frontend # IDENTITY OF THE SERVICE
spec:
  type: NodePort
  ports:
  - port: 31000 # INTERNAL PORT NUMBER  
    targetPort: 5000 # PORT AT WHICH THE SERVICE IS EXPOSED
    protocol: TCP # PROTOCOL
    name: http
  selector:
    k8s-app: drt-darp-solver-frontend
```
**2. Design of the Deployment kind**
```yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: drt-darp-solver-frontend
  namespace: drt
  labels:
    k8s-app: drt-darp-solver-frontend
spec:
  replicas: 1
  selector:
    matchLabels:
      k8s-app: drt-darp-solver-frontend
  template:
    metadata:
      name: drt-darp-solver-frontend    
      labels:
        k8s-app: drt-darp-solver-frontend
    spec:
      containers:
       - name: drt-darp-solver-frontend
         image: 312801333485.dkr.ecr.eu-central-1.amazonaws.com/rideshare/drt-darp-solver-frontend # IMAGE FROM AWS ECR PRIVATE REPOSITORY
         env: # ENVIRONMENT VARIABLES
         - name: DRT_DARP_SOLVER_URL
           value: http://drt-darp-solver.drt:31001/optimize # POINTS TO THE drt-darp-solver POD
         - name: D2D_DIRECTIONS_DRIVING_SERVICE_URL
           value: http://osrm-routing-core.drt:31002/route # POINTS TO osrm-routing-core POD
         - name: MAPBOX_ACCESS_TOKEN
           value: pk.eyJ1IjoiYWxscnlkZXIiLCJhIjoidWs5cUFfRSJ9.t8kxvO3nIhCaAl07-4lkNw  
         securityContext:
          privileged: false
          procMount: Default 
      imagePullSecrets: # THIS WILL PULL THE registry-creds WHICH WERE CREATED BEFORE
       - name: awsecr-cred
    strategy: # SPECIFIES THE STRATEGY USED TO REPLACE OLD PODS BY NEW ONES
    type: RollingUpdate
    rollingUpdate: 
      maxUnavailable: 25% #  MAX NO. OF PODS UNAVAILABLE DURING THE UPDATE PROCESS
      maxSurge: 25% # SPECIFIES THE MAX NO. OF PODS THAT CAN BE CREATED OVER THE DESIRED NO. OF PODS  
  status:
    conditions:
    - type: Available
      status: 'True'
    - type: Progressing
      status: 'True'  
```

###### For this project, the below topology has 4 images out of which, drt-darp-solver-frontend (1st) and drt-darp-solver (2nd) are pulled and deployed separately while there is a common deployment (3rd) for osrm-routing-core and osrm-data-graphs. In the third deployment, we keep osrm-routing-core as a main container and osrm-data-graphs as an initContainer. The reason is, we just need to have the graphs from /usr/src/graphs/graph_2019_11_01 directory from osrm-data-graphs container copied to in one of the directories of osrm-routing-core so that osrm-routing-core can locally access the driving.osrm file required to show graphs on the map which is done by using an environment variable pointing to the local directory where the files from /graph_2019_11_01 directory exist.
The process here for replicating file(s) or directory from one container to another is done using command tool in a YAML file and the most importantly, using volumeMounts. Using initContainer is optional whereas in this project it is required.

312801333485.dkr.ecr.eu-central-1.amazonaws.com/rideshare/drt-darp-solver-frontend

312801333485.dkr.ecr.eu-central-1.amazonaws.com/rideshare/drt-darp-solver

312801333485.dkr.ecr.eu-central-1.amazonaws.com/rideshare/osrm-routing-core

312801333485.dkr.ecr.eu-central-1.amazonaws.com/rideshare/osrm-data-graphs

**3. Deployment with initContainer and volumeMounts.**
```yaml
apiVersion: extensions/v1beta1
kind: Deployment
# CONTENT FOR METADATA REMOVED FOR BREVITY. REFER TO THE YAML FILES IN THE REPO FOR DETAIL SPECS.
    spec:        
      containers: # Refer '@1'
      - image: 312801333485.dkr.ecr.eu-central-1.amazonaws.com/rideshare/osrm-routing-core
        imagePullPolicy: IfNotPresent
        name: osrm-routing-core
        env:
        - name: NODE_ENV
          value: development
        - name: OSRM_GRAPH
          value: /usr/src/app/data/driving/driving.osrm # Refer '@2'
        - name: STAGE
          value: development    

        volumeMounts: # Refer '@3'
        - mountPath: /usr/src/app/data/driving 
          name: demo-volume

      initContainers: # Refer '@4'
      - image: 312801333485.dkr.ecr.eu-central-1.amazonaws.com/rideshare/osrm-data-graphs
        imagePullPolicy: IfNotPresent
        name: osrm-data-graphs
        command: ["/bin/sh"] # Refer '@5'
        args: ["-c", "cp -r /usr/src/graphs/driving/graph_2019_11_01/* /usr/src/graphs/OSRM; sleep 10"]

        volumeMounts: # Refer '@6' 
        - mountPath: /usr/src/graphs/OSRM
          name: demo-volume
        
      volumes: # Refer '@7'
      - name: demo-volume
        emptyDir: {} 
      imagePullSecrets:
      - name: awsecr-cred  
```
#### Explanation:
**@1** is the main container which will run as usual. **@2** mentions the path where it will fetch the driving.osrm to display the graphs on the map, but initially this path doesn't exist. Soon you'll get know the setup.
Now, what's **@4** (initContainers)? initContainers always initiate before main container(s) **@1**, they always run to completion, and each init container must complete successfully before the next one starts.
If initContainers fail to complete then main containers would never initiate.
The reason for **@4** is, we only require osrm-data-graphs container for the graph files (to be copied to the main container's directory), but not to run it as a server, and that's the reason it is supposed to run only for a few seconds

What are volumeMounts (**@3** and **@6**) and volumes (**@7**)? The paths mentioned in volumeMounts could be already existing or non-existing, for this project they're originally not available. When the deployment is deployed, these directories will be automatically created. The reason they need to be mounted is that they can share resources with each other, in short, changes in either of the container's mounted volumes will be replicated onto the other container(s) mounted path. The replication is not a direct copy process, but rather it happens via 'volumes' **@7**. In our case, it's an emptyDir, but you can also use a local static path or a remote storage as a volume (refer: https://kubernetes.io/docs/concepts/storage/volumes/). 'volumes' is basically an intermidiate storage between the volumeMounts. **Note: Replication happens only for the data created post pods' creation, existing data is not replicated.**
**@5** is a command which copies all the files and directories inside /graph_2019_11_01 to it's another directory locally /OSRM (mounted path), and then the container waits for 10 seconds (sleep 10) before it terminates. Without using 'sleep', the container may not have enough time to copy large chunks of data before it terminates. 
Curiosity: as the container terminates even before the main container initiates, how would the data from **@4** be replicated into **@1**? Answer: the moment all the data gets copied to /OSRM directory, it is also stored in the 'emptyDir: {}' volume, and when the main container is deployed and the path mentioned in the mountPath is created, the data from the 'emptyDir: {}' volume is thus replicated into the main container's mounted path.

#### Stepwise: 
**@4** initContainer initiates >> **@6** mountPath is created and mounted >> **@5** files are copied from /graph_2019_11_01 to /OSRM and the same time from /OSRM to emptyDir (**@7**) >> initContainer waits 10 seconds before terminating >> **@1** starts-up >> **@3** Volume is mounted and data from emptyDir gets copied >> Now, **@1** has data and the path for the environment variable OSRM_GRAPH.


#### Useful commands - 
1. **minikube start**: to start your minikube environment
2. **minikube dashboard**: to have a GUI for all the Kubernetes components
3. **minikube stop**: to stop the minikube environment
4. **kubectl create -f NAME.yaml**: to deploy a service
5. **kubectl events -n NAMESPACE -w**: to watch the events in a particular namespace after deploying a service
6. **kubectl get pods -n NAMESPACE**: to view the pods in a particular namaspace
7. **kubectl logs PODNAME -n NAMESPACE**: to view the logs
8. **minikube service SERVICENAME -n NAMESPACE --url**: to get the url of an exposed service
9. **kubectl exec PODNAME -n NAMESPACE -i -t -- /bin/sh**: to get into pod directories
10. **kubectl --namespace NAMESPACED port-forward PODNAME 3000**: port-forwarding

