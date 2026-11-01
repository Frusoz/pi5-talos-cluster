# pi5-talos-cluster

Flux reads this repository and applies it to a single-node Kubernetes cluster. The node
is a Raspberry Pi 5. It runs Talos Linux, and Omni manages it.

## Bootstrap

The Talos machine configuration installs Flux. Omni holds that configuration. The
GitRepository and the Kustomization live there too, and the Kustomization reads `./apps`.

To change the branch, the path, or the interval, edit the machine configuration in Omni.

## Layout

```text
apps/
  kustomization.yaml     # the list of applications
  gateway-api/           # Gateway API CRDs, from the upstream Git repository
  local-path-storage/    # storage provisioner
  cert-manager/          # certificate controller
  traefik/               # gateway controller
```

## Applications

| Name | Chart | Version |
|---|---|---|
| local-path-provisioner | `local-path-provisioner` | 0.0.38 |
| cert-manager | `cert-manager` | v1.21.3 |
| traefik | `traefik` | 42.2.0 |

One Kustomization applies the whole path in one pass. A release that needs another
release first declares `dependsOn`.

The Gateway API CRDs come from a second Flux source, not from a chart. Helm keeps a copy
of the whole chart in a Secret, and a Secret holds 1 MiB at most. A chart with these CRDs
passes that limit.

## Talos notes

- The `local-path-storage` namespace and the `traefik` namespace carry the label
  `pod-security.kubernetes.io/enforce: privileged`. Talos enforces the `baseline` level,
  and `baseline` forbids hostPath and hostPort.
- The provisioner writes to `/var/local-path-provisioner`. Talos mounts a read-only root
  filesystem, and `/var` is the one writable path that stays after a reboot.
- Each chart must provide `linux/arm64` images.

## Certificates

`apps/cert-manager/ca.yaml` holds a self-signed root CA for the `home.lan` domain. The
`home-lan-ca` ClusterIssuer signs each certificate of that domain. The private key stays
in a Secret in the cluster, and this repository holds no key.

The Traefik Gateway presents the `home-lan-tls` certificate on its `websecure`
listener. That certificate covers `home.lan` and `*.home.lan`.

A device trusts the domain after an import of the CA certificate:

```sh
kubectl -n cert-manager get secret home-lan-ca \
  -o jsonpath='{.data.tls\.crt}' | base64 -d > home-lan-ca.crt
```

## Commands

```sh
kustomize build apps/     # check the manifests before a commit
flux get helmreleases -A
kubectl get gateway -A
```
