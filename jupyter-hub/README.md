# jupyter-hub_on_PKS
Instructions to get Jupyter Hub deployed and running on Kubernetes Cluster operated by PKS

## Assumptions:
* PKS v1.2+
* NSX-T v2.3+
* A PKS Kubernetes cluster with at least 1 master and 1 worker nodes
* A PKS Kubernetes cluster based on a BOSH Plan that has the following security settings configured in the Plan. NOTE: Use with Caution
** "Enable Privileged Containers"
** "Disable DenyEscalatingExec"
* Ensure you have a storage class created by the name 'default', this storage class will be used by the Persistent Volume claims needed for stateful sets.
 
* To add a storage class, copy the following into a file and name it pks-storageclass.yaml
```yaml
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: default
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
provisioner: kubernetes.io/vsphere-volume
parameters:
  diskformat: thin  
  
kubectl apply -f pks-storageclass.yaml
 ```
## Helm

Helm is the package manager for Kubernetes that runs on a local machine with kubectl access to the Kubernetes cluster. The installation process for Prometheus and the Certificate Manager leverage Helm charts available on the public Helm repo. For more information, see using [Helm with PKS](https://docs.pivotal.io/runtimes/pks/1-3/helm.html).  

* Download and install the [Helm CLI](https://github.com/helm/helm/releases) if you haven't already done so.  

* Create a service account for Tiller and bind it to the cluster-admin role. Copy the following into a file and name it rbac-config.yaml

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: tiller
  namespace: kube-system
---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: tiller-clusterrolebinding
subjects:
- kind: ServiceAccount
  name: tiller
  namespace: kube-system
roleRef:
  kind: ClusterRole
  name: cluster-admin
  apiGroup: rbac.authorization.k8s.io
 
kubectl apply -f rbac-config.yaml
 or 
  
kubectl create serviceaccount --namespace kube-system tiller
kubectl create clusterrolebinding tiller-clusterrolebinding --clusterrole=cluster-admin --serviceaccount=kube-system:tiller
  ```

* Deploy Helm using the service account by running the following command:

    `helm init --service-account tiller`


## Install Jupyter-Hub

From a Linux machine, generate a random hex string to be used as a security token by Jupyter Hub.

    `openssl rand -hex 32`
    `c46350ed823f9433312d110bf39700a765ee3cbc08f0220dff86cc63a570d3be`


We are going to edit / update the values.yaml for our jupyter-hub HELM package installation.

```yaml
singleuser:
  image:
    name: jupyter/scipy-notebook
    tag: a95cb64dfe10

  storage:
    type: none
  lifecycleHooks:
    postStart:
      exec:
        command: ["/usr/bin/git", "clone", "https://github.com/tkrausjr/finance-analysis.git", "finance-analysis"]
proxy:
  secretToken: "--put-your-randome-hex-value-here--"
```
    
* We will deploy Jupyter Hub components in a separate jupyter namespace.  Let's create the namespace : 

    `kubectl create namespace jupyter`  
    

* Add the jupyterhub HELM Repo

    `helm repo add jupyterhub https://jupyterhub.github.io/helm-chart/` 
    `helm repo update`

* Install Jupyter Hub using helm chart and reference the values.yml

    `helm upgrade --install jupyter-hub jupyterhub/jupyterhub --values values.yaml --namespace jupyter --timeout 450`  
    
    
## Jupyter Hub Validation

To verify your Jupyter deployment is successful, following Kubernetes objects must be running:

```
kubectl get all -n jupyter
    NAME                         READY   STATUS    RESTARTS   AGE
    pod/hub-66956df448-tggfz     1/1     Running   0          94s
    pod/proxy-66dc699669-nlsmc   1/1     Running   0          109s
     
    NAME                   TYPE           CLUSTER-IP       EXTERNAL-IP               PORT(S)        AGE
    service/hub            ClusterIP      10.100.200.232   <none>                    8081/TCP       3m14s
    service/kubernetes     ClusterIP      10.100.200.1     <none>                    443/TCP        6h8m
    service/proxy-api      ClusterIP      10.100.200.45    <none>                    8001/TCP       3m14s
    service/proxy-public   LoadBalancer   10.100.200.169   10.51.0.68,100.64.32.15   80:32562/TCP   3m14s
     
    NAME                    DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
    deployment.apps/hub     1         1         1            1           3m14s
    deployment.apps/proxy   1         1         1            1           3m14s
    
    NAME                               DESIRED   CURRENT   READY   AGE
    replicaset.apps/hub-66956df448     1         1         1       94s
    replicaset.apps/hub-6f98d5448d     0         0         0       3m14s
    replicaset.apps/proxy-66dc699669   1         1         1       109s
    replicaset.apps/proxy-8669976748   0         0         0       3m14s
```
**Note**:  Each time someone accesses the Jupyter Hub Web page, an additional POD will be instantiated for that user.

**Note**:  Jupyter Hub Proxy Server will have an External IP for the Service.

* To use JupyterHub, enter the external IP for the proxy-public service in to a browser. you You can access the Jupyter Hub UI using http://10.51.0.68.  JupyterHub is running with a default dummy authenticator so entering any username and password combination will let you enter the hub.

* To access the newer JupyterLab UI you can use http://10.51.0.68/lab/.
 

