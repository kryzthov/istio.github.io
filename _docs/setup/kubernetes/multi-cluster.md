# Istio Multi-Cluster Setup on GKE

## Istio CA Configuration

When using a single Istio control plane, you can skip this step.

Make sure all Istio CA deployments (`istio-ca`) use the same CA certificate.
TODO: Add some more details here.

## Credentials

### Generate client certificates for all clusters

In each cluster, generate client certificates to be used by Pilot and by CoreDNS.

For example, if using [CFSSL](https://github.com/cloudflare/cfssl):
```
go get -v github.com/cloudflare/cfssl/cmd/cfssl
```

1. Generate a client private key and CSR:
```bash
cfssl genkey - << EOF | cfssljson -bare multi-cluster-client
{
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "CN": "multi-cluster-client"
}
EOF
```
This command creates 2 files:
 - `multi-cluster-client-key.pem` : an RSA private key;
 - `multi-cluster-client.csr`  : a Certificate Signing Request for the Common Name (CN) `multi-cluster-client`.

2. Repeat the following steps in each cluster:
   ```
   CLUSTER_NAME='...'
   K8S_CONTEXT='...'  # kubectl context for the cluster
   ```

    1. Send the CSR to each K8s cluster:
```bash
cat <<EOF | kubectl --context=${K8S_CONTEXT} create -f -
apiVersion: certificates.k8s.io/v1beta1
kind: CertificateSigningRequest
metadata:
  name: multi-cluster-client-csr
spec:
  groups:
  - system:authenticated
  request: $(base64 multi-cluster-client.csr | tr -d '\n')
  usages:
  - digital signature
  - key encipherment
  - client auth
EOF
```

    2. Have the cluster approve the CSR:
```bash
kubectl --context=${K8S_CONTEXT} certificate approve multi-cluster-client-csr
```
    3. Retrieve the client certificates signed by the cluster:
```bash
kubectl --context=${K8S_CONTEXT} get csr multi-cluster-client-csr -o jsonpath='{.status.certificate}' \
    | base64 --decode \
    > multi-cluster-client-cert-${CLUSTER_NAME}.pem
```

    4. Authorize the client `multi-cluster-client` for which we generated a certificate:
```bash
cat << EOF | kubectl --context=${K8S_CONTEXT} apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: multi-cluster-client
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
  - apiGroup: rbac.authorization.k8s.io
    kind: User
    name: multi-cluster-client
EOF
```

3. Summary

After these steps are completed in each cluster, you should have:
 - the client private key `multi-cluster-client-key.pem`;
 - one client certificate per cluster: `multi-cluster-client-${CLUSTER_NAME}-cert.pem`.

## Pilot Configuration

The Pilot multi-cluster configuration relies on:
 - a ConfigMap that contains one [Cluster](https://github.com/kubernetes/cluster-registry/blob/8ec3a68e0da81c80f146cb3e010907edbb0737a6/pkg/apis/clusterregistry/v1alpha1/types.go#L29-L45) YAML descriptor per cluster;
 - a Secret that contains one `kubeconfig` YAML file per cluster.

### Cluster Registry ConfigMap

Below is an example of a Cluster Registry ConfigMap describing 2 clusters named `cluster-1` and `cluster-2`.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
    namespace: istio-system
    name: pilot-cluster-registry
data:
    cluster-1: |
        apiVersion: clusterregistry.k8s.io/v1alpha1
        kind: Cluster
        metadata:
            name: cluster-1
            annotations:
                config.istio.io/pilotEndpoint: "127.0.0.1:8080"
                config.istio.io/platform: "Kubernetes"
                config.istio.io/pilotCfgStore: true
                config.istio.io/accessConfigSecret: multi-cluster-kubeconfig
                config.istio.io/accessConfigSecretNamespace: istio-system
        spec:
            kubernetesApiEndpoints:
                serverEndpoints:
                  - clientCidr: "0.0.0.0/0"
                    serverAddress: "123.132.231.213"

    cluster-2: |
        apiVersion: clusterregistry.k8s.io/v1alpha1
        kind: Cluster
        metadata:
            name: cluster-2
            annotations:
                config.istio.io/pilotEndpoint: "127.0.0.1:8080"
                config.istio.io/platform: "Kubernetes"
                config.istio.io/pilotCfgStore: false
                config.istio.io/accessConfigSecret: multi-cluster-kubeconfig
                config.istio.io/accessConfigSecretNamespace: istio-system
        spec:
            kubernetesApiEndpoints:
                serverEndpoints:
                  - clientCidr: "0.0.0.0/0"
                    serverAddress: "123.132.213.231"
```

Notes:
 - that exactly one cluster must have the annotation `config.istio.io/pilotCfgStore` set to `true`. This identifies the master cluster in the mesh.
 - The ConfigMap references a K8s secret named `multi-cluster-kubeconfig` in the `istio-system` namespace (see below).

### Access Config Secret

In order for Pilot to be able to connect to the cluster, we need to construct one self-contained `kubeconfig` file per cluster.

#### Cluster `kubeconfig` access file

```shell
CLUSTER_NAME='...'

kubectl \
    --kubeconfig=kubeconfig-${CLUSTER_NAME} \
    config set-context ${CLUSTER_NAME} \
    --cluster=${CLUSTER_NAME} \
    --user=multi-cluster-client

kubectl \
    --kubeconfig=kubeconfig-${CLUSTER_NAME} \
    config set-cluster ${CLUSTER_NAME}\
    --embed-certs \
    --server=https://123.132.231.213 \
    --certificate-authority=...

kubectl \
    --kubeconfig=kubeconfig-${CLUSTER_NAME} \
    config set-credentials multi-cluster-client \
    --embed-certs \
    --client-key=multi-cluster-client-key.pem \
    --client-certificate=multi-cluster-client-cert-${CLUSTER_NAME}.pem

kubectl \
    --kubeconfig=kubeconfig-${CLUSTER_NAME} \
    config use-context ${CLUSTER_NAME}
```

Notes:
 - Make sure to use the correct address for the cluster, and to also provide the public certificate of the cluster.
 - On GKE, you can retrieve the cluster's address (Endpoint) and public certificate from the Kubernetes Engine page on the Google Cloud Console. Alternatively, you can also retrieve these details using `gcloud container clusters describe --zone=${CLUSTER_ZONE} ${CLUSTER_NAME}`, or:
```shell
CLUSTER_NAME='...'
CLUSTER_ZONE='...'

gcloud container clusters describe --zone=${CLUSTER_ZONE} ${CLUSTER_NAME} \
    --format='value(masterAuth.clusterCaCertificate)' \
    | base64 --decode \
    > ${CLUSTER_NAME}-cert.pem

kubectl \
    --kubeconfig=kubeconfig-${CLUSTER_NAME} \
    config set-cluster ${CLUSTER_NAME}\
    --embed-certs \
    --server=https://"$(gcloud container clusters describe --zone=${CLUSTER_ZONE} ${CLUSTER_NAME} --format='value(endpoint)')" \
    --certificate-authority=${CLUSTER_NAME}-cert.pem
```

The `kubeconfig-${CLUSTER_NAME}` file should look like:
```yaml
apiVersion: v1
kind: Config
current-context: ${CLUSTER_NAME}
clusters:
- name: ${CLUSTER_NAME}
  cluster:
    certificate-authority-data: [base-64 encoded public cert of the CA]
    server: https://123.132.213.231
contexts:
- name: ${CLUSTER_NAME}
  context:
    cluster: ${CLUSTER_NAME}
    user: multi-cluster-client
users:
- name: multi-cluster-client
  user:
    client-certificate-data: [base-64 encoded client certificate]
    client-key-data: [base-64 encoded client private key]
```

You can verify the `kubeconfig` file with:
```
kubectl --kubeconfig=kubeconfig-${CLUSTER_NAME} version
```

#### K8s Secret

At this point, you should have one `kubeconfig` file per cluster.
We can create the K8s Secret referenced from the ConfigMap above.

Assuming a mesh with 2 clusters named `cluster-1` and `cluster-2`:

```shell
kubectl create secret generic multi-cluster-kubeconfig --namespace=istio-system \
    --from-file=cluster-1=kubeconfig-cluster-1 \
    --from-file=cluster-2=kubeconfig-cluster-2
```

Note:
 - It is important that each kubeconfig file is keyed using the cluster name in the secret.
When Pilot initializes the cluster named `cluster-1`, Pilot looks for an entry `cluster-1` in
the secret, and expects this entry to be a kubeconfig file.
 - This step must be repeated in all clusters where Pilot will be running.

#### Pilot configuration

For Pilot 0.8+, make sure Pilot includes the following flag:
```
    --clusterRegistriesConfigMap=pilot-cluster-registry
    --clusterRegistriesNamespace=istio-system
```

##### Pilot 0.7.1

For Pilot 0.7.1, the configuration and access config secrets must be mounted as volumes
into the discovery container, and Pilot must include the following flags:
```
    --clusterRegistriesDir=/path/to/configmap
```

The configuration file is also slightly different and relies on `config.istio.io/accessConfigFile` annotations to reference `kubeconfig` files, instead of K8s secrets.

For instance, a Cluster configuration looks like:
```yaml
apiVersion: clusterregistry.k8s.io/v1alpha1
kind: Cluster
metadata:
  name: cluster-2
  annotations:
    config.istio.io/pilotEndpoint: "127.0.0.1:8080"
    config.istio.io/platform: "Kubernetes"
    config.istio.io/pilotCfgStore: false
    config.istio.io/accessConfigFile: /path/to/secret/kubeconfig
spec:
  kubernetesApiEndpoints:
    serverEndpoints:
      - clientCidr: "0.0.0.0/0"
        serverAddress: "123.132.213.231"
```

## CoreDNS Configuration

In order for service names to resolve correctly across the mesh,
we rely on CoreDNS with the Kubernetai plugin.

### Compile CoreDNS with Kubernetai

- Download CoreDNS:
```
go get -d github.com/coredns/coredns github.com/coredns/kubernetai
```
- Edit the plugin configuration file: `src/github.com/coredns/coredns/plugin.cfg`,
and add the [Kubernetai](https://github.com/coredns/kubernetai) plugin:
```shell
cat << EOF >> src/github.com/coredns/coredns/plugin.cfg
kubernetai:github.com/coredns/kubernetai/plugin/kubernetai
EOF
```
- Update the generated code:
```
go generate github.com/coredns/coredns
```
- Rebuild CoreDNS:
```
go install github.com/coredns/coredns
```
- Verify CoreDNS has the Kubernetai plugin reported in the list:
```
coredns --plugins
```
- Package CoreDNS into a container image.
  For example, using a `Dockerfile` similar to:
```
FROM scratch
COPY coredns /coredns
```


### Configure CoreDNS

#### Master cluster

In the master cluster, where Pilot is configured with the `config.istio.io/pilotCfgStore`
annotation set to `true`, KubeDNS is sufficient and CoreDNS is not required.

#### Secondary clusters

In other clusters, we can use CoreDNS with the following configuration:
```
. {
  log
  errors

  # Resolve `.cluster.local` names against services from the master cluster first:
  kubernetai cluster.local {
    endpoint https://123.132.213.231
    tls /config/cert.pem /config/key.pem /config/cacert.pem
    fallthrough
  }

  # Otherwise, resolve `.cluster.local` names against services declared in this cluster:
  kubernetai cluster.local {
  }

  # Everything else goes to a regular DNS:
  forward . 8.8.8.8:53 {
    except cluster.local
  }
}
```

Create a ConfigMap with the CoreDNS configuration file, and with the client credentials for the master cluster.

Deploy CoreDNS in the `kube-system` namespace (alongside `kube-dns`):
```yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: coredns
  namespace: kube-system
spec:
  replicas: 1
  template:
    metadata:
      labels:
        k8s-app: kube-dns
        app: coredns
    spec:
      containers:
      - name: coredns
        image: gcr.io/ct-istio/coredns:20180418-2149
        command: [
            "/coredns",
            "--log",
            "--conf=/config/coredns.conf",
        ]
        volumeMounts:
        - mountPath: /config
          name: config
      volumes:
      - name: config
        configMap:
          defaultMode: 420
          name: coredns
```

Update the `kube-dns` service definition to use the CoreDNS pods only (through selector `app: coredns`):
```yaml
apiVersion: v1
kind: Service
metadata:
  name: kube-dns
  namespace: kube-system
spec:
  type: ClusterIP
  selector:
    app: coredns
    k8s-app: kube-dns
  ports:
  - name: dns
    port: 53
    protocol: UDP
    targetPort: 53
  - name: dns-tcp
    port: 53
    protocol: TCP
    targetPort: 53
```
