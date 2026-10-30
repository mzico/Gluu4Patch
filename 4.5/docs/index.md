# Gluu CN 4.5.14: patch oxPassport for CVE-2026-111497 (Node.js Brotli)

Applies to: Gluu Cloud Native (Helm chart `gluu/gluu` 1.8.49, app 4.5.14) on Kubernetes.

## What this fixes
- oxPassport ships Node.js with the vulnerable Brotli decoder (Node2Shell, CVE-2026-111497).
- Fixed Node.js versions: 22.23.4, 24.22.1, 26.12.1. The 4.5.14-1 oxPassport image runs Node v22.22.0 (vulnerable).
- The patched images (`4.5.14_dev`) run Node v22.23.4 and brotli-libs 1.2.1.

## Important: Python 3.14 and your Kubernetes CA
The patched images use Python 3.14, which strictly validates the Kubernetes CA certificate. Some clusters have a CA that fails this check. Pods then stay in a restart loop with `CERTIFICATE_VERIFY_FAILED`.

## Step 1. Check your cluster CA (every cluster)
```bash
kubectl get cm kube-root-ca.crt -n <gluu-namespace> -o jsonpath="{.data['ca\.crt']}" | openssl x509 -noout -ext basicConstraints,keyUsage,subjectKeyIdentifier,authorityKeyIdentifier
```
The CA is fine if it shows all of these:
- Basic Constraints: **critical**, `CA:TRUE`
- Key Usage: includes **Certificate Sign**
- Subject Key Identifier (and ideally Authority Key Identifier)

If something is missing, either rotate the cluster CA (best fix, needs a maintenance window: restart the CNI, ingress and all pods), or use the temporary workaround in Step 2.

## Step 2. Create `gluu-patch.yaml`
Image tags (always needed):
```yaml
config:
  image: {tag: 4.5.14_dev, pullPolicy: Always}
persistence:
  image: {tag: 4.5.14_dev, pullPolicy: Always}
oxauth:
  image: {tag: 4.5.14_dev, pullPolicy: Always}
oxtrust:
  image: {tag: 4.5.14_dev, pullPolicy: Always}
oxd-server:
  image: {tag: 4.5.14_dev, pullPolicy: Always}
oxpassport:
  image: {tag: 4.5.14_dev, pullPolicy: Always}
```
Add this **only if your CA fails Step 1** (temporary workaround):
```yaml
global:
  sslCertFromDomain: "true"
  usrEnvs:
    normal:
      GLUU_DISABLE_X509_STRICT_VERIFICATION: "true"
config:
  usrEnvs:
    normal:
      GLUU_DISABLE_X509_STRICT_VERIFICATION: "true"
persistence:
  usrEnvs:
    normal:
      GLUU_DISABLE_X509_STRICT_VERIFICATION: "true"
```
Merge the two blocks if a key (for example `config:`) appears twice.

## Step 3. Upgrade
```bash
kubectl -n <gluu-namespace> delete job gluu-config gluu-persistence --ignore-not-found
helm upgrade <release> gluu/gluu --version 1.8.49 -n <gluu-namespace> --reuse-values -f gluu-patch.yaml
```
(Existing Jobs cannot be patched, so delete them first.)

## Step 4. Verify
```bash
kubectl -n <gluu-namespace> get pods
kubectl -n <gluu-namespace> exec deploy/<release>-oxpassport -- node --version
```
- All pods `Running`, config and persistence jobs `Completed`.
- Node version is **v22.23.4 or higher**.

## Notes
- `4.5.14_dev` is a pre-release tag; use the final tag when it is published.
- The workaround only relaxes the CA check inside Gluu pods. Fix the CA when you can.
- The same Python requirement applies to 4.5.15 and later images.
- Tested on MicroK8s 1.30 with a CA that has no Key Usage extension.
