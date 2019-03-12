# kubernetes-jenkins
Integrate Jenkins with PKS provisioned Kubernetes Cluster

## Prepare the Kubernetes Cluster and Chart Values

### Clone the repo to a local directory
`$ git clone https://github.com/csaroka/kubernetes-jenkins.git` \
`$ cd kubernetes-jenkins`

### Create the project Namespace
`$ kubectl create ns jenkins` \
`$ kubectl config set-context $(kubectl config current-context) --namespace=jenkins`

### Prepare Persistence Storage
#### Create a storage class
`$ cat jenkins-sc.yaml`
```
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: jenkins-disk
provisioner: kubernetes.io/vsphere-volume
parameters:
    diskformat: thin
```
`$ kubectl apply -f jenkins-sc.yaml` \
`$ kubectl get sc`

#### Create a persistent volume claim:
`$ cat jenkins-pvc.yaml`
```
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: jenkins-data
  annotations:
    volume.beta.kubernetes.io/storage-class: jenkins-disk
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 20Gi
```
`$ kubectl apply -f jenkins-pvc.yaml` \
`$ kubectl get pvc`

### Build a Custom Jenkins Slave Image for using the Kubernetes/Helm CLI plugin
The default jenkins/jnlp-slave image does not contain the kubectl or helm binaries, so you will need to build a custom image to use the kubernetes-cli plugin. Afterwards, push it to a registry\
`$ cd jnlp-slave-image` \
`$ docker build -t jenkins-slave-k8s .` \
`$ docker tag jenkins-slave-k8s <Private Registry FQDN>/<Project>/jenkins-slave-k8s:v1`\
`$ docker push <Private Registry FQDN>/<Project>/jenkins-slave-k8s:v1` \
`$ cd ..` 

Alternatively, you could pull from \
`$ docker pull csaroka/jenkins-slave-k8s:lts`

### (Optional) Pull the Jenkins Master Image and Push to a Private Registry. If pulling direct from public registry, skip this step.
`$ docker pull jenkins/jenkins:lts` \
`$ docker tag jenkins/jenkins <Private Registry FQDN>/<Project>/jenkins-master:v1` \
`$ docker push <Private Registry FQDN>/<Project>/jenkins-master:v1` 

### Install Helm Client and Tiller Server

Follow instructions available here for installing the Helm client:  
https://docs.helm.sh/using_helm/#installing-helm 

Apply the Tiller Service Account and RBAC policy

`$ cat tiller-rbac.yaml`
```
apiVersion: v1
kind: ServiceAccount
metadata:
  name: tiller
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: tiller
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
  - kind: ServiceAccount
    name: tiller
    namespace: kube-system
```

`$ kubectl apply -f tiller-rbac.yaml`

Deploy Tiller to the Kubernetes Cluster \
`$ helm init --service-account tiller`

### Prepare the Jenkins Helm Chart values.yaml

`$ cd jenkins` \
Open `values.yaml` with a text editor

#### Identify Jenkins Container Images' Location

Replace the *Image* and *ImageTag* values with the appropriate private registy paths. The defualt public images are commented out.
>If intending to use the Kubernetes CLI Plugin, you will need to follow the steps above to build a custom slave image with the kubectl and helm binaries, then push it to a registry. 
```
Master:
  Image: "harbor.lab.local/jenkins/jenkins-master"
  ImageTag: "v1"
# Image: "jenkins/jenkins"
# ImageTag: "lts"
Agent:
  Image: "harbor.lab.local/jenkins/jenkins-slave-k8s"
  ImageTag: "v1"
# Image: jenkins/jnlp-slave
# ImageTag: 3.10-1
```
If behind a proxy, uncomment and update the paths
```
# InitContainerEnv:
#   - name: http_proxy
#     value: "http://192.168.64.1:3128"
# ContainerEnv:
#   - name: http_proxy
#     value: "http://192.168.64.1:3128"
```
#### Change the Admin Password
```
AdminPassword: 'VMware1!'
```
or comment out the parameter to randomly generate a password for you
```
# AdminPassword: 'VMware1!'
```
#### Configure the Ingress Resource

##### Set the Ingress Hostname
>Note: The hostname is customizable but the domain needs to match cluster's ingress controller's wildcard record in DNS. For example, *.pksk8s01apps.lab.local
```
HostName: jenkins.k8s01apps.lab.local
```
##### (Optional) Set the path
```
JenkinsUriPrefix: "/jenkins"
```

#### (Optional) Use LoadBalancer, as opposed to a Ingress Resource
Change the ServiceType from:
```
ServiceType: ClusterIP
```
to 
```
ServiceType: LoadBalancer
```

Comment out Ingress Hostname and URI Prefix: 
```
# HostName: jenkins.k8s01apps.lab.local
# JenkinsUriPrefix: "/jenkins"
```
#### Configure Persistence

Verify claim name matches PVC created above
```
Persistence:
  ExistingClaim: "jenkins-data"
```
Verify class name matches Storage Class created above
```
Persistence:
  StorageClass: "jenkins-disk"
```

Save changes to `values.yaml` and close

## Install the Chart to the Kubernetes Cluster

`$ cd ..` \
or `cd` to the directory containing the Jenkins chart directory

### Use the Helm client to install the chart to the cluster
`$ helm install --name=jenkins ./jenkins`

```
NAME:   jenkins
LAST DEPLOYED: Thu Dec 27 18:22:32 2018
NAMESPACE: jenkins
STATUS: DEPLOYED

RESOURCES:
==> v1/ClusterRoleBinding
NAME                  AGE
jenkins-role-binding  0s

==> v1/Service
NAME           TYPE       CLUSTER-IP      EXTERNAL-IP  PORT(S)    AGE
jenkins-agent  ClusterIP  10.100.200.215  <none>       50000/TCP  0s
jenkins        ClusterIP  10.100.200.15   <none>       8080/TCP   0s

==> v1/Deployment
NAME     DESIRED  CURRENT  UP-TO-DATE  AVAILABLE  AGE
jenkins  1        1        1           0          0s

==> v1beta1/Ingress
NAME     HOSTS                           ADDRESS           PORTS  AGE
jenkins  jenkins.pksk8s01apps.lab.local  10.12.0.6,100...  80     0s

==> v1/Pod(related)
NAME                      READY  STATUS    RESTARTS  AGE
jenkins-7d48db75c5-2mpn9  0/1    Init:0/1  0         0s

==> v1/Secret
NAME     TYPE    DATA  AGE
jenkins  Opaque  2     0s

==> v1/ConfigMap
NAME           DATA  AGE
jenkins        5     0s
jenkins-tests  1     0s

==> v1/ServiceAccount
NAME     SECRETS  AGE
jenkins  1        0s


NOTES:
1. Get your 'admin' user password by running:
  printf $(kubectl get secret --namespace jenkins jenkins -o jsonpath="{.data.jenkins-admin-password}" | base64 --decode);echo

2. Visit http://jenkins.k8s01apps.lab.local

3. Login with the password from step 1 and the username: admin

For more information on running Jenkins on Kubernetes, visit:
https://cloud.google.com/solutions/jenkins-on-container-engine
Configure the Kubernetes plugin in Jenkins to use the following Service Account name jenkins using the following steps:
  Create a Jenkins credential of type Kubernetes service account with service account name jenkins
  Under configure Jenkins -- Update the credentials config in the cloud section to use the service account credential you created in the step above.
```
Run `$ watch kubectl get pods` to monitor the pod creation status or simply run `kubectl get pods` until the output reports the pod is "Running". Note: Transitioning from "Init" to "Running" may take 5-10 minutes.
```
NAME                       READY   STATUS    RESTARTS   AGE
jenkins-7d48db75c5-2mpn9   1/1     Running   0          5m
```
## Access the Jenkins Web UI for Initial Configuration

>The `NOTE No 2.` from the helm install output does not account for a URI Prefix, if set in the values.yaml. If you did not change the value.yaml option: `JenkinsUriPrefix: "/jenkins"`, the `NOTE No 2.` should actually indicate to: \
`Visit http://jenkins.k8s01apps.lab.local/jenkins`

Open a web browser to (parameters from values.yaml):
`http://<HostName>/<JenkinsUriPrefix>`

![alt text](https://github.com/csaroka/kubernetes-jenkins/blob/master/images/jenkins-welcome.png)

Login with credentials \
Username: **admin** \
Password: **VMware1!**

> Note: If you forgot the password, run the command `$ printf $(kubectl get secret --namespace jenkins jenkins -o jsonpath="{.data.jenkins-admin-password}" | base64 --decode);echo`

> If at any time while attempting to access the Jenkins Web UI, you receive a browser error, i.e. 502, clear the browser cache and restart it or either open an "New Incognito Window" (Google Chrome) or "New Private Window" (Firefox).

### Installing the Required Updates and Plugins

Select **Manage Jenkins**>**Manage Plugins**. Select the **Available** tab and enter "kubernetes" in the search filter. Choose the following plugin options:
	
- Kubernetes Continuous Deploy
- Kubernetes Cli
- Kubernetes Credentials Provider
- Kubernetes :: Pipeline :: DevOps Steps
- Kubernetes :: Pipeline :: Kubernetes Steps

Select **Download now and install after restart** \
On the following page, choose **Restart Jenkins when installation is complete and no jobs are running**

After a few minutes, if the status does not change, select **Return to Dashboard** and complete the login prompt. An ENABLE AUTO REFRESH link is avaible in the upper right of the window but the results aren't consistant across browsers. Again, if after waiting a few minutes for auto-refresh to update, replace to the current brower URL with the welcome page URL, `http://<HostName>/<JenkinsUriPrefix>`

> Before proceeding, consider returning to the Plugin Manager, searching for "docker" in the list of available plugins and repeating the previous process for installing plugins: "docker-build-step" and "Docker"

### Configuring the Kubernetes Credentials and Plugin

From the Jenkins Dashboard select **Credentials**>**System**>**Global credentials (unrestricted)**>**Add Credentials** \
Select the *Kind* drop-down and choose **Kubernetes Service Account**

![alt text](https://github.com/csaroka/kubernetes-jenkins/blob/master/images/kubernetes-service-account.png)

Select **OK**

Again, Select **Add Credentials** \
Select the *Kind* drop-down and choose **Kubernetes configuration (kubeconfig)**

Open a command-line and run, \
`$ kubectl get sa` \
`$ kubectl describe sa jenkins` \
Record the secret value from the *Tokens:* parameter
```
Tokens:              jenkins-token-6xx5l
```
`$ kubectl config view` \
Record the URL value from the *server* parameter
```
apiVersion: v1
clusters:
- cluster:
    server: https://pksk8s01api.lab.local:8443
```
With a vi or another text editor, edit the file *svc-acct-kubeconfig.sh*. \
For the *server* variable, enter the *server* value from the kubectl config view output. \
For the *name* variable, enter the *Tokens* value from the kubectl describe sa jenkins output. \
Save and close the file. Then, run the commands to execute the script and generate a skeleton kubeconfig for the service account\
`$ chmod +x svc-acct-kubeconfig.sh` \
`$ ./svc-acct-kubeconfig.sh` \
With a vi or another text editor, edit the new file sa.kubeconfig, and compare the contents to the output from `$ kubectl config view`. Replace the "default-cluster" and "default-context" with the values from the `$ kubectl config view` output. Replace the default (namespace) value with the name of the jenkins namespace.  Replace default-user value with the name of the service account. For example,

```
apiVersion: v1
kind: Config
clusters:
- name: k8s01staging
  cluster:
    certificate-authority-data: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSUMrekNDQWVPZ0F3SUJBZ0lVUDVpUHd0RlJJSjdwM0NsSE9PU3pKK3ZUR2ZZd0RRWUpLb1pJaHZjTkFRRUwKQ
    <Truncated>
    5QOFVFWjNyblZXWVRidWNxeDEzdUV3b2lvQi93dU4vUFkKcWI2UkdjRW1qbUNFR1gwT0tiaG9rL1BYS3RzZ1ZrY0EzdURCeldZRm1HVUhhQTdJWEplTys3Q0pPMVEzQk13PQotLS0tLUVORCBDRVJUSUZJQ0FURS0tLS0tCg==
    server: https://pksk8s01api.lab.local:8443
contexts:
- name: k8s01staging
  context:
    cluster: k8s01staging
    namespace: jenkins
    user: jenkins
current-context: k8s01staging
users:
- name: jenkins
  user:
    token: eyJhbGciOiJSUzI1NiIsImtpZCI6IiJ9.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1l
    c3BhY2UiOiJqZW5raW5zIiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZWNyZXQubmFtZSI6ImplbmtpbnMtdG9rZW4tNnh4NWwiLCJrdWJlcm5ldGVzL
    <Truncated>
    pt7EGu-Z-I5gkLget_AGRrk8KxE-xjRr06gYg7pQ8UggGo3L-ocx3QO-nsVUhnr3-d7XyCDNncivXNlMy4n6-oPXpTbbxb9_nXe_rc3XXBi5O0hkHh0k_SuiVWygvtNrmYH-m8v-isUZFO57IjXC0I7_OJZxZKnrAE-WZ9vFIQOBI-rHVT7n1A1PfZglix-ylgKgJxBoVEFhSwypjDAxbF6FA
```
Return to the Jenkins Web UI, where you left off configuring the **Kubernetes configuration (kubeconfig)** credentials. Enter a *description* such as **Jenkins Service Account Kubeconfig**. Select the radio button for **Enter Directly**. Paste the content from the updated *sa.kubeconfig* file

![alt text](https://github.com/csaroka/kubernetes-jenkins/blob/master/images/sa-kubeconfig.png)

Select **Save**

Return to the Jenkins Dashboard and select **Manage Jenkins**>**Configure System**. Scroll to the section *Cloud*>*Kubernetes* and notice the required fields.

Open the command-line and issue the command \
`$ kubectl config view` \
Copy the URL from the server parameter output and replace the form value for *Kubernetes URL*
```
apiVersion: v1
clusters:
- cluster:
    server: https://pksk8s01api.lab.local:8443
```
To collect the *Kubernetes server certificate key*, run the command: \
`$ openssl s_client -connect <Kubernetes Cluster API FQDN>:8443` \
For example, \
`$ openssl s_client -connect pksk8s01api.lab.local:8443` \
Copy the complete Server certificate and paste to the form value for *Kubernetes server certificate key*
```
-----BEGIN CERTIFICATE-----
MIIDyzCCArOgAwIBAgIUQcRSyQ0Tm99eSoAjBoYQbyCzyKgwDQYJKoZIhvcNAQEL
BQAwDTELMAkGA1UEAxMCY2EwHhcNMTgxMjA5MTQyNDE1WhcNMTkxMjA5MTQyNDE1
<Truncated>
hY8kAGeuT10K0cOVwtvsVXurIByTVettpfKOK3vMn5y5CXRtKXzJHYs8F25wojXs
1Jczjvdbfnw4miV4fK8D
-----END CERTIFICATE-----
```
Select the *Credentials* drop-down option and choose **Secret Text** 

Select **Test Connection** and test results should indicate **Connection test successful** 

![alt text](https://github.com/csaroka/kubernetes-jenkins/blob/master/images/kubernetes-cloud-config.png)

For basic operations, all other fields' default value should've populated with input from the values.yaml at the time of deployment. Verify the following:

- *Cloud*/*Kubernetes*/*Jenkins URL* = **http://jenkins:8080/jenkins**
- *Cloud*/*Kubernetes*/*Images*/*Kubernetes Pod Template*/*Containers*/*Container Template*/*Docker Image* = **< Registry Path to Jenkins Slave Image >** 
>For example, harbor.lab.local/jenkins/jenkins-slave-k8s:v1

Select **Save** and return to the Jenkins Dashboard

### Create Test Projects to Launch Executor Pods
Select **New Item** \
Enter a name for the first test project, such as **"Test Project 1"**, select **Freestyle Project**, and select **OK**

![alt text](https://github.com/csaroka/kubernetes-jenkins/blob/master/images/jenkins-testproject-1.png)

Scroll to the *Build* section, select the **Add build step** drop-down and choose **Execute shell**.  In the *Command* field enter `sleep 30`, then select **Save**

![alt text](https://github.com/csaroka/kubernetes-jenkins/blob/master/images/jenkins-build-testproject-1.png)

Repeat the process to create a second test project, **"Test Project 2"**

![alt text](https://github.com/csaroka/kubernetes-jenkins/blob/master/images/jenkins-testprojects-list.png)

Hover over the right of each project hyperlink to find a drop-down menu. For each project select **Build Now**. After initiating the builds, a build for each project should enter the build queue \

![alt text](https://github.com/csaroka/kubernetes-jenkins/blob/master/images/jenkins-buildqueue.png)

Shortly following the builds entering the build queue, Jenkins should automatically launch two additional executors inside the Kubernetes cluster for processing the queue and executing the shell commands \

![alt text](https://github.com/csaroka/kubernetes-jenkins/blob/master/images/jenkins-executor.png)

Both projects shall complete successfully \

![alt text](https://github.com/csaroka/kubernetes-jenkins/blob/master/images/jenkins-testprojects-success.png)

### Kubernetes plugin - Create and Run a Declarative Pipeline
Return to the Dashboard and Select **New Item** \
*Enter an Item Name* such as **Declarative Pipeline Example**, select **Pipeline**, and select **OK**
Scroll to the *Pipeline* section, paste the content from [Declarative Pipeline Example](https://raw.githubusercontent.com/csaroka/jenkins-kubernetes-plugin/master/examples/declarative-multiple-containers.groovy)
into the *Script* field, and select **OK*
From the Dashboard, select **Build Now**

The pipeline creates a single Kubernetes pod with two containers, from maven and busybox images

![alt text](https://github.com/csaroka/kubernetes-jenkins/blob/master/images/jenkins-declarative-success.png)

### Kubernetes-cli plugin - Execute kubectl Commands from the Shell
Return to the Dashboard and Select **New Item** \
*Enter an Item Name* such as **Kubernetes CLI Test**, select **Freestyle project**, and select **OK**

Scroll to the *Build Environment* section and select **Configure Kubernetes CLI (kubectl)** \
Select the *Credentials* drop-down menu and choose **Secret Text** \
Populate the *Kubernetes server endpoint*,*Context name*, and *Cluster name* values with data from the kubeconfig. \

![alt text](https://github.com/csaroka/kubernetes-jenkins/blob/master/images/kube-cli-buildenv.png)

Scroll to the next section, *Build*
Enter the kubectl commands to execute from the shell, for instance
>Note: Be sure to update the context and the nginx image path and tag.
```
kubectl create ns frontend-test
kubectl config set-context k8s01staging --namespace=frontend-test
kubectl run nginx01 --image=harbor.lab.local/library/nginx:v1 --replicas=4 --port=80
kubectl expose deployment nginx01 --port=80 --type=LoadBalancer
sleep 30
curl http://nginx01.frontend-test.svc.cluster.local
kubectl get all
kubectl delete ns frontend-test
```

![alt text](https://github.com/csaroka/kubernetes-jenkins/blob/master/images/kube-cli-build.png)

Select **Save**

In the left pane, select **Build Now** \
Following the build, the console output should report namespace objects and "Finished: Success"

![alt text](https://github.com/csaroka/kubernetes-jenkins/blob/master/images/kube-cli-build-out.png)

## Integrate Jenkins with a GitHub Account
Enter the login credentials and from the Dashboard select **Manage Jenkins**>**Manage Plugins**. Select the **Available** tab and enter "kubernetes" in the search filter. Choose the following plugin options:
- GitHub
- GitHub API
- GitHub Authentication
- GitHub Integration 

Select **Download now and install after restart** \
On the following page, choose **Restart Jenkins when installation is complete and no jobs are running**

After a few minutes, if the status does not change, select **Return to Dashboard** and complete the login prompt.

Login into your GitHub Account. Select the drop-down next to our profile picture, and choose **Settings**. Then select **Developer Settings**>**Personal Access Tokens**. Select **Generate New Token**. Name the token and select the privilege *admin:repo_hook*.  Copy the new access token to the clipboard or a temporary text file.

From the Jenkins Dashboard select **Credentials**>**System**>**Global credentials (unrestricted)**>**Add Credentials** \
Select the *Kind* drop-down and choose **Username with Password**
Enter your **< GitHub Username >** in the Username field, past the **< API Access Token >** in the Password field, and enter **github** for both ID and description
Select **OK**

References: 
- [Jenkins plugin to run dynamic slaves in a Kubernetes/Docker environment](https://github.com/jenkinsci/kubernetes-plugin/tree/master/examples)
- [Kubernetes Tutorials-CI/CD Pipeline](https://kubernetes.io/docs/tutorials/#ci-cd-pipeline)
- [Jenkins Helm Chart](https://github.com/helm/charts/tree/master/stable/jenkins)
