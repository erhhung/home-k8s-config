# Erhhung's Home Kubernetes Cluster Configuration

This project configures and provisions Erhhung's high-availability Kubernetes cluster at home using Rancher.

The top-level Ansible playbook `main.yml` run by `play.sh` will provision 5 VM hosts (`rancher` and `k8s1`..`k8s4`)
in the XCP-ng "Home" pool, all running Ubuntu Server 24.04 Minimal without customizations besides basic networking
and authorized SSH key for user `erhhung`.

A single-node K3s Kubernetes cluster will be installed on host `rancher` along with Rancher Server on that cluster.
A 4-node RKE2 Kubernetes cluster with a high-availability control plane on a virtual IP will be installed on hosts
`k8s1`..`k8s4`. The Longhorn distributed block storage engine will also be installed on the RKE2 cluster to manage
a pool of LVM logical volumes created on each cluster node. Finally, one or more virtual clusters running K0s will
be created for testing and learning purposes.

The Rancher Server UI at https://rancher.fourteeners.local will be provisioned with a TLS certificate from Erhhung's
private CA server at pki.fourteeners.local.

## Cluster Topology

<p align="center">
<img src="images/topology.drawio.svg" alt="topology.drawio.svg" />
</p>

## Cluster Services

<p align="center">
<img src="images/services.drawio.svg" alt="services.drawio.svg" />
</p>

### Service Endpoints

|                       Service URL | Description
|----------------------------------:|:----------------------
| https://rancher.fourteeners.local | Rancher Server console
|  https://harbor.fourteeners.local | Harbor OCI registry
|     https://sso.fourteeners.local | Keycloak IAM console
|        postgres.fourteeners.local | PostgreSQL via Pgpool _(mTLS only)_
|   https://kiali.fourteeners.local | Kiali dashboard
| https://grafana.fourteeners.local | Grafana dashboard
|  https://argocd.fourteeners.local | Argo CD console

## Ansible Vault

The Ansible Vault password is stored in macOS Keychain under item "`Home-K8s`" for account "`ansible-vault`".

```bash
export ANSIBLE_CONFIG=./ansible.cfg
VAULTFILE="group_vars/all/vault.yml"

ansible-vault create $VAULTFILE
ansible-vault edit   $VAULTFILE
ansible-vault view   $VAULTFILE
```

Some variables stored in Ansible Vault _(there are many more)_:

|   Infrastructure Secrets   |    User Passwords
|:--------------------------:|:-------------------:
| `ansible_become_pass`      | `rancher_admin_pass`
| `github_access_token`      | `harbor_admin_pass`
| `age_secret_key`           | `keycloak_admin_pass`
| `k3s_token`                | `grafana_admin_pass`
| `rke2_token`               | `argocd_admin_pass`
| `harbor_secret`            |
| `harbor_ca_key`            |
| `postgresql_pass`          |
| `keycloak_smtp_pass`       |
| `kiali_oidc_client_secret` |

## Connections

All managed hosts are running **Ubuntu 24.04** with SSH key from https://github.com/erhhung.keys already authorized.  

Ansible will authenticate as user `erhhung` using private key "`~/.ssh/erhhung.pem`";  
however, all privileged operations using `sudo` will require the password stored in Vault.

## Playbooks

Set the config variable first for the `ansible-playbook` commands below:

```bash
export ANSIBLE_CONFIG=./ansible.cfg
```

1. Install required packages

    1.1. **Tools** — `emacs`, `jq`, `yq`, `git`, and `helm`  
    1.2. **Python** — Pip packages in user **virtualenv**  
    1.3. **Helm** — Helm plugins: e.g. `helm-diff`

    ```bash
    ansible-playbook packages.yml
    ```

2. Configure system settings

    2.1. **Host** — host name, time zone, and locale  
    2.2. **Network** — DNS servers and search domains  
    2.3. **Login** — customize login MOTD messages  
    2.4. **Certs** — add CA certificates to trust store

    ```bash
    ansible-playbook basics.yml
    ```

3. Set up admin user's home directory

    3.1. **Dot files**: `.bash_aliases`, `.emacs`

    ```bash
    ansible-playbook files.yml
    ```

4. Set up **Rancher Server** on single-node **K3s** cluster
    ```bash
    ansible-playbook rancher.yml
    ```

5. Set up **Kubernetes cluster** with **RKE** on 4 nodes

    Installs **RKE2** with a single control plane node and 3 worker nodes, all permitting workloads,  
    or RKE2 in HA mode with 3 control plane nodes and 1 worker node, all permitting workloads.  
    _In HA mode, the cluster will be accessible thru a virtual IP address courtesy of `kube-vip`._

    ```bash
    ansible-playbook cluster.yml
    ```

6. Set up **Longhorn** dynamic PV provisioner

    6.1. Creates a pool of LVM logical volumes  
    6.2. Installs Longhorn storage components  
    6.3. Installs NFS dynamic PV provisioners

    ```bash
    ansible-playbook storage.yml
    ```

7. Create resources from manifest files

    **IMPORTANT**: Resource manifests must specify the namespaces they wished to be installed  
    into because the playbook simply applies each one without targeting a specific namespace.

    ```bash
    ansible-playbook manifests.yml
    ```

8. Set up **Harbor** private OCI registry
    ```bash
    ansible-playbook harbor.yml
    ```

9. Set up **PostgreSQL** database in _**HA**_ mode

    9.1. Runs initialization SQL script to create roles and databases for downstream applications

    ```bash
    ansible-playbook postgresql.yml
    ```

10. Set up **Keycloak** IAM & OIDC provider

    10.1. Bootstraps PostgreSQL database with `Homelab` realm, user `erhhung`, and OIDC clients

    ```bash
    ansible-playbook keycloak.yml
    ```

11. Set up **Istio** service mesh in _**ambient**_ mode
    ```bash
    ansible-playbook istio.yml
    ```

12. Set up **Argo CD** GitOps delivery tool
    ```bash
    ansible-playbook argocd.yml
    ```

13. Create **virtual clusters** in RKE running **K0s**
    ```bash
    ansible-playbook vclusters.yml
    ```

Alternatively, **run all playbooks** automatically in order:

```bash
# pass options like -v and --step
./play.sh [ansible-playbook-opts]
```

Output from `play.sh` will be logged in "`ansible.log`".

### Optional Playbooks

1. Shut down all/specific VMs

    ```bash
    ansible-playbook shutdownvms.yml [-e targets={group|host|,...}]
    ```

2. Create/revert/delete VM snapshots

    2.1. Create new snaphots

    ```bash
    ansible-playbook snapshotvms.yml [-e targets={group|host|,...}] \
                                      -e '{"desc":"text description"}'
    ```

    2.2. Revert to snapshots

    ```bash
    ansible-playbook snapshotvms.yml  -e do=revert \
                                     [-e targets={group|host|,...}]  \
                                      -e '{"desc":"text to search"}' \
                                     [-e '{"date":"YYYY-mm-dd prefix"}']
    ```

    2.3. Delete old snaphots

    ```bash
    ansible-playbook snapshotvms.yml  -e do=delete \
                                     [-e targets={group|host|,...}]  \
                                      -e '{"desc":"text to search"}' \
                                      -e '{"date":"YYYY-mm-dd prefix"}'
    ```

3. Restart all/specific VMs

    ```bash
    ansible-playbook startvms.yml [-e targets={group|host|,...}]
    ```

## VM Storage

To expand the VM disk on a cluster node, the VM must be shut down
(attempting to resize the disk from Xen Orchestra will fail with
error: `VDI in use`).

Once the VM disk has been expanded, restart the VM and SSH into
the node to resize the partition and LV.

```bash
$ sudo su

# verify new size
$ lsblk /dev/xvda

# resize partition
$ parted /dev/xvda
) print
Warning: Not all of the space available to /dev/xvda appears to be used...
Fix/Ignore? Fix

) resizepart 3 100%
# confirm new size
) print
) quit

# sync with kernel
$ partprobe

# confirm new size
$ lsblk /dev/xvda3

# resize VG volume
$ pvresize /dev/xvda3
Physical volume "/dev/xvda3" changed
1 physical volume(s) resized...

# confirm new size
$ pvdisplay

# show LV volumes
$ lvdisplay

# grow desired LV
$ lvextend -vrl +90%FREE /dev/ubuntu-vg/ubuntu-lv
Extending logical volume ubuntu-vg/ubuntu-lv to up to...
fsadm: Executing resize2fs /dev/mapper/ubuntu--vg-ubuntu--lv
The filesystem on /dev/mapper/ubuntu--vg-ubuntu--lv is now...

# confirm new size
$ df -h /
```

After expanding all desired disks, run `./diskfree.sh`
to verify available disk space on all cluster nodes.

## Troubleshooting

Ansible's [ad-hoc commands](https://docs.ansible.com/ansible/latest/command_guide/intro_adhoc.html#managing-services) are useful in these scenarios.

1. Restart Kubernetes cluster services on all nodes

    ```bash
    ansible rancher          -m ansible.builtin.service -b -a "name=k3s         state=restarted"
    ansible control_plane_ha -m ansible.builtin.service -b -a "name=rke2-server state=restarted"
    ansible workers_ha       -m ansible.builtin.service -b -a "name=rke2-agent  state=restarted"
    ```

    _**NOTE:** remove `_ha` suffix from target hosts if the RKE cluster was deployed in non-HA mode._

2. All `kube-proxy` static pods on continuous `CrashLoopBackOff`

    This turns out to be a [Linux kernel bug](https://bugs.launchpad.net/ubuntu/+source/linux/+bug/2104282) in `linux-image-6.8.0-56-generic` and above _(discovered on upgrade to `linux-image-6.8.0-57-generic`)_, causing this error in the container logs:
    ```
    ip6tables-restore v1.8.9 (nf_tables): unknown option "--xor-mark"
    ```
    Workaround is to downgrade to an earlier kernel:
    ```bash
    # list installed kernel images
    ansible -v cluster -a 'bash -c "dpkg -l | grep linux-image"'

    # install working kernel image
    ansible -v cluster -b -a 'apt-get install -y linux-image-6.8.0-55-generic'

    # GRUB use working kernel image
    ansible -v cluster -m ansible.builtin.shell -b -a '
        kernel="6.8.0-55-generic"
        lvuuid=$(blkid -s UUID -o value /dev/mapper/ubuntu--vg-ubuntu--lv)
        menuid="gnulinux-advanced-$lvuuid>gnulinux-$kernel-advanced-$lvuuid"
        sed -Ei "s/^(GRUB_DEFAULT=).+$/\\1\"$menuid\"/" /etc/default/grub
        grep GRUB_DEFAULT /etc/default/grub
    '
    # update /boot/grub/grub.cfg
    ansible -v cluster -b -a 'update-grub'

    # reboot nodes, one at a time
    ansible -v cluster -m ansible.builtin.reboot -b -a "post_reboot_delay=120" -f 1

    # confirm working kernel image
    ansible -v cluster -a 'uname -r'

    # remove old backup kernels only
    # (keep latest non-working kernel
    # so upgrade won't install again)
    ansible -v cluster -b -a 'apt-get autoremove -y --purge'
    ```

## Roadmap

These are additional components to be deployed:

- [X] [NFS Dynamic Provisioners](https://computingforgeeks.com/configure-nfs-as-kubernetes-persistent-volume-storage/) — create PVs on NFS shares
    * Install on K3s and RKE clusters using [`nfs-subdir-external-provisioner`](https://github.com/kubernetes-sigs/nfs-subdir-external-provisioner/) Helm chart
- [X] [Harbor Container Registry](https://goharbor.io/) — private container registry
    * Install into the same K3s cluster as Rancher Server using [`harbor/harbor`](https://github.com/goharbor/harbor-helm/) Helm chart
- [X] [Argo CD Declarative GitOps](https://argo-cd.readthedocs.io/) — manage deployment of other applications in the main RKE cluster
- [X] [PostgreSQL Database](https://www.postgresql.org/docs/current/) — install using Bitnami's [`postgresql-ha`](https://github.com/bitnami/charts/tree/main/bitnami/postgresql-ha) Helm chart
- [ ] [Prometheus Monitoring Stack](https://github.com/prometheus-operator/kube-prometheus) — Prometheus, Grafana, and rules using the Prometheus Operator
    * Install into the main RKE cluster using [`kube-prometheus-stack`](https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack/README.md) Helm chart
- [X] [Keycloak IAM & OIDC Provider](https://www.keycloak.org/) — identity and access management and OpenID Connect provider
- [X] [Istio Service Mesh](https://istio.io/latest/about/service-mesh/) with [Kiali UI](https://kiali.io/) — secure, observe, trace, and route traffic between cluster workloads
    * Install into the main RKE cluster using [`istioctl`](https://istio.io/latest/docs/ambient/install/istioctl/)
    * Install Kiali using [`kiali-operator`](https://kiali.io/docs/installation/installation-guide/install-with-helm/#install-with-operator/) Helm chart
- [ ] [OpenTelemetry Collector](https://opentelemetry.io/docs/collector/) with [Jaeger UI](https://www.jaegertracing.io/) -- telemetry collector agent and distributed tracing backend
    * Install into the main RKE cluster using [OpenTelemetry Collector](https://opentelemetry.io/docs/platforms/kubernetes/helm/collector/) Helm chart
    * Install Jaeger using [Jaeger](https://github.com/jaegertracing/helm-charts/tree/main/charts/jaeger) Helm chart
- [ ] [MinIO Object Storage](https://github.com/minio/minio) — object storage server and console
    * Install into the main RKE cluster using [MinIO Operator](https://min.io/docs/minio/kubernetes/upstream/operations/installation.html)
- [ ] [Velero Backup & Restore](https://velero.io/docs/latest/basic-install) — back up and restore persistent volumes
    * Install into the main RKE cluster using [Velero](https://github.com/vmware-tanzu/helm-charts/tree/main/charts/velero) Helm chart
- [ ] [Backstage Developer Portal](https://backstage.io/) — software catalog and developer portal
- [ ] [Meshery](https://github.com/meshery/meshery) — visual and collaborative GitOps platform
- [ ] [NATS](https://docs.nats.io/) — high performance message queues (Kafka alternative) with [JetStream](https://docs.nats.io/nats-concepts/jetstream) for persistence
- [ ] [KEDA](https://keda.sh/) — Kubernetes Event Driven Autoscaler
