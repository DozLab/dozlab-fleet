# dozlab-fleet

What Flux runs on DozLab's clusters. GitHub Actions in each app repo builds, tests and pushes images to
GHCR; it never boots a lab, because the hosted runners have no KVM for Firecracker. Flux runs on the
node, deploys the `dozlab` Helm chart and rolls out new images by committing their tags here.
Why: dozlab-api `docs/decision.md`, "CI on GitHub Actions, CD with Flux and the dozlab Helm chart".

```
clusters/local/                 the single-node k3s cluster (kubeconfig ~/.kube/dozlab-local.yaml)
  flux-system/                  written by `flux bootstrap`; don't edit
  dozlab/
    chart.yaml                  OCIRepository: oci://ghcr.io/dozlab/charts/dozlab, semver >=0.4.0 <1.0.0
    helmrelease.yaml            HelmRelease `dozlab` in dozlab-system: the controller, the API (its migration
                                hook, the /api Ingress), RBAC, the CRD; after dozlab-secrets
    images.yaml                 ImageRepository + ImagePolicy per image (newest main-<sha7>-<run>)
    image-automation.yaml       commits the tags those policies pick into helmrelease.yaml
  secrets/
    external-secrets.yaml       HelmRelease: External Secrets Operator 2.12.0 (upstream chart)
    dozlab-secrets.yaml         OCIRepository + HelmRelease: dozlab-infra's dozlab-secrets chart, after ESO
```

## How a change reaches the cluster

| Change | What happens |
|---|---|
| Merge to `main` in dozlab-controller, -terminal-sidecar, -infra (Firecracker) or -rootfs-manager | CI pushes `ghcr.io/dozlab/<image>:main-<sha7>-<run>`. Within ~5 min Flux commits the new tag to `helmrelease.yaml` here ("Roll out new images") and Helm upgrades the release |
| Chart change in dozlab-infra | Merge, then tag `chart-v<version>`. Flux picks up any version in the semver range |
| Values (settings) | Edit `helmrelease.yaml` here through a PR |
| Roll back an image | `git revert` Flux's commit here. The policy will pick the newest tag again on the next push, so fix forward or pin by removing the `$imagepolicy` comment |

Tags and labels are in the decision doc ("Tags and labels"): every image gets `main-<sha7>-<run>` and
`sha-<full sha>` from `main`, and `1.2.3`/`1.2`/`1` from a `v1.2.3` git tag.

The frontend isn't here: it's on GitHub Pages, and calls the API on the Funnel URL, which goes to
Traefik and on to the API through its `/api` Ingress (`~/dozlab-browser.sh status` checks the chain). Postgres and RabbitMQ stay outside Flux for now (`~/dozlab.sh postgres`,
`~/dozlab.sh rabbitmq`); they hold data on PVCs, so if they move here, Flux must never prune them.

## First bring-up (owner; cluster writes)

1. **Images and chart exist.** The CI PRs and dozlab-infra's chart PR are merged, each repo's CI has
   pushed a `main-*` image, and the chart is tagged:
   ```bash
   cd ~/dozlab/dozlab-infra && git fetch && git tag chart-v0.4.0 origin/main && git push origin chart-v0.4.0
   ```
2. **GHCR packages are public**, so the node pulls without a secret: github.com/orgs/DozLab/packages →
   each of `dozlab-controller`, `dozlab-firecracker`, `dozlab-init-vm`, `charts/dozlab`
   → Package settings → Change visibility → Public. Also `charts/dozlab-secrets` after its first
   release. `dozlab-terminal` stays private: lab pods (`controller.labPod.imagePullSecrets`) and
   Flux's `dozlab-terminal` ImageRepository pull it with `ghcr-pull`, which ESO writes (see "Secrets").
3. **The tags in `helmrelease.yaml` are real**: each is the first `main-*` image CI pushed. After
   bootstrap, Flux keeps them current.
4. **Hand the running controller and CRD to the release** (dozlab-infra `helm/dozlab/README.md`,
   "Adopting an existing install"):
   ```bash
   export KUBECONFIG=~/.kube/dozlab-local.yaml
   own() {
     kubectl label "$@" app.kubernetes.io/managed-by=Helm --overwrite
     kubectl annotate "$@" meta.helm.sh/release-name=dozlab meta.helm.sh/release-namespace=dozlab-system --overwrite
   }
   own crd labsessions.dozlab.io
   own -n dozlab-system deployment dozlab-controller
   own -n dozlab-system service dozlab-controller-metrics
   own -n dozlab-system serviceaccount dozlab-controller
   ```
5. **Bootstrap Flux.** Needs a GitHub token that can add a deploy key to this repo (fine-grained:
   Administration and Contents read/write on `DozLab/dozlab-fleet`). `--read-write-key` lets image
   automation push.
   ```bash
   export GITHUB_TOKEN=<token>
   flux bootstrap github --owner=DozLab --repository=dozlab-fleet --branch=main \
     --path=clusters/local --read-write-key \
     --components-extra=image-reflector-controller,image-automation-controller
   ```
6. **Check**, then remove the old RBAC, which the chart replaced under new names:
   ```bash
   flux get all -A
   kubectl -n dozlab-system get pods -L app.kubernetes.io/version
   kubectl delete clusterrolebinding dozlab-controller-rolebinding
   kubectl delete clusterrole dozlab-controller-role
   ```

## Secrets

Values live in the `dozlab-secrets` namespace; External Secrets Operator copies them where they're
needed (Kubernetes provider, ClusterSecretStore `dozlab`). Both are Helm releases in `secrets/`. A
source Secret is created empty by the chart and filled once by hand, since a value can't be in git:

```bash
kubectl -n dozlab-secrets patch secret ghcr --type merge \
  -p "{\"stringData\":{\"username\":\"<github user>\",\"token\":\"$(cat ~/.dozlab-ghcr-token)\"}}"
kubectl get externalsecrets -A           # SecretSynced = the copy is current
```

Adding one: a value in the chart's `values.yaml`/templates (dozlab-infra `helm/dozlab-secrets`),
a chart release, then fill it as above.

## Day to day

```bash
flux get helmreleases -A                 # is the release Ready, which chart version
flux get images all -A                   # newest tag each policy sees
flux reconcile source git flux-system    # pull this repo now instead of waiting
flux logs --kind=HelmRelease --name=dozlab -n dozlab-system
```
