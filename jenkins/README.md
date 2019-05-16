#Install Jenkins on a PKS Cluster using Helm
Using the standard stable Helm Chart we will have a Jenkins Master exposed using NSX-T

## Prerequisites:
Before performing the procedure in this topic, you must have installed and configured the following:

* PKS v1.2+ (Currently tested against PKS v1.4)
* NSX-T v2.3+
* A Kubernetes cluster created with PKS with at least 1 master and 1 worker nodes
* Ensure you have a storage class created with the name 'default', this storage class will be used by the Persistent Volume claims needed for stateful sets.

To add the necessary storage class, create a file named `pks-storageclass.yaml` by adding the following section:
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
```

Then apply the configuration using the following command:

  `kubectl apply -f pks-storageclass.yaml`

## Helm

Helm is the package manager for Kubernetes that runs on a local machine with `kubectl` access to the Kubernetes cluster. The installation process for Jenkins and the Certificate Manager leverage Helm charts available on the public Helm repo. For more information, see [Using Helm with PKS](https://docs.pivotal.io/runtimes/pks/1-3/helm.html).  

* Download and install the [Helm CLI](https://github.com/helm/helm/releases) if you haven't already done so.  

* Create a service account for Tiller and bind it to the cluster-admin role. Copy the following into a file named `rbac-config.yaml`

```yaml
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

* Apply the configuration using the following command:

    `kubectl apply -f rbac-config.yaml`

* Alternatively, you can use:

    `kubectl create serviceaccount --namespace kube-system tiller`

    `kubectl create clusterrolebinding tiller-clusterrolebinding --clusterrole=cluster-admin --serviceaccount=kube-system:tiller`


* Deploy Helm using the service account by running the following command:

    `helm init --service-account tiller`

**Note**: For added security, consider enable SSL between [Helm and Tiller](https://docs.helm.sh/using_helm/#using-ssl-between-helm-and-tiller).

## Certificate Manager

The cert-manager is a native Kubernetes certificate management controller.  The cert-manager can help with issuing certificates and will ensure certificates are valid and up to date; it will also attempt to renew certificates at a configured time before expiry. The documentation is available [here](https://docs.cert-manager.io/en/latest/). We will be following the [Helm based installation](https://docs.cert-manager.io/en/latest/getting-started/install.html#installing-with-helm). At the time of this writing, v7.2 is the latest stable.

Start by creating the cert-manager CustomResourceDefinition (CRD):

* Install the CustomResourceDefinition resources separately

  `kubectl apply -f https://raw.githubusercontent.com/jetstack/cert-manager/release-0.7/deploy/manifests/00-crds.yaml`

* Create the namespace for cert-manager

  `kubectl create namespace cert-manager`

* Label the cert-manager namespace to disable resource validation

  `kubectl label namespace cert-manager certmanager.k8s.io/disable-validation=true`

* Add the Jetstack Helm repository

  `helm repo add jetstack https://charts.jetstack.io`

* Update your local Helm chart repository cache

  `helm repo update`

* Install the cert-manager Helm chart
```
  helm install \
    --name cert-manager \
    --namespace cert-manager \
    --version v0.7.2 \
    jetstack/cert-manager
```

* We will deploy Jenkins in the `jenkins` namespace:

    `kubectl create namespace jenkins`

* Generate cert and upload as secret into your PKS cluster (scoped to a namespace).  
```
    #Generate a signing key pair
    openssl genrsa -out ca.key 2048

    #Create a self signed Certificate
    COMMON_NAME=example.com
    openssl req -x509 -new -nodes -key ca.key \
        -subj "/CN=${COMMON_NAME}" -days 3650 \
        -reqexts v3_req -extensions v3_ca -out ca.crt

    #store cert in a Kubernetes Secret resource.
    kubectl create secret tls ca-key-pair \
        --cert=ca.crt \
        --key=ca.key \
        --namespace=jenkins
```


**Note**: Issuer can be namespace scoped (`kind: Issuer`) or cluster scoped (`kind: ClusterIssuer`). We will use namespace scoped issuer in this example.

**Note**: The sample above is provided as an example only.  You should follow your enterprise processes for Certificate management.   

* To create a certificate issuer, copy / paste the YAML sample below in a file named `issuer.yaml`

```yaml
apiVersion: certmanager.k8s.io/v1alpha1
kind: Issuer
metadata:
  name: ca-issuer
  namespace: jenkins
spec:
  ca:
    secretName: ca-key-pair
```
Then apply the configuration with:

  `kubectl create -f issuer.yaml`

* In order to obtain a Certificate, we must create a Certificate resource in the same namespace as the Issuer.  In this example, the Issuer is a namespaced resource.  To obtain a signed Certificate, copy / paste the following into a file named `desired-cert.yaml`.

```yaml
apiVersion: certmanager.k8s.io/v1alpha1
kind: Certificate
metadata:
  name: example-com
  namespace: jenkins
spec:
  secretName: example-com-tls
  issuerRef:
    name: ca-issuer
    # We can reference ClusterIssuers by changing the kind here.
    # The default value is Issuer (i.e. a locally namespaced Issuer)
    kind: Issuer
  commonName: example.com
  organization:
  - VMware
  dnsNames:
  - example.com
```
Then apply the configuration with:

  `kubectl create -f desired-cert.yaml`

**Note**: The sample above is provided as an example only.  You should follow your enterprise processes for Certificate management.

**Note**: The namespace you use will be based on design requirements. We are using the `default` namespace in this example as this namespace is available in all PKS deployments.   

**Note**: `secretName` in the certificate request will be referenced by the ingress controller.

### Certificate Manager Validation

To verify your installation was successful:
```
root@cli-vm:~/app# kubectl get pod -n cert-manager
NAME                                   READY     STATUS      RESTARTS   AGE
cert-manager-6874795dc8-njsxn          1/1       Running     0          1d
cert-manager-webhook-ca-sync-vsxqc     0/1       Completed   0          1d
cert-manager-webhook-d85fcf8cf-nln8v   1/1       Running     0          1d
root@cli-vm:~/app#
```

Confirm self-signed and signed certificates are loaded in key store:

```
root@cli-vm:~/app# kubectl get secret -n jenkins | grep kubernetes.io/tls
ca-key-pair                                        kubernetes.io/tls                     2         1d
example-com-tls                                    kubernetes.io/tls                     3         1d
root@cli-vm:~/app#
```

Retrieve the signed TLS key pair

  `root@cli-vm:~/app#kubectl get secret example-com-tls -n jenkins -o yaml`

## Installing Jenkins

We are going to customize our Jenkins deployment to leverage a larger persistent volumes for metrics and expose Jenkins Master service to external users using an ingress controller, with SSL termination for added security. cert-manager will issue and maintain the certificate required by the ingress controller. If you would like to have back ups enabled using S3 storage, add a `backup` section following the default [values.yaml](https://github.com/helm/charts/blob/master/stable/jenkins/values.yaml). Copy the following into a file and name it `custom.yaml`:

```yaml
master:
  adminUser: "admin"
  adminPassword: "VMware1!"
  ingress:
    enabled: true
    annotations:
      kubernetes.io/ingress.class: nginx
      certmanager.k8s.io/issuer: ca-issuer
      kubernetes.io/tls-acme: "true"
      path: "/jenkins"
    hosts:
      - jenkins.test.example.com
    tls:
      - secretName: example-com-tls
        hosts:
          - jenkins.test.example.com

persistence:
  size: "20Gi"
```
**Note**: `secretName` must match the name specified in the certificate signing request.  See `desired-cert.yaml` from prior steps.

* Install a nginx ingress controller using helm

    `helm install stable/nginx-ingress --name nginx`

* Install Jenkins using helm chart and reference the `custom.yaml` but disable the CRD provisioning.

    `helm install stable/jenkins --name jenkins --namespace jenkins -f custom.yaml`  

    If helm install fails, make sure the following repository is available     
    `root@cli-vm:~/helm_rback# helm repo list`    
    `NAME    URL`  
    `stable  https://kubernetes-charts.storage.googleapis.com`

## Jenkins Validation

To verify that your Jenkins deployment was successful, check that the following PODS are running:

```
root@cli-vm:~/app# kubectl get pods -n jenkins
NAME                       READY   STATUS    RESTARTS   AGE
jenkins-676586d584-9rxg5   1/1     Running   0          3m23s

```
**Note**:  //The number of node exporter PODS will vary depending on the number of worker nodes in the cluster.//KILL IT

```
root@cli-vm:~/app# kubectl get svc -n monitoring
NAME            TYPE           CLUSTER-IP      EXTERNAL-IP                  PORT(S)          AGE
jenkins         LoadBalancer   10.100.200.45   10.173.62.121,100.64.80.33   8080:32112/TCP   4m
jenkins-agent   ClusterIP      10.100.200.98   <none>                       50000/TCP        4m
```
**Note**:  Jenkins Agents do not have External IPs Mapped.

* For the Jenkins dashboard access, the ingress controller is set up to route based on the incoming URL. To reach the correct Jenkins endpoint, we must use the Fully Qualified Domain Name (FQDN) and file path to the ingress controller. In our example, FQDN (jenkins.test.example.com) must be resolvable to an IP.  To find the external IP address of the Jenkins service

    `kubectl get svc | grep  nginx-nginx-ingress-controller | awk '{print $4}' | awk -F , '{print $1}'`

    Or it can found by simply looking at the services and seeing what Load Balancer IP has been allocated from NSX-T

    ```
    root@cli-vm:~/app# kubectl get svc
    NAME                                  TYPE           CLUSTER-IP       EXTERNAL-IP                  PORT(S)                      AGE
    kubernetes                            ClusterIP      10.100.200.1     <none>                       443/TCP                      4d20h
    nginx-nginx-ingress-controller        LoadBalancer   10.100.200.235   10.173.62.118,100.64.80.33   80:32544/TCP,443:30207/TCP   14m
    nginx-nginx-ingress-default-backend   ClusterIP      10.100.200.56    <none>                       80/TCP                       14m
    ```

In a production environment, you would register external IP to FQDN mapping in a DNS server. Alternatively, for a pre-production development environment, you can create a temporary lookup entry in your /etc/hosts file.

* To access jenkins, enter `https://jenkins.test.example.com/` in a web browser -  Username/Password: admin/VMware1!

  **Note**:  Edit the `etc/hosts` file of the local computer putting the IP and DNS name for the Jenkins dashboard to appear in a browser.
