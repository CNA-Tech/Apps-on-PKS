# Installing Prometheus in a Kubernetes Cluster created by PKS with NSX-T Using Helm
This topic describes how to install [Prometheus](https://prometheus.io/) in a Kubernetes cluster created by PKS with NSX-T using Helm.

## Prerequisites:
Berfore performaing the procedure in this topic, you must have installed and cofigured the following: 

* PKS v1.2+
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

Helm is the package manager for Kubernetes that runs on a local machine with `kubectl` access to the Kubernetes cluster. The installation process for Prometheus and the Certificate Manager leverage Helm charts available on the public Helm repo. For more information, see [Using Helm with PKS](https://docs.pivotal.io/runtimes/pks/1-3/helm.html).  

* Download and install the [Helm CLI](https://github.com/helm/helm/releases) if you haven't already done so.  

* Create a service account for Tiller and bind it to the cluster-admin role. Copy the following into a file named `rbac-config.yaml`

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
 ```

* Apply the configuration using the following command: 

    `kubectl apply -f rbac-config.yaml`

* Alternatively, you can use:

    `kubectl create serviceaccount --namespace kube-system tiller`
    
    `kubectl create clusterrolebinding tiller-clusterrolebinding --clusterrole=cluster-admin --serviceaccount=kube-system:tiller`


* Deploy Helm using the service account by running the following command:

    `helm init --service-account tiller`

## Cert Manager

The cert-manager is a native Kubernetes certificate management controller.  The cert-manager can help with issuing certificates and will ensure certificates are valid and up to date; it will also attempt to renew certificates at a configured time before expiry. Start by install the cert-manager CRD:

* To Install the Kubernetes cert-manager CRDs:

```
    kubectl apply
       -f https://raw.githubusercontent.com/jetstack/cert-manager/release-0.6/deploy/manifests/00-crds.yaml
```

* Update helm repository cache

    `helm repo update`

* Install cert-manager with Helm
```
helm install \
   --name cert-manager \
   --namespace cert-manager \
   --version v0.6.0 \
   stable/cert-manager
```

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
        --namespace=default
```

  
**Note**: Issuer can be namespace(Kind: Issuer) or cluster(Kind: ClusterIssuer) scoped. We will use namespace scoped issuer in this example.

**Note**: Above is an example only.  You should follow your enterprise processes for Certificate management.   

* To create a certificate issuer, copy / paste the below YAML file as issuer.yaml

```yaml
apiVersion: certmanager.k8s.io/v1alpha1
kind: Issuer
metadata:
  name: ca-issuer
  namespace: default
spec:
  ca:
    secretName: ca-key-pair
    
kubectl create -f issuer.yaml
```

* In order to obtain a Certificate, we must create a Certificate resource in the same namespace as the Issuer.  Issuer is a namespaced resource in this example.  To obtain a signed Certificate, copy / paste the following into a file and name it as desired-cert.yaml. 

```yaml
apiVersion: certmanager.k8s.io/v1alpha1
kind: Certificate
metadata:
  name: example-com
  namespace: default
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

kubectl create -f desired-cert.yaml
```
**Note**: Above is an example only.  You should follow your enterprise processes for Certificate management.

**Note**: Actual namespace you use will be based on design requirements. We are using namespace default in this example as this namespace is available in all PKS deployments.   

**Note**: secretName in the certificate request will be referenced by the ingress controller.

## Cert Manager Validation 

To verify your installation has complete:
```
root@cli-vm:~/app# kubectl get pod -n cert-manager
NAME                                   READY     STATUS      RESTARTS   AGE
cert-manager-6874795dc8-njsxn          1/1       Running     0          1d
cert-manager-webhook-ca-sync-vsxqc     0/1       Completed   0          1d
cert-manager-webhook-d85fcf8cf-nln8v   1/1       Running     0          1d
root@cli-vm:~/app#
```

Confirm self-signed and signed cert are loaded in key store:

```
root@cli-vm:~/app# kubectl get secret | grep kubernetes.io/tls
ca-key-pair                                        kubernetes.io/tls                     2         1d
example-com-tls                                    kubernetes.io/tls                     3         1d
root@cli-vm:~/app#

Retrieve the signed TLS key pair

root@cli-vm:~/app#kubectl get secret example-com-tls -o yaml

```


## Install Prometheus

We are going to create the following customization file so our Prometheus deployment will leverage persistent volume for metrics and expose Grafana service to external users using an ingress controller, with SSL termination for added security.  Cert manager will issue and maintain the certificate required by the ingress controller. Core Prometheus services will not be externally accessible unless explicit port forwarding is enabled.  Copy following into a file and name it custom.yaml:

```yaml
alertmanager:
  alertmanagerSpec:
    storage:
      volumeClaimTemplate:
        spec:
          accessModes: ["ReadWriteOnce"]
          resources:
            requests:
              storage: 1Gi

prometheus:
  prometheusSpec:
    storage:
      volumeClaimTemplate:
        spec:
          accessModes: ["ReadWriteOnce"]
          resources:
            requests:
              storage: 1Gi

grafana:
  adminPassword: "VMware1!"
  ingress:
    enabled: true
    annotations:
      kubernetes.io/ingress.class: nginx
    hosts:
      - grafana.test.example.com
    tls:
      - secretName: example-com-tls
        hosts:
          - grafana.test.example.com
  persistence:
    enabled: true
    accessModes: ["ReadWriteOnce"]
    size: 1Gi
```

**Note**: secretName must match name specificied in the certificate sign request.  See desired-cert.yaml from prior steps.

* Install a nginx ingress controller using helm

    `helm install stable/nginx-ingress --name quickstart`
    
* We will deploy Promethus components in the monitoring namespace.  Let's create a monitoring namespace : 

    `kubectl create namespace monitoring`  
    
  
* Install Prometheus Operator using helm chart and reference the custom.yml

    `helm install stable/prometheus-operator --name prometheus --namespace monitoring -f custom.yaml`  
    
    make sure following repo is available if helm install fails     
    `root@cli-vm:~/helm_rback# helm repo list`    
    `NAME    URL`  
    `stable  https://kubernetes-charts.storage.googleapis.com`
    
## Prometheus Validation

To verify your Prometheus deployment is successful, following PODS must be running:

```
root@cli-vm:~/app# kubectl get pods -n monitoring
NAME                                                     READY     STATUS    RESTARTS   AGE
alertmanager-prometheus-prometheus-oper-alertmanager-0   2/2       Running   0          1d
prometheus-grafana-7c695dc65d-c5xlq                      3/3       Running   0          1d
prometheus-kube-state-metrics-7459b6864b-l4dpg           1/1       Running   0          1d
prometheus-prometheus-node-exporter-4cs2m                1/1       Running   0          1d
prometheus-prometheus-node-exporter-8bjk6                1/1       Running   0          1d
prometheus-prometheus-node-exporter-lndjb                1/1       Running   0          1d
prometheus-prometheus-oper-operator-58b59489fc-96m9m     1/1       Running   0          1d
prometheus-prometheus-prometheus-oper-prometheus-0       3/3       Running   1          1d
```
**Note**:  Number of node exporters PODS would vary depending on the number of worker nodes in the cluster.

```
root@cli-vm:~/app# kubectl get svc -n monitoring
NAME                                      TYPE           CLUSTER-IP       EXTERNAL-IP                PORT(S)             AGE
alertmanager-operated                     ClusterIP      None             <none>                     9093/TCP,6783/TCP   1d
prometheus-grafana                        ClusterIP      10.100.200.136   <none>                     80/TCP              1d
prometheus-kube-state-metrics             ClusterIP      10.100.200.54    <none>                     8080/TCP            1d
prometheus-operated                       ClusterIP      None             <none>                     9090/TCP            1d
prometheus-prometheus-node-exporter       ClusterIP      10.100.200.196   <none>                     9100/TCP            1d
prometheus-prometheus-oper-alertmanager   ClusterIP      10.100.200.189   <none>                     9093/TCP            1d
prometheus-prometheus-oper-operator       ClusterIP      10.100.200.137   <none>                     8080/TCP            1d
prometheus-prometheus-oper-prometheus     ClusterIP      10.100.200.109   <none>                     9090/TCP            1d
```
**Note**:  Prometheus services do not have External IPs Mapped.

* Outside of Grafana, Prometheus services are not accessible outside of the cluster.  If you want to reach Prometheus externally (from your desktop as an example), you can use port forwarding
 
    `kubectl port-forward prometheus-prometheus-oper-prometheus -n monitoring 9090:9090`
 
 Once port-forwarding is enabled, you can access the promethus UI using http://127.0.0.1:9090
 
 
* For the Grafana dashboard access, the ingress controller is set up to route based on the incoming URL.  To reach the correct Grafana endpoint, we must pass the entire FQDN and file path to the ingress controller.   FQDN (grafana.test.example.com) must be resolvable to an IP.  To find the external IP address of the Grafana service

    `kubectl get svc -n monitoring | grep  quickstart-nginx-ingress-controller | awk '{print $4}' | awk -F , '{print $1}'`
	
In a production environment, you would register external IP to FQDN mapping in a DNS server. In a pre-production development environment, it's easiest to create a temporary lookup entry in your /etc/hosts file.

* paste `https://grafana.test.example.com/` into a web browser -  Username/Password: admin/VMware1!.
