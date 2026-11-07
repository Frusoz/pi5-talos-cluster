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
  gateway-api-crds/      # Gateway API CRDs, from the upstream Git repository
  local-path-storage/    # storage provisioner
  cert-manager/          # certificate controller
  traefik/               # gateway controller
  pihole/                # DNS server and ad blocker
  home-assistant/        # home automation
```

## Applications

| Name | Chart | Version |
|---|---|---|
| local-path-provisioner | `local-path-provisioner` | 0.0.38 |
| cert-manager | `cert-manager` | v1.21.3 |
| traefik | `traefik` | 42.2.0 |
| pihole | `pihole` | 2.2.1 |
| home-assistant | `home-assistant` | 3.3.2 |

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

The Gateway carries that one listener. Traefik binds host port 443 and nothing else, so
an `http://` address does not answer. Every address of this cluster starts `https://`.

A device trusts the domain after an import of the CA certificate:

```sh
kubectl -n cert-manager get secret home-lan-ca \
  -o jsonpath='{.data.tls\.crt}' | base64 -d > home-lan-ca.crt
```

## Pi-hole

Pi-hole answers DNS on port 53 at the node address. The pod uses `hostNetwork`, so each
query keeps the address of the client. The web interface listens on port 8080 on the
node.

One dnsmasq line answers the whole domain:

```text
address=/home.lan/192.168.1.185
```

`home.lan` and every name under it resolve to the node, so a new application behind the
Traefik Gateway needs a HTTPRoute and no record here.

Pi-hole binds the node address alone. `listeningMode: BIND` with `FTLCONF_dns_interface`
keeps it off the wildcard address, because Talos host DNS holds `127.0.0.53:53` on the
same node.

The web interface answers at `https://pihole.home.lan`. The chart writes a random
password at the first install:

```sh
kubectl -n pihole get secret pihole -o jsonpath='{.data.password}' | base64 -d
```

## Home Assistant

Home Assistant answers at `https://home-assistant.home.lan`. The first visit opens the
account setup page, and this repository holds no credential.

The release keeps `/config` on a 10 GiB `local-path` volume. A StatefulSet owns that
volume, so a `helm uninstall` leaves the claim in place.

The pod stays on the cluster network. Discovery over mDNS and SSDP needs the host
network, and the `home-assistant` namespace enforces the `baseline` Pod Security level,
which forbids it. A device that answers on a fixed address is added by hand instead.

Traefik terminates TLS, so Home Assistant sees the address of the gateway and not the
address of the client. Settings -> System -> Network holds the trusted proxy list that
restores the client address.

The chart carries its own Namespace manifest. `apps/home-assistant/namespace.yaml`
creates that namespace first, because the HelmRepository and the HelmRelease live in it,
and the Helm ownership metadata on it lets the release adopt it.

## Commands

```sh
kustomize build apps/     # check the manifests before a commit
flux get helmreleases -A
kubectl get gateway -A
```
