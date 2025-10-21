# CNL Setup

This setup expects you to have two clusters running and uses the following conventions for KUBECONFIGS:

```sh
# kubeconfig to access the cluster kcp is running in
export KUBECONFIG=kcp-cluster.kubeconfig

# kubeconfig to access the cluster cloudnativepg is running in
export KUBECONFIG=provider.kubeconfig
```

# Install kcp

following commands are all in kcp Kubernetes cluster

```sh
export KUBECONFIG=kcp-cluster.kubeconfig
```

```sh
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.18.2/cert-manager.crds.yaml
helm upgrade \
  --install \
  --wait \
  --namespace cert-manager \
  --create-namespace \
  --version v1.18.2 \
  cert-manager jetstack/cert-manager
```

```sh
helm upgrade \
  --install \
  --values ./kcp/values.yaml \
  --namespace kcp \
  --create-namespace \
  --version "0.12.5" \
  kcp kcp/kcp
```

## Retrieving the inital kcp access admin token

```sh
export KCP_EXTERNAL_HOSTNAME=$(yq '.externalHostname' kcp/values.yaml)
kubectl --kubeconfig=kcp-admin.kubeconfig config set-cluster base --server https://$KCP_EXTERNAL_HOSTNAME:8443 --certificate-authority=ca.crt
kubectl --kubeconfig=kcp-admin.kubeconfig config set-cluster root --server https://$KCP_EXTERNAL_HOSTNAME:8443/clusters/root --certificate-authority=ca.crt
```

```sh
kubectl apply -f kcp/admin-client-cert-request.yaml
kubectl get secret cluster-admin-client-cert -o=jsonpath='{.data.tls\.crt}' | base64 -d > client.crt
kubectl get secret cluster-admin-client-cert -o=jsonpath='{.data.tls\.key}' | base64 -d > client.key
chmod 600 client.crt client.key
kubectl --kubeconfig=kcp-admin.kubeconfig config set-credentials kcp-admin --client-certificate=client.crt --client-key=client.key --embed-certs=true
kubectl --kubeconfig=kcp-admin.kubeconfig config set-context base --cluster=base --user=kcp-admin
kubectl --kubeconfig=kcp-admin.kubeconfig config set-context root --cluster=root --user=kcp-admin
kubectl --kubeconfig=kcp-admin.kubeconfig config use-context root
```

# Setup Provider Workspace

```sh
export KUBECONFIG="kcp-admin.kubeconfig"
kubectl create workspace provider
```

# SetUp CloudNativePG 

All the following commands are to be run using the provider kubeconfig

```sh
export KUBECONFIG=provider.kubeconfig
```

```sh
kubectl apply --server-side -f \
  https://raw.githubusercontent.com/cloudnative-pg/cloudnative-pg/release-1.27/releases/cnpg-1.27.0.yaml
```

# Live Demo

## Create the Provider workspace

```sh
k apply -f provider/databases-apiexport.yaml
```

## Install kcp-api-syncagent

Create the provider kubeconfig. Note for breviety we are just re-using the admin certificate here.
For prod setups, of course you would re-use the Certificate approach outlined earlier.

```sh
yq '.clusters[0].cluster.server += ":provider"' kcp-admin.kubeconfig | sed 's/admin-kcp/provider-kcp/g' > provider-kcp.kubeconfig
kubectl --kubeconfig=provider-kcp.kubeconfig config set-credentials kcp-admin --client-key=client.key --client-certificate=client.crt --embed-certs=true
kubectl --kubeconfig=provider-kcp.kubeconfig config set-cluster "workspace.kcp.io/current" --server https://$KCP_EXTERNAL_HOSTNAME:8443/clusters/root:provider --certificate-authority=ca.crt --embed-certs=true
```

Install kcp-api-syncagent:

```sh
kubectl create namespace kcp-sync-agent
kubectl create secret generic kcp-kubeconfig -n kcp-sync-agent --from-file=kubeconfig=provider-kcp.kubeconfig
kubectl apply -f api-syncagent/rbac.yaml
helm upgrade \
  --install \
  --values api-syncagent/values.yaml \
  --namespace kcp-sync-agent \
  --create-namespace \
  --version "0.4.2" \
  kcp-api-syncagent kcp/api-syncagent
```

## Create the published resource

```sh
kubectl apply -f api-syncagent/published-resource.yaml
```

## Create the consumer workspace and kubeconfig

```sh
export KUBECONFIG=kcp-admin.kubeconfig
kubectl create workspace consumer
yq '.clusters[0].cluster.server += ":consumer"' kcp-admin.kubeconfig | sed 's/admin-kcp/provider-kcp/g' > consumer-kcp.kubeconfig
kubectl --kubeconfig=consumer-kcp.kubeconfig config set-cluster "workspace.kcp.io/current" --server https://$KCP_EXTERNAL_HOSTNAME:8443/clusters/root:consumer --certificate-authority=ca.crt --embed-certs=true
```

## Create the ApiBinding & Database Object

```sh
export KUBECONFIG=consumer-kcp.kubeconfig
kubectl apply -f consumer/api-binding.yaml
```

```sh
kubectl apply -f consumer/postgres-cluster.yaml
```

