# dt-edgeconnect

How to deploy **Dynatrace EdgeConnect** on Kubernetes with the **Dynatrace Operator**, and use it for the **Kubernetes Connector** in Dynatrace Workflows (and, optionally, for requests to other private hosts).

This guide assumes a **first deployment in a new environment**. Every step has a check, so you can confirm it works before you continue. Sample manifests are in [`assets/`](assets/).

> **Last verified against the official Dynatrace docs:** 2026-10-30.
> The walkthrough was also exercised in a lab (single-node K3s, Dynatrace Operator 1.9.1 and 1.11.0). Always check the [references](#references) for the latest product behavior.

## Contents

1. [What you build](#what-you-build)
2. [Prerequisites](#1-prerequisites)
3. [Repository layout](#2-repository-layout)
4. [Deployment steps](#3-deployment-steps)
5. [Variants](#4-variants)
6. [Operations](#5-operations)
7. [Security notes](#6-security-notes)
8. [Troubleshooting](#7-troubleshooting)
9. [References](#references)

---

## What you build

**EdgeConnect** is a container that runs inside your private network. It opens an **outbound** secure connection to Dynatrace and runs HTTP(S) requests inside your network on behalf of Dynatrace. Dynatrace does not need any inbound connection.

```
 Dynatrace SaaS  (Workflows, app functions)
        ^
        |  outbound only: HTTPS / WSS on port 443 (EdgeConnect starts the connection)
        |
 +------+--------------- your Kubernetes cluster -----------------+
 |  EdgeConnect pod ---> Kubernetes API server (in-cluster)       |
 |                  ---> other private HTTP(S) services           |
 +-----------------------------------------------------------------+
```

**EdgeConnect and Kubernetes Connector are not the same thing:**

| | EdgeConnect | Kubernetes Connector |
|---|---|---|
| What it is | The "bridge" (a container image, `dynatrace/edgeconnect`) | A set of Workflow actions that work like `kubectl` |
| Purpose | Lets Dynatrace reach private hosts | Lets Workflows read and change Kubernetes resources |
| Where it runs | Anywhere with access to your target systems | Uses an EdgeConnect that runs **inside the cluster** it controls |
| Extra objects | Host patterns | A *connection* in Dynatrace and a ServiceAccount + RBAC in Kubernetes |

In short: the Kubernetes Connector sends its requests **through** an EdgeConnect that has access to the Kubernetes API.

**What you have at the end:**

- An EdgeConnect pod that shows **online** in Dynatrace.
- A Kubernetes Connector connection, created automatically by the Operator.
- A workflow that lists pods and restarts a Deployment through EdgeConnect.

---

## 1. Prerequisites

### 1.1 Versions

| Component | Minimum | Notes |
|---|---|---|
| Dynatrace | SaaS, version **1.275+** | EdgeConnect is available for SaaS environments from 1.275. Kubernetes Connector is documented for the "Latest Dynatrace" (SaaS) platform. |
| Dynatrace Operator | **1.3.0+** | Needed for the `v1alpha2` EdgeConnect API and the Operator-supported setup. Docs recommend at least *n-2* and the latest patch version. |
| Kubernetes | **1.22+** | Required by the Kubernetes Connector. |
| EdgeConnect image | latest | Docs recommend running the latest version. |
| Tools | `kubectl`, `helm` | `helm` is only needed if you install or upgrade the Operator. |

The Dynatrace Operator must already be installed. See the Operator [install and update guide](https://docs.dynatrace.com/docs/ingest-from/setup-on-k8s/guides/deployment-and-configuration/updates-and-maintenance/update-uninstall-operator). This guide uses the namespace `dynatrace`.

### 1.2 Dynatrace privileges

| Task | Permission (from the docs) |
|---|---|
| Create an OAuth client | Access to **Account Management** (OAuth clients are managed by Dynatrace administrators) |
| Create an EdgeConnect configuration | `app-engine:edge-connects:write` **and** `oauth2:clients:manage` where `oauth2:scopes = "app-engine:edge-connects:connect"` |
| Read configurations | `app-engine:edge-connects:read` (included in the Standard and Pro default policies) |
| Update a configuration | `app-engine:edge-connects:write` |
| Rotate an OAuth secret | `oauth2:clients:manage` with the same scope condition |
| Delete a configuration | `app-engine:edge-connects:delete` **and** `oauth2:clients:manage` with the same scope condition |
| Let other users use the Kubernetes connection | A policy with `ALLOW app-settings:objects:read WHERE settings:schemaId = "app:dynatrace.kubernetes.connector:connection"`, assigned to their group |
| Change the sharing of a connection owned by another user | `app-settings:object:admin` |

The **Admin** default policy already contains the EdgeConnect permissions. Creating an EdgeConnect configuration also creates an OAuth client, so OAuth client management is required.

### 1.3 OAuth client scopes

The OAuth client that the Operator uses needs these scopes:

| Scope | Purpose |
|---|---|
| `app-engine:edge-connects:connect` | Connect EdgeConnect |
| `app-engine:edge-connects:write` | Create and update EdgeConnect configurations |
| `app-engine:edge-connects:read` | Read EdgeConnect configurations |
| `app-engine:edge-connects:delete` | Delete EdgeConnect configurations |
| `oauth2:clients:manage` | Manage OAuth clients for EdgeConnect configurations |
| `settings:objects:read` | Read settings objects (used for the Kubernetes connection) |
| `settings:objects:write` | Create settings objects (used for the Kubernetes connection) |
| `fleet-management:container-images:read` | **Optional.** Only needed with Operator 1.11.0+ if the Operator resolves the EdgeConnect image automatically. Otherwise set `imageRef` in the custom resource. |

> The docs page lists the labels for the `read` and `write` EdgeConnect scopes in swapped order compared with the scope names. Select scopes by their **names** as written above.

### 1.4 Kubernetes privileges

The person who runs the steps needs permission to:

- create a `Secret`, a `ServiceAccount` and an `EdgeConnect` resource in the Operator namespace,
- create a `Role` and `RoleBinding` in each namespace you want to manage,
- create a `ClusterRole` and `ClusterRoleBinding` (usually requires cluster-admin).

### 1.5 Network and sizing

| Direction | Target | Port / protocol |
|---|---|---|
| Outbound | `sso.dynatrace.com` (`/sso/oauth2/token`) | 443 / HTTPS |
| Outbound | `<environment-id>.apps.dynatrace.com` | 443 / HTTPS and WSS |
| Outbound | Your internal systems (only if you use host patterns) | As needed (HTTP/HTTPS/WSS) |
| Inbound | None | EdgeConnect does not need inbound connections |

- In provisioner mode the **Dynatrace Operator** also calls Dynatrace (the SSO token endpoint and `<environment-id>.apps.dynatrace.com`) to create the EdgeConnect configuration. In the lab, the Operator pod needed the same outbound access as the EdgeConnect pod.
- EdgeConnect can run behind an HTTP proxy. If the proxy intercepts TLS, add the proxy certificate (see [Troubleshooting](#7-troubleshooting)).
- Recommended size for a typical deployment: **1 GB memory and 1 CPU**. Memory use grows with payload size.
- Limits per the docs: **120 seconds** per request, **6 MB** request or response body, **20 concurrent requests** per container (run more replicas to scale).

### 1.6 Check the prerequisites

Run these commands before you start:

```bash
# Kubernetes version (server must be 1.22 or higher)
kubectl version

# Operator version (label on the Operator Deployment)
kubectl -n dynatrace get deploy dynatrace-operator \
  -o jsonpath='{.metadata.labels.app\.kubernetes\.io/version}{"\n"}'

# EdgeConnect CRD exists and serves v1alpha2
kubectl get crd edgeconnects.dynatrace.com \
  -o jsonpath='{.spec.versions[*].name}{"\n"}'

# Can you create the objects you need?
kubectl auth can-i create edgeconnects.dynatrace.com -n dynatrace
kubectl auth can-i create secrets -n dynatrace
kubectl auth can-i create clusterroles
```

Expected: server version 1.22 or higher, Operator version 1.3.0 or higher, `v1alpha2` in the CRD versions, and `yes` for each `can-i`.

In Dynatrace, confirm that you can open **Account Management > Identity & access management > OAuth clients** and **Settings > General > External requests**.

---

## 2. Repository layout

```
assets/
  01-oauth-secret.template.yaml     Secret for the OAuth client (template, no real values)
  02-serviceaccount.yaml            ServiceAccount used by EdgeConnect
  03-rbac-cluster-readonly.yaml     Cluster-wide read-only access
  04-rbac-namespace-restart.yaml    Restart Deployments in ONE namespace
  05-edgeconnect.yaml               EdgeConnect (Kubernetes Connector + private hosts)
  optional/
    edgeconnect-k8s-only.yaml       Kubernetes Connector only
    edgeconnect-general-only.yaml   Private hosts only (no Kubernetes Connector)
    rbac-multi-namespace.yaml       Restart Deployments in several namespaces
    rbac-cluster-wide-restart.yaml  All namespaces (not recommended)
```

Replace every value in `<angle brackets>` before you apply a file. The samples use the production SaaS stage (`apps.dynatrace.com`, `sso.dynatrace.com`). Use the endpoints of your own Dynatrace stage if it is different.

---

## 3. Deployment steps

### Step 1 - Create the OAuth client

1. In Dynatrace, open **Account Management > Identity & access management > OAuth clients**.
2. Select **Create client**.
3. Enter the email of the owner and a description.
4. Select the scopes from [section 1.3](#13-oauth-client-scopes).
5. Select **Create client**, then copy and store:
   - **Client ID**
   - **Client secret** (shown only once)
   - **Account URN** (format `urn:dtaccount:<account-uuid>`)

**Check:** you have all three values. The client secret cannot be shown again. Keep it in a password manager.

### Step 2 - Create the Secret

Create the Secret **without writing the secret to a file or your shell history**:

```bash
read -rp "Client ID: " CID
read -rsp "Client secret: " CSEC; echo
kubectl -n dynatrace create secret generic edgeconnect-oauth \
  --from-literal=oauth-client-id="$CID" \
  --from-literal=oauth-client-secret="$CSEC"
unset CID CSEC
```

[`assets/01-oauth-secret.template.yaml`](assets/01-oauth-secret.template.yaml) shows the same Secret as YAML. Use it only as a template. Never commit real values to Git.

**Check:**

```bash
kubectl -n dynatrace get secret edgeconnect-oauth
```

Expected: the Secret exists with `DATA` = `2`.

### Step 3 - ServiceAccount and RBAC

EdgeConnect calls the Kubernetes API with a ServiceAccount. Give it the least access it needs.

1. Create the ServiceAccount. It must be in the **same namespace** as the EdgeConnect resource.
2. Give it **read-only** access cluster-wide. Edit the resource kinds if you need more or fewer.
3. Give it **write** access only in the namespaces where workflows must change things. Edit `<target-namespace>` first.

```bash
kubectl apply -f assets/02-serviceaccount.yaml
kubectl apply -f assets/03-rbac-cluster-readonly.yaml
# edit <target-namespace> first:
kubectl apply -f assets/04-rbac-namespace-restart.yaml
```

Rules from the Dynatrace docs:
- A Role and its RoleBinding must be in the same namespace.
- EdgeConnect and the ServiceAccount must be in the same namespace.
- Prefer a namespaced `Role`. Use a `ClusterRole` only when you need cluster-wide access.

**Check:** ask Kubernetes what the ServiceAccount may do (this does not change anything):

```bash
SA=system:serviceaccount:dynatrace:edgeconnect-k8s-sa
kubectl auth can-i list pods -n <target-namespace>   --as=$SA   # yes
kubectl auth can-i patch deployments -n <target-namespace> --as=$SA   # yes
kubectl auth can-i patch deployments -n kube-system --as=$SA   # no
```

### Step 4 - Deploy EdgeConnect

Edit [`assets/05-edgeconnect.yaml`](assets/05-edgeconnect.yaml):

| Field | Set to |
|---|---|
| `spec.apiServer` | `<environment-id>.apps.dynatrace.com` |
| `spec.oauth.resource` | The account URN from Step 1 |
| `spec.oauth.endpoint` | The SSO token endpoint of your Dynatrace stage |
| `spec.hostPatterns` | Your private hosts, or remove the examples if you do not need general requests |
| `spec.oauth.clientSecret` | The Secret name from Step 2 |
| `spec.serviceAccountName` | The ServiceAccount from Step 3 |

Then apply it:

```bash
kubectl apply -f assets/05-edgeconnect.yaml
```

What the key settings mean:

- `oauth.provisioner: true` - the Operator creates and manages the EdgeConnect configuration and the Kubernetes connection. Host patterns are then managed in this resource, not in the Dynatrace UI.
- `kubernetesAutomation.enabled: true` - turns on the Kubernetes Connector support.
- `hostPatterns` - requests to these hosts are routed through EdgeConnect. A host pattern can belong to only one EdgeConnect configuration. Wildcards replace the leading labels (`*.example.org`).

The Operator reconciles EdgeConnect resources only in its own namespace.

### Step 5 - Verify the deployment

**In the cluster:**

```bash
kubectl -n dynatrace get edgeconnects
kubectl -n dynatrace get pods | grep -i edgeconnect
kubectl -n dynatrace logs deploy/edgeconnect-k8s --tail=25
```

Expected:
- The EdgeConnect shows `STATUS` = `Running`.
- The pod is `1/1 Running`.
- The log shows that the OAuth flow succeeded and that the connection is established and verified. Look for lines like `OAuth 2.0 Client Credentials flow succeeded`, `Connection established` and `Connection verified`.

> Seen in the lab: in provisioner mode the pod logs in with its own, narrower client (scope `connect` only) that the Operator prepares. The client ID in the pod log is therefore not the one you created in Step 1. This is expected.

**In Dynatrace:**

1. Open **Settings > General > External requests > EdgeConnect**. Your EdgeConnect (name = `metadata.name`) must show **online** in the **Availability** column, with your host patterns.
2. Open **Settings > Connections > Kubernetes**. A connection with the same name as the EdgeConnect must exist (the Operator creates it).

If the Operator does not create the connection, see [Troubleshooting](#7-troubleshooting).

### Step 6 - Allow workflows to use the connection

1. **Share the connection.** Create a policy with this statement and assign it to the groups that should use the connection. All users in those groups can use all Kubernetes connections.

   ```
   ALLOW app-settings:objects:read WHERE settings:schemaId = "app:dynatrace.kubernetes.connector:connection";
   ```

   If the OAuth client owner is a different user, you need `app-settings:object:admin` to change the sharing of the connection.

2. **Grant permissions to Workflows.** Open **Workflows > Settings > Authorization settings** and select these permissions, in addition to the general Workflows permission:
   - `app-settings:objects:read`
   - `state:app-states:read`
   - `state:app-states:write`

### Step 7 - Test with a workflow

Start with a read-only test.

1. Create a workflow with an **On demand** trigger.
2. Add the action **List resources** (works like `kubectl get`).
3. Select your connection, resource type `pods` and a namespace. Run the workflow.

**Check:** the execution succeeds. The log shows a request to the in-cluster Kubernetes API through your EdgeConnect, and the **Result** tab shows a `PodList`.

> **Get resource** needs a **Name** (one specific object, like `kubectl get pod <name>`). If the name is empty the action fails with `name: Invalid input: expected string, received undefined`. Use **List resources** when you do not have a name.

Then test a change. Pick a stateless Deployment in a namespace where you granted write access in Step 3:

1. Add the action **Rollout restart resource** (works like `kubectl rollout restart`). The older action **Restart deployment** is deprecated.
2. Select the connection, resource type `deployments`, the namespace and the Deployment name. Run it.

**Check:**

```bash
kubectl -n <target-namespace> get pods -w
kubectl -n <target-namespace> rollout status deployment/<name>
```

Expected: a new pod replaces the old one. Without the Role from Step 3 the action should fail with a `forbidden` error. That is a good test that your RBAC is working.

**Test a private host (optional).** If you added `hostPatterns`, open **Settings > General > External requests** and use **Verify URL** to see which EdgeConnect handles a URL and to test a request. Any HTTP request from workflows, app functions or ad-hoc functions that matches a host pattern goes through that EdgeConnect.

---

## 4. Variants

| Need | Use |
|---|---|
| Kubernetes Connector **and** private hosts through one EdgeConnect | [`05-edgeconnect.yaml`](assets/05-edgeconnect.yaml) |
| Kubernetes Connector only | [`optional/edgeconnect-k8s-only.yaml`](assets/optional/edgeconnect-k8s-only.yaml) (follows the docs example, which has no `hostPatterns`; the Operator reference says `hostPatterns` is mandatory with `provisioner: true`, so validate this variant in your environment) |
| Private hosts only | [`optional/edgeconnect-general-only.yaml`](assets/optional/edgeconnect-general-only.yaml) |
| Restart Deployments in several namespaces | [`optional/rbac-multi-namespace.yaml`](assets/optional/rbac-multi-namespace.yaml): one ClusterRole as a template and one RoleBinding per namespace |
| All namespaces | [`optional/rbac-cluster-wide-restart.yaml`](assets/optional/rbac-cluster-wide-restart.yaml). **Not recommended.** It includes system namespaces and the Operator namespace. |

---

## 5. Operations

- **Scale or add redundancy.** Set `spec.replicas`. Multiple instances of one configuration share the load. One container handles up to 20 concurrent requests.
- **Change host patterns.** Edit the `hostPatterns` in the EdgeConnect resource and apply it. In provisioner mode you cannot edit the configuration in the Dynatrace UI.
- **Rotate the OAuth credentials.** Create a new OAuth client, then update the Secret. The change is not applied immediately. Authentication errors can continue until the Operator reconciles the EdgeConnect again, so check the EdgeConnect status afterwards.
- **Updates.** The Operator parameter `autoUpdate` (default `true`) restarts EdgeConnect pods when a new version is available. Update the Operator by following the Operator [update guide](https://docs.dynatrace.com/docs/ingest-from/setup-on-k8s/guides/deployment-and-configuration/updates-and-maintenance/update-uninstall-operator).
- **Remove everything.** Delete the EdgeConnect resource and wait for the pods to stop. Then delete the OAuth client in Account Management, the Secret, and the RBAC objects. Afterwards confirm in Dynatrace that the EdgeConnect configuration and the Kubernetes connection are gone. Remove them manually if they remain.

---

## 6. Security notes

- EdgeConnect works in the context of the Dynatrace environment. **Anyone who can trigger a workflow, app function or ad-hoc function can send requests to the hosts that match your host patterns.** Keep host patterns as specific as possible.
- Limit the network access of the EdgeConnect pod to the systems it needs.
- The Operator parameter `hostRestrictions` (`restrict_hosts_to` in a plain `edgeConnect.yaml`) is an extra local filter. It can only narrow what the host patterns allow. It does not route any request by itself.
- Give the ServiceAccount the **least** Kubernetes access. Prefer a namespaced `Role`. The `patch` verb allows changing any field of an object, not only restarting it.
- Never commit the OAuth client secret. If you need GitOps, use a tool made for secrets (for example SOPS, Sealed Secrets or External Secrets).
- Container images are signed and come with an SBOM, which helps with vulnerability management.

---

## 7. Troubleshooting

| Symptom | What to check |
|---|---|
| EdgeConnect resource is not `Running`, or no pod | `kubectl -n dynatrace logs deploy/dynatrace-operator \| grep -i edgeconnect` |
| Operator log shows `oauth2: "invalid_request"` | Check that the OAuth client was created in **Account Management**, has **all scopes** from [section 1.3](#13-oauth-client-scopes), and that `oauth.resource` is the account URN of that client |
| `admission webhook ... invalid oauth.endpoint` | `oauth.endpoint` must start with `https://` and use the SSO host of your Dynatrace stage. The error message lists example valid values. |
| Pod runs but is **offline** in Dynatrace | Read the pod log. Check outbound access on port 443 to `sso.dynatrace.com` and `<environment-id>.apps.dynatrace.com`, and your proxy settings |
| Certificate errors (self-signed, unable to get issuer, hostname mismatch, expired) | Add every certificate of the signing chain of the target server (and of a TLS-intercepting proxy). With the Operator, use `caCertsRef` (a ConfigMap with the certificates under `certs`). |
| Kubernetes connection is missing in **Settings > Connections > Kubernetes** | Confirm `kubernetesAutomation.enabled: true` and `oauth.provisioner: true`, that the OAuth client has `settings:objects:read` and `settings:objects:write`, and check the Operator log. In the lab, an EdgeConnect configuration created manually in the UI conflicted with provisioner mode; remove it first (this is lab experience, not a documented rule). |
| Workflow action fails with `forbidden` | Check the Role and RoleBinding with `kubectl auth can-i ... --as=system:serviceaccount:dynatrace:edgeconnect-k8s-sa` |
| **Get resource** fails with `name: expected string` | Fill in **Name**, or use **List resources** |
| Workflow cannot use the connection | Check the sharing policy (Step 6) and the Workflows authorization settings |
| Request fails or times out | Limits: 120 seconds per request, 6 MB body (larger fails with HTTP 400), 20 concurrent requests per container |

---

## References

Official Dynatrace documentation (checked on 2026-10-30):

- [Configure and deploy EdgeConnect](https://docs.dynatrace.com/docs/ingest-from/edgeconnect)
- [Kubernetes Connector overview](https://docs.dynatrace.com/docs/analyze-explore-automate/workflows/actions/kubernetes-automation)
- [Set up Kubernetes Connector](https://docs.dynatrace.com/docs/analyze-explore-automate/workflows/actions/kubernetes-automation/kubernetes-workflows-setup)
- [Actions for Kubernetes Connector](https://docs.dynatrace.com/docs/analyze-explore-automate/workflows/actions/kubernetes-automation/kubernetes-workflow-actions)
- [EdgeConnect for Kubernetes Connector (ServiceAccount and RBAC)](https://docs.dynatrace.com/docs/ingest-from/setup-on-k8s/guides/deployment-and-configuration/edgeconnect/kubernetes-automation)
- [Set up EdgeConnect for Kubernetes Connector using the Dynatrace Operator](https://docs.dynatrace.com/docs/ingest-from/setup-on-k8s/guides/deployment-and-configuration/edgeconnect/kubernetes-automation/edge-connect-kubernetes-automation-operator-supported-setup)
- [Provision EdgeConnect for a Dynatrace environment](https://docs.dynatrace.com/docs/ingest-from/setup-on-k8s/guides/deployment-and-configuration/edgeconnect/edge-connect-provision)
- [EdgeConnect parameters for the Dynatrace Operator](https://docs.dynatrace.com/docs/ingest-from/setup-on-k8s/reference/edgeconnect-parameters)
- [Update or uninstall the Dynatrace Operator](https://docs.dynatrace.com/docs/ingest-from/setup-on-k8s/guides/deployment-and-configuration/updates-and-maintenance/update-uninstall-operator)
- [OAuth clients](https://docs.dynatrace.com/docs/manage/identity-access-management/access-tokens-and-oauth-clients/oauth-clients)
