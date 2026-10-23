# Vault on OrbStack

This chart wraps the official [HashiCorp Vault chart](https://developer.hashicorp.com/vault/docs/platform/k8s/helm). It pins chart `0.34.1` and Vault `2.0.4`, which has a Linux ARM64 image for Apple Silicon.

The defaults are for one Mac mini running OrbStack Kubernetes:

- One non-dev Vault server with the upstream `file` storage backend.
- A `1Gi` PVC mounted at `/vault/data`, using the default StorageClass.
- Manual initialization and unseal. No fixed root token or stored unseal key.
- ClusterIP services and the Vault UI. No Ingress, Injector, or CSI provider.
- HTTP inside the cluster. This is not a production or high-availability setup.

Other Pods can reach the HTTP service. Use this configuration only in a trusted local cluster. Do not expose it through cloudflared, a LoadBalancer, or a public Ingress without configuring TLS and access controls first.

## Install

Run from the repository root with Helm 3.6+ and Kubernetes 1.29+. Enable Kubernetes in OrbStack first. Confirm that the context is named `orbstack` and that a default StorageClass exists:

```sh
kubectl config get-contexts
kubectl --context orbstack get storageclass

helm dependency build ./vault
helm upgrade --install vault ./vault \
  --kube-context orbstack --namespace vault --create-namespace

kubectl --context orbstack -n vault get pods,pvc
```

Do not add `--wait` on the first install: Vault stays `Running` with `0/1` ready until you initialize and unseal it. If the PVC stays `Pending`, check the default StorageClass and the PVC events.

Downloaded dependencies are ignored by Git. Commit `Chart.yaml` and `Chart.lock`, not `charts/`. All upstream values are under the `vault:` key in this wrapper's `values.yaml`.

## Initialize and open the UI

Keep this command running in a separate terminal. It binds only to the Mac's loopback address:

```sh
kubectl --context orbstack -n vault port-forward \
  --address 127.0.0.1 service/vault-ui 8200:8200
```

Open <http://127.0.0.1:8200/ui/>.

1. For this local single-user setup, initialize with **1 key share** and **1 key threshold**.
2. Save the unseal key and initial root token in a password manager. Keep a recoverable copy outside this Mac. Do not put either value in this repository, Helm values, or a Kubernetes ConfigMap.
3. Unseal Vault with the key, then sign in with the initial root token.
4. Configure a normal authentication method and policies. Use the root token only for initial setup and recovery, not for applications.

One key is enough to unseal this local instance. Anyone with that key and access to its storage can recover the data; if the key is lost, the data cannot be recovered. Do not initialize again when you reuse an existing data PVC.

After unseal, the Pod becomes ready:

```sh
kubectl --context orbstack -n vault rollout status statefulset/vault
```

## Restart and update

A Vault process restart, including a Mac or OrbStack restart, seals Vault again. Restart the port-forward if needed, then use the UI to unseal with the same key. Do not run initialization again.

The upstream StatefulSet uses the `OnDelete` update strategy. After an image or configuration update, apply the release and recreate the Pod during a planned interruption:

```sh
helm dependency build ./vault
helm upgrade --install vault ./vault \
  --kube-context orbstack --namespace vault --create-namespace
kubectl --context orbstack -n vault delete pod vault-0
```

The replacement Pod uses the same PVC and must be unsealed. Back up data and review the Vault upgrade notes before changing versions. Set the PVC capacity before the first install; later expansion depends on the StorageClass and requires a separate PVC resize procedure.

## Storage and removal

A PVC preserves data when a Pod is replaced. It is not a backup and does not protect against Mac failure or an OrbStack reset.

This setup uses the `file` backend, not Raft. For a consistent backup, stop the Vault server and copy the complete data volume to separate storage. Keep the unseal key with your recovery materials. A Raft snapshot command does not back up this backend.

```sh
helm uninstall vault --kube-context orbstack --namespace vault
kubectl --context orbstack -n vault get pvc
```

The StatefulSet retention policy keeps its data PVC after uninstall. Reinstall with the same release name and namespace to reuse it. Deleting the PVC or the namespace can permanently delete the data, depending on the volume's reclaim policy.

## Validate without a cluster

```sh
helm dependency build ./vault
helm lint ./vault --strict
helm template vault ./vault --namespace vault
```
