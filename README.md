# openbao-helm

ADT umbrella (wrapper) chart for [OpenBao](https://openbao.org), following the
same pattern as `step-ca-helm`: it wraps the upstream
[`openbao/openbao`](https://github.com/openbao/openbao-helm) chart and adds the
bits that chart doesn't ship.

Deployed by Argo CD (`openbao-dev` Application in `adt-grc-gitops`) with
`releaseName: openbao` and per-environment values from
`environments/openbao/dev/values.yaml`.

## What this chart adds on top of the upstream chart

- **Auto-init + self-unseal.** A run-once bootstrap `Job` (`openbao-bootstrap`)
  execs into the server pod, runs `bao operator init` with a single unseal
  share, and stores the unseal key + root token in the `openbao-keys` Secret,
  then unseals. On pod restart a `postStart` lifecycle hook re-unseals from the
  same Secret (mounted optionally so first boot works before the Secret exists).
- **Bootstrap RBAC.** A ServiceAccount + namespaced Role/RoleBinding letting the
  Job exec into the pod and manage the `openbao-keys` Secret.

## Deployment shape (dev / single-node CRC)

- Standalone server, **file storage** on a 5Gi PVC
  (`crc-csi-hostpath-provisioner`) — not Raft/HA.
- Plain HTTP listener in-cluster; TLS terminated at the OpenShift **Route**
  (edge).
- Agent **injector disabled**, CSI provider disabled, `authDelegator` disabled
  (no Kubernetes auth method yet).
- UI enabled.

## Security note

The dev config uses a **single unseal share** and keeps the unseal key + root
token **at rest** in a Kubernetes Secret so the vault can self-unseal
unattended. This is a throwaway TEST posture. Do **not** reuse it for a
production vault — use multiple shares / a proper auto-unseal seal and keep the
root token offline.

## Updating the pinned OpenBao version

```sh
# edit dependencies.version in Chart.yaml, then re-vendor the subchart:
helm dependency update
git add Chart.lock charts/*.tgz
```
