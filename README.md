# OpenShift-on-OpenShift Virt Provisioning

Three roles, one playbook, per-cluster vars in `group_vars/<cluster_name>.yml`.

## Layout

```
group_vars/
  all.yml        # shared: management_kubeconfig, openshift_clusters_dir, pull secret path
  dev.yml        # per-cluster: node list (name/role/mac/ip), sizing, network
roles/
  provision_vms/       # creates namespace, blank boot disks, stopped VMs
  generate_agent_iso/  # renders install-config/agent-config, builds+uploads the agent ISO
  install_cluster/     # starts VMs, waits for bootstrap/install, saves kubeconfig
site.yml
```

## Output layout

Everything for a cluster lives under `openshift_clusters_dir` (default
`/root/openshift/clusters`, set in `group_vars/all.yml`):

```
/root/openshift/clusters/dev/
  credentials/
    id_rsa, id_rsa.pub     # generated per-cluster; public key goes into install-config.yaml's sshKey
    kubeconfig              # copy of the cluster's kubeconfig
  output/
    install-config.yaml
    agent-config.yaml
    agent.x86_64.iso
    auth/
      kubeconfig             # written natively by openshift-install
      kubeadmin-password
      id_rsa, id_rsa.pub     # copied here too, alongside the kubeconfig
```

`credentials/` is meant as the one place to find a cluster's SSH key and
kubeconfig; `output/` is openshift-install's own work dir (also useful for
debugging a failed install via `.openshift_install.log` there).

## Prerequisites

- `ansible-galaxy collection install -r requirements.yml`
- `openshift-install` and `virtctl` binaries on PATH (matching `openshift_version` in group_vars/all.yml)
- A Multus `NetworkAttachmentDefinition` already created in the target namespace matching `network_attachment`
- `PULL_SECRET_PATH` env var (or edit `pull_secret_path` in group_vars/all.yml)
- `KUBECONFIG` pointed at the management (hub) cluster running OpenShift Virt
- Each bare metal node that should host a control-plane VM manually labeled:
  ```bash
  oc label node <bare-metal-node-1> virt_control_plane_number=1
  oc label node <bare-metal-node-2> virt_control_plane_number=2
  oc label node <bare-metal-node-3> virt_control_plane_number=3
  ```
- A local (LVM-backed) StorageClass already set up on those same nodes
  (e.g. via the LVM Storage / TopoLVM operator), name matching
  `etcd_storage_class` in group_vars

## Control-plane pinning

Any node entry with `control_plane_number` set gets:
- a `nodeSelector: virt_control_plane_number: "<N>"` on the VM, so it schedules onto the matching labeled bare metal node
- its boot DataVolume provisioned from `etcd_storage_class` (RWO) instead of `default_storage_class` (RWX)

This keeps the VM and the local PV backing its disk on the same physical
node — required since local storage can't follow a live-migrated VM.
`provision_vms` fails fast if a `control_plane_number` has no matching
labeled node before creating anything. Worker nodes should omit
`control_plane_number` and use `default_storage_class`.

## Run

```bash
ansible-playbook site.yml -e cluster_name=dev
```

Add a new cluster by copying `group_vars/dev.yml` to `group_vars/<name>.yml` and
adjusting the node list, sizing, and network block.

Run a single phase with `--tags vms`, `--tags iso`, or `--tags install`.

## Notes / things to adapt for your environment

- VM sizing (`vm_cpu_cores`, `vm_memory`, `vm_disk_size`) and node MAC/IP
  addresses in `group_vars/<cluster_name>.yml` are examples — set real values
  for your bridge/VLAN.
- `agent-config.yaml.j2` assumes static IPs via NMState, which is the
  realistic case for OpenShift Virt bridged networking. Switch to DHCP by
  dropping the `networkConfig` block per host if your `network_attachment`
  hands out addresses.
- `install_cluster`'s wait steps block for the full bootstrap+install
  duration (commonly 30–60+ min) since `openshift-install agent wait-for`
  is itself blocking; there's nothing to poll around that.
- The DataVolume `accessModes: ReadWriteMany` assumes a storage class that
  supports it (needed since KubeVirt live-migrates by re-attaching); use
  `ReadWriteOnce` if you don't need migration and your storage class prefers it.
