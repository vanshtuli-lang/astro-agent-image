# How to Set Up Astro Remote Execution with Azure

This guide walks through standing up Astro Remote Execution end-to-end on Azure: AKS with Workload Identity, Key Vault for secrets, Blob Storage for XCom, and a Helm-deployed agent — all authenticated without any stored credentials.

Most of Part A is done in the Azure Portal UI. Parts B and C require a terminal (Azure CLI, kubectl, Helm, Docker).

⚠️ **Follow these steps in order.** Several steps depend on values (IDs, tokens, names) generated in earlier steps.

---

## Part A — Azure Environment Setup

### A1 — Create a Resource Group

1. In the Azure Portal, search **"Resource groups"** → **+ Create**.
2. **Subscription**: select your subscription.
3. **Resource group**: give it a name (e.g. `astro-remote-execution-rg`).
4. **Region**: pick a region close to your users (e.g. `East US`). This region should be used for every other resource in this guide.
5. **Review + create** → **Create**.

---

### A2 — Create the AKS Cluster

1. Search **"Kubernetes services"** → **+ Create** → **Create a Kubernetes cluster**.
2. **Basics tab:**
   - **Resource group**: from A1
   - **Cluster preset configuration**: `Standard` (or `Dev/Test` for a proof-of-concept)
   - **Kubernetes cluster name**: e.g. `astro-remote-execution-aks`
   - **Region**: match your resource group
   - **Kubernetes version**: 1.30 or later
3. **Node pools tab**: default (2 nodes, e.g. `Standard_D4ds_v5`) is fine to start.
4. **Security/Networking tab** — the exact tab name may vary by Portal version. Enable **both**:
   - ✅ **Enable OIDC issuer**
   - ✅ **Enable Microsoft Entra Workload ID**

   🔴 **This is the most important step in Part A.** Everything else — the federated identity, Key Vault access, Blob Storage access — depends on these two toggles.
5. **Review + create** → **Create**. Provisioning takes ~5–10 minutes.

**Double-check before moving on:**
- Open the cluster → **Security configuration** (or **Cluster configuration**, depending on Portal version) → confirm **OIDC issuer** and **Workload Identity** both show **Enabled**.
- Note the **OIDC Issuer URL** somewhere — you may need it in A8.

---

### A3 — Create the Container Registry and Attach It to AKS

1. Search **"Container registries"** → **+ Create**.
2. **Resource group**: your resource group.
3. **Registry name**: globally unique, lowercase letters/numbers only (e.g. `astroremoteexecacr`).
4. **Location**: match your other resources.
5. **SKU**: `Basic` is fine for a proof-of-concept.
6. **Review + create** → **Create**.

**Attach it to AKS:**
- Open your AKS cluster's **Overview** page. Next to **Container registries**, click **Attach a registry**, select your registry from the dropdown, and confirm.
- (Alternative path if that link isn't available: open the registry → **Access control (IAM)** → **+ Add role assignment** → **AcrPull** → **Assign access to: Managed identity** → select **Kubernetes service** → pick your AKS cluster's kubelet identity → **Review + assign**.)

**Double-check:** on the AKS cluster's Overview page, confirm **Container registries** no longer says "Attach a registry" and instead lists your registry's name.

---

### A4 — Create the Storage Account and Blob Container for XCom

1. Search **"Storage accounts"** → **+ Create**.
2. **Resource group**: your resource group.
3. **Storage account name**: globally unique, lowercase, no dashes (e.g. `astroremoteexecxcom`).
4. **Region**: match your other resources. **Performance**: Standard. **Redundancy**: LRS is fine for a proof-of-concept.
5. **Review + create** → **Create**.

**Create the container:**
1. Open the storage account → **Data storage → Containers** → **+ Container**.
2. **Name**: `xcom`. **Public access level**: Private.
3. **Create**.

---

### A5 — Create the User-Assigned Managed Identity

1. Search **"Managed Identities"** → **+ Create**.
2. **Resource group**: your resource group.
3. **Region**: match your other resources.
4. **Name**: e.g. `astro-remote-execution-identity`.
5. **Review + create** → **Create**.

**Note two values now** — you'll need both repeatedly for the rest of this guide:
- **Client ID** — shown on the identity's **Overview** page.
- **Tenant ID** — search **"Microsoft Entra ID"** in the Portal → **Overview** page → copy the **Tenant ID** field (sometimes labeled **Directory ID**).

🔴 **Double-check:** write both values down somewhere reliable. A mismatched or mistyped Client ID or Tenant ID later in `values.yaml` will cause silent authentication failures against Key Vault and Blob Storage that are hard to diagnose after the fact.

---

### A6 — Grant RBAC Roles to the Managed Identity (Storage)

Grant the role **on the target resource** (the storage account), not on the managed identity's own page.

1. Open the storage account from A4.
2. **Access control (IAM)** → **+ Add** → **Add role assignment**.
3. Search for and select **Storage Blob Data Contributor** → **Next**.
4. **Assign access to**: Managed identity → **+ Select members** → choose the identity from A5.
5. **Review + assign**.

(The Key Vault role assignment comes later, in A9, since the vault doesn't exist yet.)

---

### A7 — Create the Kubernetes Namespace and Service Account

**Create the namespace (Portal):**
1. Open your AKS cluster → **Kubernetes resources** → **Namespaces**.
2. **+ Create** → Name: `remote-execution` → **Add**.

**Create the service account (terminal):**
1. Merge cluster credentials into your local kubeconfig:
   ```bash
   az aks get-credentials --resource-group <your-rg> --name astro-remote-execution-aks
   ```
   Replace `<your-rg>` with your actual resource group name (no angle brackets).
2. Create the service account:
   ```bash
   kubectl create serviceaccount remote-exec-sa -n remote-execution
   ```

---

### A8 — Federate the Managed Identity to the Service Account

This step establishes Workload Identity — it tells Microsoft Entra that tokens presented by `remote-exec-sa`, in the `remote-execution` namespace, on this cluster's OIDC issuer, represent your managed identity.

1. Open the managed identity from A5 → **Federated credentials** → **+ Add credential**.
2. **Federated credential scenario**: Kubernetes accessing Azure resources.
3. Select your AKS cluster (or paste in the OIDC Issuer URL from A2 if prompted directly for it).
4. **Kubernetes namespace**: `remote-execution`
5. **Kubernetes service account name**: `remote-exec-sa`
6. **Name**: e.g. `remote-exec-fic`
7. **Add**.

---

### A9 — Create the Key Vault

**Create the vault:**
1. Search **"Key vaults"** → **+ Create**.
2. **Resource group**: your resource group.
3. **Key vault name**: globally unique, **3–24 characters**, must start with a letter and end with a letter or digit (e.g. `astro-remote-exec-kv`).
4. **Region**: match your other resources.
5. On **Access configuration**, confirm **Permission model** is **Azure role-based access control (RBAC)**.
6. **Review + create** → **Create**.

**Grant yourself write access:**
1. Open the vault → **Access control (IAM)** → **+ Add** → **Add role assignment**.
2. Select **Key Vault Secrets Officer** → **Next**.
3. **Assign access to**: your own user account.
4. **Review + assign**.

**Grant the managed identity read access:**
1. Same **Access control (IAM)** blade → **+ Add** → **Add role assignment**.
2. Select **Key Vault Secrets User** → **Next**.
3. **Assign access to**: Managed identity → the identity from A5.
4. **Review + assign**.

**Add your Airflow connections and variables**, using this naming convention (letters, digits, and dashes only — **no underscores**; use a dash instead):

| Type | Secret name pattern | Example |
|---|---|---|
| Airflow Variable | `airflow-variables-<variable-name>` | `airflow-variables-test-var` |
| Airflow Connection | `airflow-connections-<connection-id>` | `airflow-connections-my-database` |

1. Open the vault → **Objects → Secrets** → **+ Generate/Import**.
2. **Upload options**: Manual. **Name**: per the pattern above. **Secret value**: the actual value.
3. **Create**. Repeat for each secret.

A good first test secret to confirm the whole pipeline later:
- Name: `airflow-variables-test-var`
- Value: `hello-from-keyvault`

---

## Part B — Astro Control Plane Setup

1. In the Astro UI, open your Workspace → **Create Deployment**.
2. **Execution mode**: **Remote Execution**.
3. **Runtime version**: an **Airflow 3.x** version (required for Remote Execution).
4. Complete the wizard.
   - 🔴 **Double-check:** if you set an **Allowed IP Address Range** under Advanced settings, you must include your AKS cluster's outbound public IP, or the agent's heartbeat traffic will be blocked with a `403 Forbidden` later. You can add this now or after Part C once you know the exact IP (see the Troubleshooting note at the end of this doc for how to find it).
5. Once provisioned, open the Deployment → **Remote Agents** tab.
6. Generate a **Deployment API Token**: **API Tokens** → create a token with the **Deployment Admin** role. Name it something descriptive (e.g. `remote-exec-agent-image-pull`). This token is used for image pulls.
7. Generate an **Agent Token**: **Remote Agents** tab → **Tokens** view → **+ Agent Token**. Name it descriptively (e.g. `remote-exec-agent-token`). **Copy it immediately** — it cannot be retrieved again. This token authenticates the running agent to the control plane.
8. Download the pre-filled Helm values file: **Remote Agents** tab → **Agents** view → **Register a Remote Agent** → **Download**.

🔴 **Double-check:** keep clear notes on which token is which — the Deployment API Token is for image pulls; the Agent Token is for the running agent. Mixing them up causes authentication failures that are easy to misdiagnose.

🔴 **Double-check:** if you ever re-create this Deployment, you must re-download the values file and generate fresh tokens from the *new* Deployment — an old Deployment ID or a token generated against a different Deployment will cause a persistent `403 Forbidden` on the agent's heartbeat that looks like a token problem but isn't.

---

## Part C — Build and Deploy the Agent

### C1 — Build the Custom Agent Image

1. Create a project folder containing:
   - `Dockerfile`
   - `packages.txt` (can be empty)
   - `requirements.txt`
   - a `dags/` folder with your DAG files

2. **Dockerfile:**
   ```dockerfile
   FROM images.astronomer.cloud/baseimages/astro-remote-execution-agent:<tag>
   ```
   No `RUN pip install` needed — the base image uses an ONBUILD pattern that installs from `packages.txt` and `requirements.txt` automatically. Find `<tag>` on the **Remote Agents → Agents → Register a Remote Agent** screen, or in the `image:` field of your downloaded `values.yaml`.

   🔴 **Double-check:** make sure this tag's agent version matches the version shown for `sentinel.image` in your `values.yaml`. A mismatch between your custom image's agent version and the Sentinel's version can cause subtle runtime issues.

3. **requirements.txt:**
   ```
   apache-airflow-providers-microsoft-azure
   apache-airflow-providers-common-io
   adlfs
   ```

4. Authenticate Docker to Astronomer's registry (using your **Deployment API Token**):
   ```bash
   docker login images.astronomer.cloud -u cli -p <your-deployment-api-token>
   ```

5. Authenticate Docker to your ACR:
   ```bash
   az acr login --name <your-acr-name>
   ```

6. Build the image — always target `linux/amd64` explicitly (AKS runs amd64; skipping this on Apple Silicon causes `exec format error` at pod startup):
   ```bash
   docker build --platform=linux/amd64 -t <your-acr-name>.azurecr.io/astro-agent:v1 .
   ```

7. Push it:
   ```bash
   docker push <your-acr-name>.azurecr.io/astro-agent:v1
   ```

---

### C2 — Configure values.yaml

Start from the file downloaded in Part B. Edit the following fields. **Do not delete or duplicate any existing top-level key** — several of the errors encountered during this setup came from accidentally creating a second copy of a key (e.g. two `commonEnv:` or two `labels:` blocks) rather than editing the existing one. YAML silently keeps only the last occurrence of a duplicated key, which can wipe out settings without any error message.

**`resourceNamePrefix`** — must be a **non-empty string** (not `~`/null, and not `""`). An empty string produces invalid Kubernetes resource names (e.g. `-sentinel-config`) that fail to install; a null value fails schema validation before that.
```yaml
resourceNamePrefix: "astro-agent"
```

**Core identity and image:**
```yaml
namespace: remote-execution
createNamespace: false
image: <your-acr-name>.azurecr.io/astro-agent:v1
imagePullPolicy: IfNotPresent
```

**Image pull secret** — required even though ACR is attached, because the Sentinel component always pulls its own image directly from Astronomer's registry:
```yaml
imagePullSecretName: astronomer-registry-secret
```

**Agent authentication:**
```yaml
agentTokenSecretName: agent-token-secret
```

**Service accounts** — point every component at your federated KSA from A7/A8, with creation disabled:
```yaml
serviceAccount:
  workers:
    create: false
    name: remote-exec-sa
  dagProcessor:
    create: false
    name: remote-exec-sa
  triggerer:
    create: false
    name: remote-exec-sa
  sentinel:
    create: false
    name: remote-exec-sa
```

**Workload Identity labels and annotations** — these are two separate fields, both currently empty in the downloaded file:
```yaml
labels:
  azure.workload.identity/use: "true"

annotations:
  azure.workload.identity/client-id: "<managed-identity-client-id>"
```
🔴 **Double-check:** this Client ID must be *identical* to the one used in the `commonEnv` entries below. A mismatch between the annotation and the `commonEnv` values causes the pod to be federated for one identity while trying to authenticate as a different one — resulting in Key Vault/Blob Storage auth failures.

**Secrets backend (Key Vault) and XCom backend (Blob Storage)** — set these top-level fields:
```yaml
secretBackend: airflow.providers.microsoft.azure.secrets.key_vault.AzureKeyVaultBackend
xcomBackend: airflow.providers.common.io.xcom.backend.XComObjectStorageBackend
```

**`commonEnv`** — this list already exists in the downloaded file with your Astro org/workspace/deployment IDs pre-filled. **Append** to it; do not replace it or create a second `commonEnv:` key.

```yaml
commonEnv:
  - name: AIRFLOW__SECRETS__BACKEND_KWARGS
    value: >-
      {"vault_url": "https://<key-vault-name>.vault.azure.net/",
      "connections_prefix": "airflow-connections",
      "variables_prefix": "airflow-variables",
      "managed_identity_client_id": "<managed-identity-client-id>",
      "workload_identity_tenant_id": "<tenant-id>"}
  - name: AIRFLOW_CONN_WASB_XCOM
    value: >-
      {"conn_type": "azure", "login": "<storage-account-name>",
      "extra": "{\"anon\": false, \"account_name\": \"<storage-account-name>\",
      \"managed_identity_client_id\": \"<managed-identity-client-id>\",
      \"workload_identity_tenant_id\": \"<tenant-id>\"}"}
  - name: AIRFLOW__COMMON_IO__XCOM_OBJECTSTORAGE_PATH
    value: "abfs://wasb_xcom@xcom/poc"
  - name: AIRFLOW__COMMON_IO__XCOM_OBJECTSTORAGE_THRESHOLD
    value: "0"
  - name: AIRFLOW__COMMON_IO__XCOM_OBJECTSTORAGE_COMPRESSION
    value: "zip"
  - name: ASTRO_ORGANIZATION_ID
    value: "<your-org-id, already in the downloaded file>"
  - name: ASTRO_WORKSPACE_ID
    value: "<your-workspace-id, already in the downloaded file>"
  - name: ASTRO_DEPLOYMENT_ID
    value: "<your-deployment-id, already in the downloaded file>"
  - name: ASTRO_DEPLOYMENT_NAMESPACE
    value: "<your-deployment-namespace, already in the downloaded file>"
```

🔴 **Double-check, this is the single most important verification in the whole file:**
- `ASTRO_DEPLOYMENT_ID` and `ASTRO_DEPLOYMENT_NAMESPACE` must match the **current** Deployment you generated your Agent Token against — not a value carried over from an earlier download. Cross-check them against the `endpoint` field under `openLineage:` further down in the same file (it's auto-generated from your most recent download and contains the correct, current Deployment ID and namespace) — the two should agree.
- The connection ID is derived from the env var name: `AIRFLOW_CONN_WASB_XCOM` → connection ID `wasb_xcom` — keep this in sync with the `abfs://wasb_xcom@xcom/poc` path if you ever rename either.
- Threshold `0` sends every XCom value, however small, to Blob Storage instead of Astro's metadata DB — this is the recommended setting for Remote Execution.

**Create the two required Kubernetes secrets** (these are referenced by name in `values.yaml` but not created by it):
```bash
kubectl create secret docker-registry astronomer-registry-secret \
  --docker-server=images.astronomer.cloud \
  --docker-username=cli \
  --docker-password=<your-deployment-api-token> \
  --namespace=remote-execution

kubectl create secret generic agent-token-secret \
  --namespace=remote-execution \
  --from-literal=token="<your-agent-token>"
```

Verify both exist:
```bash
kubectl get secrets -n remote-execution
```

**If you manually created `remote-exec-sa` in A7** (which you did), Helm needs explicit permission to co-manage it. Run this once, before installing:
```bash
kubectl label serviceaccount remote-exec-sa -n remote-execution \
  app.kubernetes.io/managed-by=Helm --overwrite

kubectl annotate serviceaccount remote-exec-sa -n remote-execution \
  meta.helm.sh/release-name=astro-agent \
  meta.helm.sh/release-namespace=remote-execution --overwrite
```

---

### C3 — Install the Helm Chart

```bash
helm repo add astronomer https://helm.astronomer.io
helm repo update

helm install astro-agent astronomer/astro-remote-execution-agent \
  -f values.yaml \
  --namespace remote-execution
```

You should see `STATUS: deployed`.

🔴 **Double-check:** if you ever need to change `values.yaml` after this point, use `helm upgrade` (not `helm install` again) with the same command structure:
```bash
helm upgrade astro-agent astronomer/astro-remote-execution-agent \
  -f values.yaml \
  --namespace remote-execution
```
Note that `commonEnv` changes only roll the Workers, DAG Processor, and Triggerer pods — **not** the Sentinel pod. If you need the Sentinel to pick up a change, restart it explicitly:
```bash
kubectl rollout restart deployment astro-agent-sentinel -n remote-execution
```

---

### C4 — Verify

**1. Check pod status:**
```bash
kubectl get pods -n remote-execution
```
All pods (worker, dag-processor, triggerer, sentinel) should reach `Running` and `1/1` Ready.

**2. Confirm registration in the Astro UI:**
Deployment → **Remote Agents** tab → **Agents** view. All agents should show status **Healthy** with a recent **Last Heartbeat**.

🔴 **If agents don't appear here and the Sentinel logs show a `403 Forbidden` on the heartbeat** (check with `kubectl logs -n remote-execution deployment/astro-agent-sentinel`), this is almost always the Deployment's **Allowed IP Address Range** blocking your cluster's egress traffic. To fix:

1. Get your AKS cluster's outbound public IP:
   ```bash
   az aks show --resource-group <your-rg> --name astro-remote-execution-aks \
     --query "networkProfile.loadBalancerProfile.effectiveOutboundIPs" -o json
   ```
2. Resolve the returned resource ID to an actual IP address:
   ```bash
   az network public-ip show --ids "<resource-id-from-above>" --query "ipAddress" -o tsv
   ```
3. In the Astro UI, go to Deployment → **Details** (or **Settings** → **Advanced**) → **Allowed IP Address Ranges** → add the IP as a `/32` CIDR, e.g.:
   ```
   135.222.240.167/32
   ```
4. **Verify the IP you added is actually correct** by checking what a pod in your cluster sees as its own outbound IP (Azure's reported "effective outbound IP" and actual egress can occasionally differ):
   ```bash
   kubectl run ip-check --rm -it --restart=Never -n remote-execution \
     --image=curlimages/curl -- curl -s https://api.ipify.org
   ```
   Compare this to what you entered in the allowlist.
5. Allow up to a minute or two for the change to take effect, then recheck the Sentinel logs and the Remote Agents UI.

**3. Prove the Key Vault path works end-to-end.** Add this DAG to your `dags/` folder:

```python
from airflow.decorators import dag, task
from airflow.models import Variable
from datetime import datetime

@dag(schedule=None, start_date=datetime(2024, 1, 1), catchup=False)
def test_keyvault():
    @task
    def read_var():
        print(Variable.get("test_var"))
    read_var()

test_keyvault()
```

Rebuild and push the image, then restart the deployments so the new DAG is picked up:
```bash
docker build --platform=linux/amd64 -t <your-acr-name>.azurecr.io/astro-agent:v1 .
docker push <your-acr-name>.azurecr.io/astro-agent:v1
kubectl rollout restart deployment -n remote-execution
```

Trigger `test_keyvault` from the Astro UI and check the task log for `hello-from-keyvault`. If it appears, the full chain is proven: **Workload Identity → federated token → Key Vault → `AzureKeyVaultBackend` → your DAG.**

---

## Summary Checklist

- [ ] A1 — Resource Group created
- [ ] A2 — AKS cluster created with OIDC issuer + Workload Identity **enabled**
- [ ] A3 — ACR created and attached to AKS
- [ ] A4 — Storage account + `xcom` container created
- [ ] A5 — Managed identity created; Client ID + Tenant ID recorded
- [ ] A6 — Storage Blob Data Contributor granted to managed identity
- [ ] A7 — Namespace + service account (`remote-exec-sa`) created
- [ ] A8 — Federated credential linking the identity to the service account
- [ ] A9 — Key Vault created; RBAC granted (Secrets Officer to you, Secrets User to identity); secrets added
- [ ] B — Astro Deployment created (Remote Execution, Airflow 3.x); Deployment API Token + Agent Token generated; values.yaml downloaded
- [ ] C1 — Custom agent image built (linux/amd64) and pushed to ACR
- [ ] C2 — values.yaml fully edited and validated; both Kubernetes secrets created; service account labeled/annotated for Helm
- [ ] C3 — Helm chart installed
- [ ] C4 — Pods healthy; agents showing in Astro UI; IP allowlist includes AKS outbound IP; Key Vault test DAG confirms `hello-from-keyvault`
