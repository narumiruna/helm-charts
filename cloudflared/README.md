# cloudflared

Run a token-authenticated Cloudflare Tunnel as a single-replica Deployment.

The chart uses the supplied Compose settings:

- Image: `cloudflare/cloudflared:latest`.
- Arguments: `tunnel --no-autoupdate run`.
- Token: `TUNNEL_TOKEN` from an existing Kubernetes Secret.
- Data: a `ReadWriteOnce` PVC mounted at `/etc/cloudflared`.

The Deployment restarts failed containers. It uses the `Recreate` strategy to prevent overlapping Pods during updates. No Service or Ingress is needed because cloudflared opens outbound connections.

## Install

Run these commands from the repository root. Set `TUNNEL_TOKEN` in your shell first. Do not put the token in `values.yaml` or commit it.

```sh
: "${TUNNEL_TOKEN:?Set TUNNEL_TOKEN to your Cloudflare Tunnel token}"

kubectl create namespace cloudflared --dry-run=client -o yaml | kubectl apply -f -
kubectl -n cloudflared create secret generic cloudflared-token \
  --from-literal=TUNNEL_TOKEN="$TUNNEL_TOKEN"

helm upgrade --install cloudflared ./cloudflared --namespace cloudflared
kubectl -n cloudflared rollout status deployment/cloudflared
kubectl -n cloudflared logs deployment/cloudflared
```

If the Secret already exists, skip the Secret creation command. The Secret must be in the same namespace as the release and contain a `TUNNEL_TOKEN` key. After you update the Secret, run `kubectl -n cloudflared rollout restart deployment/cloudflared` to load the new token.

Configure the tunnel's published application routes in Cloudflare. Use an origin address that the Pod can reach, such as `http://web.default.svc.cluster.local:80`. `localhost` refers to the cloudflared Pod, not the original Docker host.

## Values

| Key | Default | Purpose |
| --- | --- | --- |
| `image.repository` | `cloudflare/cloudflared` | Container image repository. |
| `image.tag` | `latest` | Image tag. Pin a release tag for repeatable deployments. |
| `image.pullPolicy` | `Always` | Container image pull policy. |
| `existingSecret` | `cloudflared-token` | Secret that contains `TUNNEL_TOKEN`. |
| `persistence.enabled` | `true` | Create and mount a PVC. Use `emptyDir` when `false`. |
| `persistence.size` | `1Gi` | PVC capacity. |
| `persistence.storageClass` | `null` | Use the default StorageClass. Set a name to select one, or `""` to use a pre-provisioned volume. |
| `resources` | `{}` | Container resource requests and limits. |

The cluster needs a default StorageClass unless you select a class or provide a matching volume. The PVC is new storage; the chart does not copy files from the Compose `./data` directory. If those files are needed, copy them separately and make them accessible to UID/GID `65532`.

With persistence disabled, data is lost when the Pod is removed. Disabling persistence on an existing release also deletes the chart's PVC. Back up its data first.

```sh
helm upgrade --install cloudflared ./cloudflared --namespace cloudflared \
  --set persistence.enabled=false
```

## Validate

```sh
helm lint ./cloudflared --strict
helm template cloudflared ./cloudflared --namespace cloudflared
```

## Remove

```sh
helm uninstall cloudflared --namespace cloudflared
```

Uninstalling the chart deletes its PVC. The StorageClass reclaim policy controls whether the backing volume is also deleted. The externally managed Secret is not deleted.
