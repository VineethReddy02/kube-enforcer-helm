# Aqua Kube Enforcer Helm Chart

Contents:
- [Aqua Kube Enforcer Helm Chart](#aqua-kube-enforcer-helm-chart)
  - [Configure the Kubernetes API Server (for unmanaged services) - by the Helm.](#configure-the-kubernetes-api-server-for-unmanaged-services---by-the-helm)
  - [(optional): Configure TLS Authentication to the API Server](#optional-configure-tls-authentication-to-the-api-server)
  - [Deploy The Helm Chart](#deploy-the-helm-chart)
    - [Installing the Chart](#installing-the-chart)
  - [Issues and feedback](#issues-and-feedback)

## Configure the Kubernetes API Server (for unmanaged services) - by the Helm.

This option is relevant if you are deploying the Kube-Enforcer on-premises, on Amazon Elastic Container Service (ECS), or on Microsoft Azure Kubernetes Service (ACS).

It is not relevant for deployment on managed services such as Amazon Elastic Container Service for Kubernetes (EKS), on Google Kubernetes Engine (GKE), or on Microsoft Azure Kubernetes Service (AKS).

The API Server must be configured using a preparation pod that is created on the master node. This pod enables the *ValidatingAdmissionWebhook* admission controller on the API server by modifying the API server manifest file, which is typically located at `/etc/kubernetes/manifests/kube-apiserver.yaml`.

**in values.yaml file:**
```yaml
managed: false
environment: aws # aws, acs, onprem
k8s_api_server_manifest_file_path: /etc/kubernetes/manifests/kube-apiserver.yaml
```

**Paramters**:
* managed - if you using managed kubernetes as EKS, AKS and more.
* environment - define your environment the options are
  * aws
  * acs
  * onprem
* k8s_api_server_manifest_file_path - If you have changed the location of your API server manifest file, insert its complete file path the default `/etc/kubernetes/manifests/kube-apiserver.yaml`.

## (optional): Configure TLS Authentication to the API Server

You may optionally enable TLS authentication from the API Server to the Kube-Enforcer. Perform these steps on the master node of the Kubernetes cluster.

1. Run these commands to create an API server client certificate which is signed by the CA certificate. Be sure to specify `kubernetes.default.svc` as CN.

```shell
mkdir -p /etc/kubernetes/auth && cd /etc/kubernetes/auth
openssl genrsa -out ca.key 2048
openssl req -x509 -new -nodes -key ca.key -sha256 -days 1024 -out ca.pem
openssl genrsa -out client.key 2048
openssl req -new -key client.key -out client.csr
openssl x509 -req -in client.csr -CA ca.pem -CAkey ca.key -CAcreateserial -out  \ client.pem -days 1024 -sha256
```

2. Run the following command to enable client authentication on the API server.

```shell
KUBECONFIG=/etc/kubernetes/auth/creds.yaml kubectl config set-credentials kube-enforcer.default.svc --embed-certs --client-certificate=client.pem --client-key=client.key
```

3. Add the following lines to the file /etc/kubernetes/admissionconf.yaml (or create the file with this text if it does not exist).

**/etc/kubernetes/admissionconf.yaml (snippet)**

```yaml
apiVersion: apiserver.k8s.io/v1alpha1
kind: AdmissionConfiguration
plugins:
-name: ValidatingAdmissionWebhook
  configuration:
    apiVersion: apiserver.config.k8s.io/v1alpha1
    kind: WebhookAdmission
    kubeConfigFile: /etc/kubernetes/auth/creds.yaml
```

5. Edit the file `/etc/kubernetes/manifests/kube-apiserver.yaml` and add `--admission-control-config-file=/etc/kubernetes/admissionconf.yaml` to the API server command-line arguments. Run the following to create a Kubernetes secret that contains the CA certificate:

```shell
kubectl create secret generic apiserver-creds --from-file /etc/kubernetes/auth/ca.pem
```

**and**

now you need to define at the values.yaml file (if you want to configure TLS Authentication):

```yaml
config_tls_auth: true
config_tls_secret_name: apiserver-creds
config_tls_secret_key: ca.pem
```

## Deploy The Helm Chart

**(Optional)** create the secret:

```bash
kubectl create secret docker-registry csp-registry-secret  --docker-server="registry.aquasec.com" --namespace aqua --docker-username="jg@example.com" --docker-password="Truckin" --docker-email="jg@example.com"
```

### Installing the Chart

Clone the GitHub repository with the charts

```bash
git clone https://github.com/aquasecurity/kube-enforcer-helm.git
```

***Optional*** Update the Helm charts values.yaml file with your environment's custom values. This eliminates the need to pass the parameters to the helm command. Then run one of the commands below to install the relevant services.

```bash
helm upgrade --install --namespace <namespace> kubeenforcer ./kube-enforcer-helm --set imageCredentials.username=<>,imageCredentials.password=<>,imageCredentials.email=<>
```

## Issues and feedback

If you encounter any problems or would like to give us feedback on deployments, we encourage you to raise issues here on GitHub.