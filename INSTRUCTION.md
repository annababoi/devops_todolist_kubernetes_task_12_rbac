# Task 12 — Kubernetes RBAC: Validation Instructions

## Prerequisites

Install Docker Desktop, `kind`, `kubectl`, and `curl`. Run commands from the repository root. The examples use a kind cluster named `practice` and a Django application in the `todoapp` namespace.

**Security note:** Listing Secrets is a deliberately privileged operation for this educational exercise. Use a dedicated ServiceAccount and a namespace-scoped Role. Do not paste ServiceAccount tokens, Secret values, or decoded credentials into a Pull Request.

## 1. Create or select the Kubernetes cluster

If the `practice` cluster does not already exist, create it using the repository's configuration:

```bash
kind create cluster --name practice --config cluster.yml
```

If a `practice` cluster already exists, skip creation. Select it and check the nodes:

```bash
kubectl config use-context kind-practice
kubectl get nodes
```

All nodes should eventually report `Ready`. If a different cluster name is used, select the corresponding `kind-<cluster-name>` context.

## 2. Deploy the application and RBAC resources

The repository's `bootstrap.sh` deploys MySQL, applies the `todoapp` namespace, creates RBAC resources from `security/rbac.yml` **before** creating the Django Deployment, and then deploys the Django application.

```bash
bash bootstrap.sh
```

Wait for the workload to become ready:

```bash
kubectl get pods -n mysql -o wide
kubectl get pods -n todoapp -o wide
kubectl rollout status deployment/todoapp -n todoapp --timeout=300s
```

The ToDo application Pod should become `1/1 Running`. MySQL must be ready for Django's readiness endpoint to succeed.

## 3. Verify ServiceAccount, Role, and RoleBinding

The `security/rbac.yml` manifest contains three resources in the `todoapp` namespace:

- `ServiceAccount` named `secrets-lister`.
- `Role` named `secrets-lister`, granting the `list` verb for the core API resource `secrets`.
- `RoleBinding` named `secrets-lister-binding`, linking that Role to the ServiceAccount.

Check that Kubernetes created them:

```bash
kubectl get serviceaccount secrets-lister -n todoapp
kubectl get role secrets-lister -n todoapp
kubectl get rolebinding secrets-lister-binding -n todoapp
kubectl describe role secrets-lister -n todoapp
kubectl describe rolebinding secrets-lister-binding -n todoapp
```

Verify the effective permissions:

```bash
kubectl auth can-i list secrets -n todoapp \
  --as=system:serviceaccount:todoapp:secrets-lister

kubectl auth can-i delete secrets -n todoapp \
  --as=system:serviceaccount:todoapp:secrets-lister
```

Expected results: **`yes`** for `list` and **`no`** for `delete`, assuming no additional permission grants exist.

## 4. Verify that the Deployment uses the ServiceAccount

The Django Deployment must include `serviceAccountName: secrets-lister` under `spec.template.spec`.

```bash
kubectl get deployment todoapp -n todoapp \
  -o jsonpath='{.spec.template.spec.serviceAccountName}{"\n"}'

kubectl get pods -n todoapp \
  -o custom-columns=NAME:.metadata.name,SERVICEACCOUNT:.spec.serviceAccountName
```

Both commands should identify `secrets-lister` for the Django workload.

## 5. List Secrets from inside the Django Pod using curl

The following command runs **inside** a Pod selected by the Django Deployment. It authenticates to the Kubernetes API server using the Pod's mounted ServiceAccount token and cluster CA certificate. It prints the HTTP status and **only the names** of returned Secrets, without exposing their values.

```bash
kubectl exec -n todoapp deployment/todoapp -- sh -c '
  SA=/var/run/secrets/kubernetes.io/serviceaccount
  TOKEN=$(cat "$SA/token")
  curl -sS --cacert "$SA/ca.crt" \
    -H "Authorization: Bearer $TOKEN" \
    -o /tmp/rbac-secrets-response.json \
    -w "HTTP status: %{http_code}\n" \
    https://kubernetes.default.svc/api/v1/namespaces/todoapp/secrets
  python -c "import json; data=json.load(open(\"/tmp/rbac-secrets-response.json\")); print(\"Secret names:\", [item[\"metadata\"][\"name\"] for item in data.get(\"items\", [])])"
  rm -f /tmp/rbac-secrets-response.json
'
```

Expected output resembles:

```text
HTTP status: 200
Secret names: ['app-secret']
```

The list of names may differ; `HTTP status: 200` and the returned list are the relevant evidence. This command assumes the Django image includes `curl` and Python. If `curl` is unavailable inside the image, install or use an appropriate curl-capable test container with the same ServiceAccount, and document that difference.

**Screenshot for the PR:** Capture the terminal showing the in-Pod `kubectl exec ... curl` command and its successful output. Ensure no token, password, Secret `data`, or other sensitive values are visible. Attach the screenshot to the Pull Request.

## 6. Troubleshooting

If the API returns `403 Forbidden`, verify that the ServiceAccount, Role, and RoleBinding are all in `todoapp`, that the Role specifies `resources: ["secrets"]` and `verbs: ["list"]`, and that `RoleBinding.subjects` refers to `secrets-lister`.

```bash
kubectl describe role secrets-lister -n todoapp
kubectl describe rolebinding secrets-lister-binding -n todoapp
kubectl auth can-i list secrets -n todoapp \
  --as=system:serviceaccount:todoapp:secrets-lister
```

If the Pod fails to start, inspect the Pod events and confirm that its ServiceAccount exists:

```bash
kubectl get pods -n todoapp
kubectl describe deployment todoapp -n todoapp
kubectl get serviceaccounts -n todoapp
kubectl logs -n todoapp deployment/todoapp --tail=50
```

If `kubectl` reports `localhost:8080 connection refused`, confirm that the kind cluster exists and select its context with `kubectl config use-context kind-practice`.

## 7. Submission

Make sure the Pull Request includes:

- `security/rbac.yml` with the ServiceAccount, Role, and RoleBinding.
- The updated Django Deployment manifest referencing `secrets-lister`.
- The updated `bootstrap.sh`, applying RBAC before the Django Deployment.
- This `INSTRUCTION.md`.
- A screenshot of the successful in-Pod `curl` check, showing the HTTP status and Secret names without Secret values.

Review changes before committing. Do not stage IDE directories such as `.idea/` or local credentials.

```bash
git status
git add security/rbac.yml .infrastructure/app/deployment.yml bootstrap.sh INSTRUCTION.md
git diff --cached --stat
git commit -m "Configure namespace-scoped RBAC for Django ServiceAccount"
git push -u origin develop
```

Open a Pull Request from `develop` to the repository's intended base branch, attach the screenshot, and submit the PR URL on the Mate Academy platform.
