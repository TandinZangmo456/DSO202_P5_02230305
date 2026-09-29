# DSO202 Practical: Environment-Specific Configuration with Kustomize on Kind

**Name:** ____________  **Student ID:** ____________  **Date:** 29 September 2026

## 1. Aim

Deploy one NGINX app to dev, staging and prod from a single base using Kustomize overlays, without copying the Deployment or Service, and follow the workflow render → diff → apply → verify.

## 2. Environment (Task 0)

Context `kind-dso202-a2`, Kubernetes v1.36.1, kubectl v1.34.1 with Kustomize v5.7.1. The cluster had **one control-plane node** (Ready) instead of the lab's three nodes, so all pods ran on that node. This does not change what Kustomize renders.

![Figure 1](images/fig01.png)

*Figure 1: Pre-flight checks.*

## 3. Repository and base (Tasks 1–2)

```text
examples/webapp/
├── base/      deployment.yaml, service.yaml, index.html, kustomization.yaml
└── overlays/
    ├── dev/      index.html, kustomization.yaml, namespace.yaml
    ├── staging/  index.html, kustomization.yaml, namespace.yaml
    └── prod/     index.html, kustomization.yaml, namespace.yaml, patch-resources.yaml
```

![Figure 2](images/fig02.png)

*Figure 2: Repository tree and prod patch.*

Only the base holds the Deployment and Service (once). Namespace, replicas, image tag, labels, HTML and prod resources differ and live in the overlays.

The base renders a Deployment, a Service and a ConfigMap named `web-content-587c5ckcm8`. The name has a hash because Kustomize appends a hash of the content to generated ConfigMaps and rewrites the Deployment's volume reference to match.

![Figure 3](images/fig03.png)

*Figure 3: Base render filtered: kinds and the hashed ConfigMap name in both the ConfigMap and the Deployment volume.*

## 4. Dev vs prod (Task 3)

![Figure 4](images/fig04.png)

*Figure 4: `diff -u` of rendered dev and prod (excerpt).*

| Category | Dev | Prod |
|---|---|---|
| Namespace | `webapp-dev` | `webapp-prod` |
| Environment label | `dev` (plus `debug: "true"`) | `prod` |
| Replicas | 1 | 4 |
| Image | `nginx:1.27-alpine` (after fix) | `nginx:1.27.0` |
| Resources | none | requests 200m/128Mi, limits 500m/256Mi |
| ConfigMap | `web-content-48fdkc9k5d` | `web-content-45gg4d97fm` |
| Annotation | none | `training.example.com/tier: production` |

The overlays explain every difference in a few lines, and the Service and shared Deployment fields do not appear in the diff.

## 5. Deploying dev (Tasks 4–5)

```bash
kubectl kustomize examples/webapp/overlays/dev
kubectl diff -k examples/webapp/overlays/dev || true
kubectl apply -k examples/webapp/overlays/dev
kubectl get all -n webapp-dev
```

The first `diff` reported the namespace as not found, which is expected before the first apply. `curl` through `kubectl port-forward` returned the dev page ("DEV, experimental build").

![Figure 5](images/fig05.png)

*Figure 5: Rendered dev overlay (excerpt).*

![Figure 6](images/fig06.png)

*Figure 6: `kubectl get all -n webapp-dev`: pod 1/1 Running, Deployment 1/1.*

## 6. ConfigMap hash and rollout chain (Task 6)

After editing `overlays/dev/index.html` to "DEV v2 — configuration changed":

| | Before | After |
|---|---|---|
| ConfigMap | `web-content-48fdkc9k5d` | `web-content-5ghd46c4mb` |
| Pod | `webapp-57c94f9dd9-8q2fh` | `webapp-99686859d-sx2t9` |

![Figure 7](images/fig07.png)

*Figure 7: Render with new hash, apply, rollout, and the after state.*

**Chain:** file content changed → generated ConfigMap content changed → ConfigMap name hash changed → Deployment reference changed → pod template changed → rollout occurred. The old ConfigMap remained because `apply` does not prune.

## 7. Staging and prod, and prod patch (Tasks 7–8)

Final replicas: dev 1, staging 2, prod 4 (all available).

![Figure 8](images/fig08.png)

*Figure 8: Deployments in all three namespaces, the prod patch and the rendered `resources` block.*

- **Merged or deleted?** Merged: the base had no `resources`, the patch adds it, and all other container fields are unchanged.
- **Who owns the prod policy?** The prod overlay (`patch-resources.yaml`).
- **Why a patch, not a copy?** The patch holds only the difference; a copied Deployment would drift whenever the base changes.

## 8. QA overlay (Task 9)

QA reuses the base, with namespace `webapp-qa`, 2 replicas, label `qa`, its own `index.html`, and this JSON 6902 patch:

```yaml
- op: add
  path: /metadata/annotations
  value: {}
- op: add
  path: /metadata/annotations/training.example.com~1owner
  value: qa-team
```

The first operation creates the annotations map (the base has none, and a JSON 6902 `add` fails without the parent). `~1` escapes the `/` in the key. The annotation was checked in the rendered output before applying.

![Figure 9](images/fig09.png)

*Figure 9: QA files, render with annotation, apply and verification (2 pods Running).*

## 9. Cleanup (Task 10)

All four namespaces were deleted with `kubectl delete -k`, and `kubectl get ns | grep 'webapp-'` returned nothing.

![Figure 10](images/fig10.png)

*Figure 10: Environments before deletion, delete commands and empty check.*

## 10. Challenge: `namePrefix: sbx-`

Rendered only, not applied. Predictions matched except one: the Deployment, Service and ConfigMap were prefixed and the Deployment's ConfigMap reference was rewritten automatically (`sbx-web-content-587c5ckcm8`). The Namespace was **not** prefixed (`webapp-sandbox`), because Kustomize excludes Namespace objects from `namePrefix`, so my prediction was wrong. The volume name and selectors stayed unchanged.

![Figure 11](images/fig11.png)

*Figure 11: Sandbox overlay and rendered names.*

## 11. Strategic merge vs JSON 6902

| | Strategic merge (prod) | JSON 6902 (QA) |
|---|---|---|
| Form | Kubernetes-shaped fragment | List of `op` / `path` / `value` |
| Matching | By kind and name in the patch | `target:` in `kustomization.yaml` |
| Best for | Readable nested changes such as container resources | Exact paths, `remove`, precise edits |
| Gotcha | Less precise | Parent must exist; `/` in keys becomes `~1` |

Strategic merge speaks Kubernetes; JSON 6902 speaks paths.

## 12. Reflection

The dev overlay had `newTag: 1.27-alpine-experimental` and staging had `1.27-alpine-rc1`, neither of which exists in the registry. Dev was caught before applying, but staging went to `ErrImagePull`. `kubectl describe pod` showed the image name, and `kubectl kustomize ... | grep 'image:'` showed the overlay generated it. I changed `newTag` to `1.27-alpine` and re-applied, and Kubernetes replaced the failing pods. Rendered output shows exactly what will be sent, but cannot show whether an image tag exists, so tags must be checked separately.

## 13. Conclusion

One base and four thin overlays (dev, staging, prod, qa) gave four working environments with no copied Deployment or Service. Rendering first exposed the differences and helped diagnose the image-tag fault, and the hash chain showed how a content change becomes a rollout.