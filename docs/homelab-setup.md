# Home lab setup: k3s, Argo CD, Prometheus and Grafana

This is a record of how the two Raspberry Pi k3s clusters were built and
how Argo CD, cert-manager and the monitoring stack were added to
`k3s-cluster1`. It covers what was changed, why, the commands used, and the
problems hit along the way.

## Overview

| Piece | Where |
|---|---|
| Playbooks (fork of k3s-io/k3s-ansible) | `~/DEV/k3s-ansible`, branch `master` |
| Inventory | `~/DEV/ansible-inventory/inventory.yml` (private repo `t4t001n3/ansible-inventory`) |
| GitOps repo watched by Argo CD | `~/DEV/homelab-gitops` (private repo `t4t001n3/homelab-gitops`) |
| Control node | `penguin` (x86_64), runs Ansible and kubectl |
| HTTPS with the home CA | [`argocd-tls-home-ca.md`](argocd-tls-home-ca.md) |
| LAN DNS / ad blocking (planned) | Pi-hole in a Proxmox LXC, see Part 5 |

| Cluster | Server | Agents | API |
|---|---|---|---|
| `k3s-cluster1` | gandalf | aragorn, legolas, gimli, boromir | `https://10.0.0.125:6443` (was `10.0.0.13`) |
| `k3s-cluster2` | shire | frodo, sam, pippin, merry | `https://10.0.0.233:6443` |

All nodes run k3s `v1.37.1+k3s1`. sam and merry register as `p2` and `p3`,
because those are their OS hostnames.

The cluster2 agents (10.0.0.149–152) connect through shire's bridge `br0`
(interfaces `ethpi1`–`ethpi4`). If shire reboots or loses power, all of
cluster2 is unreachable until it's back up.

```
                       ┌──────────── k3s-cluster1 ─────────────┐
penguin ──kubectl────▶ │ gandalf (server) ── Traefik :80/:443  │
  │                    │  ├─ argocd        Argo CD             │
  │                    │  ├─ cert-manager  home-ca issuer      │
  │                    │  └─ monitoring    kube-prometheus-stack│
  │                    │ aragorn · legolas · gimli · boromir   │
  │                    └───────────────────▲───────────────────┘
  └──git push──▶ github.com/t4t001n3/homelab-gitops ─┘ (Argo CD pulls)
```

---

## Part 1: k3s with k3s-ansible

### 1.1 Make the playbooks work with two clusters

Upstream k3s-ansible expects one cluster in the groups `k3s_cluster`,
`server` and `agent`. Ansible group names are global, so two clusters that
both use groups called `server` and `agent` would be merged into one. Each
cluster therefore gets its own group names, and the playbooks read them from
variables.

The play `hosts:` lines in `playbooks/site.yml`, `reset.yml`,
`upgrade.yml` and `reboot.yml` were changed to:

```yaml
hosts: "{{ cluster_group | default('k3s_cluster') }}"   # was k3s_cluster
hosts: "{{ server_group | default('server') }}"         # was server
hosts: "{{ agent_group | default('agent') }}"           # was agent
```

`reset.yml` also had its uninstall conditions changed:

```yaml
when: (server_group | default('server')) in group_names
when: (agent_group | default('agent')) in group_names
```

The defaults keep the upstream layout working. The roles already use
`server_group` and `agent_group` internally.

A play's `hosts:` line is evaluated **before** inventory variables are
loaded, so the group names must be passed with `-e` on the command line
(see 1.4). Setting them only in the inventory isn't enough.

### 1.2 Inventory

Simplified `inventory.yml`:

```yaml
all:
  children:
    k3s_cluster1:
      children:
        server1:
          hosts:
            gandalf:
              ansible_host: 10.0.0.125   # agents can't resolve the bare hostname
        agent1:
          hosts: { aragorn:, legolas:, gimli:, boromir: }
      vars:
        server_group: server1
        agent_group: agent1
        cluster_context: k3s-cluster1
        token: "<64 hex chars>"
        api_endpoint: "{{ hostvars[groups['server1'][0]]['ansible_host'] | default(groups['server1'][0]) }}"
    k3s_cluster2:
      children:
        server2:
          hosts:
            shire:
              ansible_host: 10.0.0.233
        agent2:
          hosts: { frodo:, sam:, pippin:, merry: }
      vars:
        server_group: server2
        agent_group: agent2
        cluster_context: k3s-cluster2
        token: "<64 hex chars>"
        api_endpoint: "{{ hostvars[groups['server2'][0]]['ansible_host'] | default(groups['server2'][0]) }}"
  vars:
    ansible_port: 22
    ansible_user: pi
    k3s_version: v1.37.1+k3s1
```

Notes:

- **The server needs `ansible_host` set to its IP.** `api_endpoint` is the
  address agents use to join. With only the bare hostname, the agents
  couldn't resolve `gandalf` and never joined.
- **Tokens** were generated with `openssl rand -hex 32`, one per cluster.
  They're stored in plain text in the inventory, which is a private repo.
  To encrypt them, use
  `ansible-vault encrypt_string --name token '<value>'` and run the
  playbooks with `--ask-vault-pass`.
- `cluster_context` names the kubeconfig context for each cluster.

### 1.3 Control node prerequisites

The system Ansible (2.14) is too old. The playbooks need ansible-core ≥ 2.15
and fail with *"Minimum ansible-core version required is 2.15"*. Use the
project virtualenv:

```bash
cd ~/DEV/k3s-ansible
python3 -m venv venv
./venv/bin/pip install ansible-core netaddr     # 2.19.11 at the time of writing
./venv/bin/ansible-galaxy collection install -r collections/requirements.yml
```

`netaddr` is needed by the `ansible.utils.ipwrap` filter.

### 1.4 Run the playbook, once per cluster

```bash
cd ~/DEV/k3s-ansible
INV=~/DEV/ansible-inventory/inventory.yml

# cluster1
./venv/bin/ansible-playbook -i $INV playbooks/site.yml \
  -e cluster_group=k3s_cluster1 -e server_group=server1 -e agent_group=agent1

# cluster2
./venv/bin/ansible-playbook -i $INV playbooks/site.yml \
  -e cluster_group=k3s_cluster2 -e server_group=server2 -e agent_group=agent2
```

Use the same `-e` flags with `reset.yml`, `upgrade.yml` and `reboot.yml`.

What to expect:

- The `raspberrypi` role may **reboot** Pis the first time, to enable
  cgroups.
- `TASK [k3s_agent : Enable and start K3s agent]` can sit there for several
  minutes on Pis while k3s starts. That's normal as long as the agent is
  making progress (`journalctl -u k3s-agent -f` on the agent).

### 1.5 Problems hit and how they were fixed

| Symptom | Cause | Fix |
|---|---|---|
| "Minimum ansible-core version required is 2.15" | System Ansible 2.14 | Use `./venv/bin/ansible-playbook` |
| Agents never join cluster1 | `api_endpoint` was the bare hostname `gandalf`, which agents couldn't resolve | Set `ansible_host` IPs on the servers |
| Kubeconfig not copied to the control node | kubectl wasn't installed on penguin | Install kubectl (1.6) and merge the config by hand |
| shire didn't come back after the role's reboot | Pi hung on reboot | Power-cycled by hand, then reran the playbook |
| frodo and pippin: become/sudo **timeout** | `/etc/hosts` didn't list the machine's own hostname, so sudo stalled resolving it | Added `127.0.1.1 p1 frodo` / `127.0.1.1 p4 pippin` (backups in `/etc/hosts.bak-k3s`) |

### 1.6 kubectl on the control node

```bash
# kubectl matching the cluster version, into ~/.local/bin
curl -Lo ~/.local/bin/kubectl https://dl.k8s.io/release/v1.37.1/bin/linux/amd64/kubectl
chmod +x ~/.local/bin/kubectl
```

The k3s kubeconfig (`/etc/rancher/k3s/k3s.yaml` on each server) was copied
over and merged into `~/.kube/config`. The server address was changed from
`127.0.0.1` to the server's IP, and the context renamed to `k3s-cluster1` /
`k3s-cluster2`. Switch between clusters with:

```bash
kubectl config use-context k3s-cluster1
kubectl --context k3s-cluster2 get nodes     # without switching
```

---

## Part 2: Argo CD (cluster1)

### 2.1 Install

The upstream `install.yaml` includes the `applicationsets.argoproj.io` CRD,
which is too large for a normal `kubectl apply`:

```
The CustomResourceDefinition "applicationsets.argoproj.io" is invalid:
metadata.annotations: Too long: may not be more than 262144 bytes
```

Client-side apply stores the whole object in an annotation, and annotations
are limited to 256 KB. **Server-side apply** doesn't use that annotation:

```bash
kubectl --context k3s-cluster1 create namespace argocd
kubectl --context k3s-cluster1 -n argocd apply --server-side --force-conflicts \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

At the time of writing, Argo CD v3.5.6 was installed.

Initial admin password:

```bash
kubectl --context k3s-cluster1 -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath='{.data.password}' | base64 -d; echo
```

### 2.2 Web UI through Traefik

k3s ships Traefik, which already listens on 80/443 on every node. A
LoadBalancer Service on those ports would conflict with it, so Argo CD is
published through an Ingress. Traefik terminates TLS, and argocd-server runs
in plain HTTP behind it:

```bash
kubectl --context k3s-cluster1 -n argocd patch configmap argocd-cmd-params-cm \
  --type merge -p '{"data":{"server.insecure":"true"}}'
kubectl --context k3s-cluster1 -n argocd rollout restart deployment argocd-server

kubectl --context k3s-cluster1 apply -f - <<'EOF'
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: argocd-server
  namespace: argocd
spec:
  ingressClassName: traefik
  rules:
  - host: argocd.10.0.0.125.nip.io
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: argocd-server
            port:
              number: 80
EOF
```

[nip.io](https://nip.io) resolves `anything.<IP>.nip.io` to `<IP>`, which
gives each service a hostname without editing DNS.

The Argo CD CLI needs `--grpc-web` through Traefik:
`argocd login argocd.10.0.0.125.nip.io --grpc-web`.

### 2.3 HTTPS

cert-manager v1.16.2 was installed with a self-signed home CA and a
`home-ca` ClusterIssuer. The Argo CD Ingress was annotated to use it, with
an HTTP→HTTPS redirect. The CA certificate is exported to
`~/DEV/home-lab-ca.crt` and imported on the devices used to browse.
**All steps are in [`argocd-tls-home-ca.md`](argocd-tls-home-ca.md).**

---

## Part 3: GitOps repo and the monitoring stack

### 3.1 Why kube-prometheus-stack

The first candidate was `t4t001n3/cluster-monitoring` (a fork of
carlosedp/cluster-monitoring). Argo CD could deploy its `manifests/`
directory, but it was rejected because:

- the generated YAML was hard-coded for another cluster (`192.168.2.66`),
  and fixing it needs a jsonnet rebuild;
- it ships 2020-era components (prometheus-operator v0.40, Grafana 7);
- its CRDs exceed the same 256 KB annotation limit.

The **kube-prometheus-stack** Helm chart (prometheus-community) is used
instead. It's maintained, has arm64 images, and Argo CD installs Helm charts
directly.

### 3.2 App of apps layout

```
homelab-gitops/
├── README.md
├── bootstrap/
│   └── k3s-cluster1-root.yaml           # root Application, applied once by hand
└── clusters/
    └── k3s-cluster1/
        └── apps/                        # one Argo CD Application per file
            └── kube-prometheus-stack.yaml
```

The **root Application** watches `clusters/k3s-cluster1/apps/`. Every file
in that directory is an Application, which Argo CD creates, updates or
deletes to match Git (`prune` + `selfHeal`).

```yaml
# bootstrap/k3s-cluster1-root.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: root-k3s-cluster1
  namespace: argocd
spec:
  project: default
  source:
    repoURL: git@github.com:t4t001n3/homelab-gitops.git
    targetRevision: master
    path: clusters/k3s-cluster1/apps
  destination:
    server: https://kubernetes.default.svc
    namespace: argocd
  syncPolicy:
    automated: { prune: true, selfHeal: true }
```

### 3.3 Argo CD access to the private repo

The repo is private. Argo CD reads it with a **read-only deploy key**:

```bash
ssh-keygen -t ed25519 -N '' -C argocd-k3s-cluster1 -f key
gh repo deploy-key add key.pub -R t4t001n3/homelab-gitops -t argocd-k3s-cluster1

kubectl --context k3s-cluster1 -n argocd create secret generic repo-homelab-gitops \
  --from-literal=type=git \
  --from-literal=url=git@github.com:t4t001n3/homelab-gitops.git \
  --from-file=sshPrivateKey=key
kubectl --context k3s-cluster1 -n argocd label secret repo-homelab-gitops \
  argocd.argoproj.io/secret-type=repository

shred -u key && rm key.pub      # the private key now lives only in the cluster
```

### 3.4 The kube-prometheus-stack Application

`clusters/k3s-cluster1/apps/kube-prometheus-stack.yaml`, chart version
`92.11.0`. The key settings:

| Setting | Why |
|---|---|
| `kubeControllerManager/kubeScheduler/kubeProxy/kubeEtcd.enabled: false` | k3s runs these inside the k3s binary on 127.0.0.1 and uses no etcd by default, so they can't be scraped and would only raise `TargetDown` alerts |
| `grafana.admin.existingSecret: grafana-admin` | Otherwise the chart generates a random password on every render, and Argo CD renders on every sync |
| `grafana.persistence` 5Gi, `prometheus...storageSpec` 20Gi | Keep dashboards and metrics across restarts (k3s `local-path` storage) |
| `*SelectorNilUsesHelmValues: false` | Prometheus picks up ServiceMonitors, PodMonitors and rules from any app, not only this chart |
| Ingresses with `cert-manager.io/cluster-issuer: home-ca` and `tls:` | HTTPS from the home CA for each UI |
| `syncOptions: ServerSideApply=true` | The Prometheus CRDs are too big for client-side apply |
| `syncOptions: CreateNamespace=true` | Argo CD creates `monitoring` if it's missing |
| `finalizers: resources-finalizer.argocd.argoproj.io` | Deleting the file also removes the stack from the cluster |

### 3.5 Bootstrap

```bash
# Grafana admin secret (kept out of Git)
kubectl --context k3s-cluster1 create namespace monitoring
kubectl --context k3s-cluster1 -n monitoring create secret generic grafana-admin \
  --from-literal=admin-user=admin \
  --from-literal=admin-password="$(openssl rand -base64 24)"

# Root app; Argo CD takes over from here
kubectl --context k3s-cluster1 apply -f ~/DEV/homelab-gitops/bootstrap/k3s-cluster1-root.yaml
```

Within a few minutes both apps showed **Synced / Healthy**. Prometheus was
scraping the API server, coredns, kubelet and node-exporter on all 5 nodes,
kube-state-metrics, and the stack itself.

Grafana password:

```bash
kubectl --context k3s-cluster1 -n monitoring get secret grafana-admin \
  -o jsonpath='{.data.admin-password}' | base64 -d; echo
```

### 3.6 Day-to-day use

- **Add an app:** add an Application file to `clusters/k3s-cluster1/apps/`,
  then commit and push. Argo CD polls about every 3 minutes, or click
  **Refresh** in the UI.
- **Change or upgrade the stack:** edit the values or `targetRevision` in
  `kube-prometheus-stack.yaml`, then commit and push.
- **Remove an app:** delete its file, then commit and push.

### 3.7 URLs (cluster1)

| UI | URL | Login |
|---|---|---|
| Argo CD | https://argocd.10.0.0.125.nip.io | `admin` / initial admin secret |
| Grafana | https://grafana.10.0.0.125.nip.io | `admin` / `grafana-admin` secret |
| Prometheus | https://prometheus.10.0.0.125.nip.io | none |
| Alertmanager | https://alertmanager.10.0.0.125.nip.io | none |

Prometheus and Alertmanager have no authentication, so anyone on the LAN can
use them. The monitoring Ingresses don't redirect HTTP→HTTPS, because the
`redirect-https` middleware lives in the `argocd` namespace and Traefik
doesn't allow cross-namespace middleware by default.

---

## Part 4: Changing the control-plane IP

gandalf's IP changed from `10.0.0.13` to `10.0.0.125`. Two things broke:

1. **Every URL returned 404.** Traefik answers 404 when no Ingress matches
   the requested host, and all Ingresses still used `*.10.0.0.13.nip.io`.
2. **All agents went NotReady.** The agent systemd unit has the server URL
   baked in (`--server https://10.0.0.13:6443` in
   `/etc/systemd/system/k3s-agent.service`), and most pods ran on the agents.

The k3s API certificate picked up the new IP on its own, so the server
needed no changes.

Fix, in order:

```bash
# 1. Inventory: gandalf's ansible_host → 10.0.0.125

# 2. Local kubeconfig
cp ~/.kube/config ~/.kube/config.bak-ipchange
sed -i 's#https://10.0.0.13:6443#https://10.0.0.125:6443#' ~/.kube/config

# 3. Reinstall agents with the new server URL
cd ~/DEV/k3s-ansible
./venv/bin/ansible-playbook -i ~/DEV/ansible-inventory/inventory.yml playbooks/site.yml \
  -e cluster_group=k3s_cluster1 -e server_group=server1 -e agent_group=agent1

# 4. Argo CD Ingress (not managed from Git)
kubectl --context k3s-cluster1 -n argocd patch ingress argocd-server --type json -p '[
  {"op":"replace","path":"/spec/rules/0/host","value":"argocd.10.0.0.125.nip.io"},
  {"op":"replace","path":"/spec/tls/0/hosts/0","value":"argocd.10.0.0.125.nip.io"}]'

# 5. Monitoring hostnames: replace the IP in homelab-gitops, commit, push
cd ~/DEV/homelab-gitops
sed -i 's/10\.0\.0\.13\b/10.0.0.125/g' clusters/k3s-cluster1/apps/*.yaml README.md
git commit -am "Move cluster1 hostnames to new control-plane IP" && git push
```

cert-manager issued new certificates for the new hostnames automatically.
In step 3, legolas reported a failed restart (a timeout while k3s was still
starting), but it joined a few seconds later.

**To avoid this:** set a DHCP reservation on the router for each server
(gandalf `10.0.0.125`, shire `10.0.0.233`), or better, for every Pi.

---

## Part 5: Pi-hole on Proxmox

> **Status: planned, not deployed yet.** `10.0.0.53` below is the
> intended address; pick any free IP outside the router's DHCP range.

Pi-hole runs in an unprivileged Debian LXC container on Proxmox, not in
k3s. DNS for the whole LAN shouldn't depend on the clusters, and gandalf's
IP change (Part 4) shows what happens when a core service's address moves.

### 5.1 Create the container

Either use the community script from the Proxmox node's Shell (choose
**Advanced** to set a static IP; read the script first):

```bash
bash -c "$(curl -fsSL https://raw.githubusercontent.com/community-scripts/ProxmoxVE/main/ct/pihole.sh)"
```

Or by hand. Download a Debian template on the node:

```bash
pveam update
pveam available | grep debian
pveam download local debian-12-standard_<version>_amd64.tar.zst   # exact name from the list
```

Then **Create CT** in the web UI:

| Setting | Value |
|---|---|
| Unprivileged | yes |
| Template | Debian 12 (or 13) |
| Disk / CPU / RAM | 8 GB / 1 core / 512 MB |
| Network | `vmbr0`, static `10.0.0.53/24`, gateway = the router |
| DNS (during install) | `1.1.1.1` |
| Options | **Start at boot** on |

### 5.2 Install Pi-hole

In the container's console:

```bash
apt update && apt upgrade -y
apt install -y curl
curl -sSL https://install.pi-hole.net | bash   # defaults: eth0, pick an upstream
pihole setpassword
```

The admin UI is at `http://10.0.0.53/admin`.

### 5.3 Point the LAN at it

- In the router's DHCP settings, set `10.0.0.53` as the **only** DNS
  server. A secondary public resolver (e.g. `8.8.8.8`) lets clients bypass
  blocking.
- Alternatively, turn off the router's DHCP and enable Pi-hole's
  (*Settings → DHCP*). That also gives one place for the server
  reservations from Part 4.

Check from the control node:

```bash
nslookup doubleclick.net 10.0.0.53   # should return 0.0.0.0
nslookup github.com 10.0.0.53        # should resolve normally
```

### 5.4 Local names instead of nip.io (optional)

Pi-hole can serve local hostnames, so the Ingresses no longer need to embed
gandalf's IP:

- Single names: *Settings → Local DNS Records*, e.g. `argocd.home` →
  `10.0.0.125`.
- Wildcard for all cluster1 services (Pi-hole v6): *Settings → All
  settings → `misc.dnsmasq_lines`*, add `address=/home/10.0.0.125`.

Switching to these names means updating the Argo CD Ingress and the
homelab-gitops hostnames (same places as Part 4, steps 4–5), and reissuing
the certificates with the new names.

---

## Open items

- [ ] DHCP reservations for the servers, so their IPs can't change again.
      If one does change, follow Part 4 for that cluster (inventory,
      kubeconfig, rerun `site.yml`).
- [ ] Encrypt the cluster tokens with ansible-vault.
- [ ] Commit the playbook changes and `docs/` in k3s-ansible, and the
      inventory changes in ansible-inventory.
- [ ] Optionally rename the `p2`/`p3` hostnames on sam and merry.
- [ ] Monitoring for cluster2: install Argo CD there, or register cluster2
      in cluster1's Argo CD (`argocd cluster add k3s-cluster2`), and add a
      `clusters/k3s-cluster2/` tree.
- [ ] Deploy Pi-hole on Proxmox (Part 5) and set it as the LAN's DNS.
- [ ] Optionally move the cluster1 hostnames from `*.10.0.0.125.nip.io`
      to local `*.home` names served by Pi-hole.
