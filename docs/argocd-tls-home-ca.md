# HTTPS for the Argo CD UI with a home CA (cert-manager)

This guide adds a trusted TLS certificate to the Argo CD web UI on
`k3s-cluster1`, using cert-manager and a private certificate authority (CA)
that lives in the cluster. Every device you browse from trusts the CA once,
and then any certificate it signs, now or later, works without browser warnings.

Let's Encrypt can't be used here: `argocd.10.0.0.125.nip.io` points to a
private LAN IP, so Let's Encrypt can't validate it. If you own a domain, see
[Alternative: Let's Encrypt](#alternative-lets-encrypt) at the end.

## Starting point

This guide assumes the setup already in place on `k3s-cluster1`:

- Argo CD is installed in the `argocd` namespace.
- `argocd-server` runs in plain-HTTP mode (`server.insecure: "true"` in the
  `argocd-cmd-params-cm` ConfigMap). Traefik handles TLS, so this stays as is.
- An Ingress `argocd/argocd-server` (class `traefik`) sends
  `argocd.10.0.0.125.nip.io` to service `argocd-server` port 80.

If you're starting from scratch, set those up first:

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

All commands below pass `--context k3s-cluster1` explicitly, so they work
whichever kubectl context is currently selected.

## 1. Install cert-manager

Check the [cert-manager releases](https://github.com/cert-manager/cert-manager/releases)
for the latest version and put it in the URL:

```bash
kubectl --context k3s-cluster1 apply -f \
  https://github.com/cert-manager/cert-manager/releases/download/v1.16.2/cert-manager.yaml

kubectl --context k3s-cluster1 -n cert-manager rollout status deploy/cert-manager
kubectl --context k3s-cluster1 -n cert-manager rollout status deploy/cert-manager-webhook
```

Wait until the webhook is ready before going on. If you create issuers
earlier, the webhook rejects them.

## 2. Create the home CA and an issuer

This creates three objects:

1. `selfsigned`: a ClusterIssuer used only once, to create the CA.
2. `home-ca`: the CA certificate. Its key pair is stored in the Secret
   `cert-manager/home-ca`.
3. `home-ca` (ClusterIssuer): signs certificates for any namespace using that CA.

```bash
kubectl --context k3s-cluster1 apply -f - <<'EOF'
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: selfsigned
spec:
  selfSigned: {}
---
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: home-ca
  namespace: cert-manager
spec:
  isCA: true
  commonName: Home Lab CA
  secretName: home-ca
  duration: 87600h   # 10 years
  privateKey:
    algorithm: ECDSA
    size: 256
  issuerRef:
    name: selfsigned
    kind: ClusterIssuer
---
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: home-ca
spec:
  ca:
    secretName: home-ca
EOF
```

Check that both issuers show `READY=True`:

```bash
kubectl --context k3s-cluster1 get clusterissuer
kubectl --context k3s-cluster1 -n cert-manager get certificate home-ca
```

## 3. Request a certificate on the Argo CD Ingress

Add the issuer annotation and a `tls:` section. cert-manager notices them,
creates a `Certificate`, and stores the signed key pair in
`argocd/argocd-server-tls`. It renews the certificate automatically before
it expires (the default lifetime is 90 days).

```bash
kubectl --context k3s-cluster1 -n argocd annotate ingress argocd-server \
  cert-manager.io/cluster-issuer=home-ca

kubectl --context k3s-cluster1 -n argocd patch ingress argocd-server --type merge -p \
  '{"spec":{"tls":[{"hosts":["argocd.10.0.0.125.nip.io"],"secretName":"argocd-server-tls"}]}}'

kubectl --context k3s-cluster1 -n argocd get certificate   # wait for READY=True
```

Traefik loads the secret on its own; nothing needs restarting. The UI is now
served on both `https://` and `http://`.

## 4. Trust the CA on your devices

Export the CA certificate. It's the public half only and safe to copy around:

```bash
kubectl --context k3s-cluster1 -n cert-manager get secret home-ca \
  -o jsonpath='{.data.ca\.crt}' | base64 -d > home-lab-ca.crt
```

Import it on every device you use to open the UI:

| Platform | How |
|---|---|
| Debian / Ubuntu / ChromeOS Linux | `sudo cp home-lab-ca.crt /usr/local/share/ca-certificates/ && sudo update-ca-certificates` |
| Chrome / Chromium (Linux, ChromeOS) | Settings → Privacy and security → Security → Manage certificates → Authorities → Import, tick "Trust this certificate for identifying websites" |
| Firefox | Settings → Privacy & Security → Certificates → View Certificates → Authorities → Import, tick "Trust this CA to identify websites" |
| macOS | Open the file in Keychain Access, add it to *System*, then set *When using this certificate* to **Always Trust** |
| Windows | Double-click the file → Install Certificate → Local Machine → *Trusted Root Certification Authorities* |
| iOS | AirDrop or email the file and install the profile, then enable it under Settings → General → About → Certificate Trust Settings |
| Android | Settings → Security → Encryption & credentials → Install a certificate → CA certificate |

Chrome on ChromeOS and Firefox keep their own certificate stores, so
`update-ca-certificates` alone doesn't help them.

> **Keep the `cert-manager/home-ca` Secret private.** It contains the CA's
> private key, and anyone who has it can create certificates your devices
> will trust for *any* website. Don't commit it to Git, and limit who can
> read Secrets in the `cert-manager` namespace.

## 5. (Optional) Redirect HTTP to HTTPS

```bash
kubectl --context k3s-cluster1 apply -f - <<'EOF'
apiVersion: traefik.io/v1alpha1
kind: Middleware
metadata:
  name: redirect-https
  namespace: argocd
spec:
  redirectScheme:
    scheme: https
    permanent: true
EOF

kubectl --context k3s-cluster1 -n argocd annotate ingress argocd-server \
  traefik.ingress.kubernetes.io/router.middlewares=argocd-redirect-https@kubernetescrd
```

The annotation's value has the form `<namespace>-<middleware name>@kubernetescrd`.

## Verify

```bash
# Certificate issued by the home CA, and the hostname it covers
curl -sv --cacert home-lab-ca.crt https://argocd.10.0.0.125.nip.io/ -o /dev/null 2>&1 \
  | grep -E 'issuer:|subject:|SSL certificate verify'

# Redirect, if step 5 was applied (expect 301/308 to https://)
curl -sI http://argocd.10.0.0.125.nip.io/ | head -3
```

Then open **https://argocd.10.0.0.125.nip.io**. There should be no browser warning.

Argo CD CLI login through Traefik:

```bash
argocd login argocd.10.0.0.125.nip.io --grpc-web
# on a machine that doesn't trust the CA: add --insecure
```

## Using the CA for other services

Any other Ingress on the cluster can get a certificate the same way: add the
annotation and a `tls:` section with its own `secretName`:

```yaml
metadata:
  annotations:
    cert-manager.io/cluster-issuer: home-ca
spec:
  tls:
  - hosts: [myapp.10.0.0.125.nip.io]
    secretName: myapp-tls
```

Devices that already trust the CA trust these certificates as well.

For `k3s-cluster2`, either repeat steps 1–2 there, which creates a second
CA that has to be trusted separately, or copy the `cert-manager/home-ca`
Secret to cluster2 before creating the `home-ca` ClusterIssuer there, so one
CA covers both clusters.

## Troubleshooting

| Symptom | Check |
|---|---|
| `Certificate` stays `READY=False` | `kubectl --context k3s-cluster1 -n argocd describe certificate argocd-server-tls` and `kubectl --context k3s-cluster1 -n cert-manager logs deploy/cert-manager` |
| Applying the issuers fails with a webhook error | cert-manager isn't ready yet; rerun the `rollout status` commands from step 1 and apply again |
| Browser still shows the Traefik default certificate | The `tls.hosts` entry must match the URL exactly; check that `argocd/argocd-server-tls` exists |
| Browser shows "not trusted" | The CA isn't in *that* browser's store (see the note under step 4) |
| Redirect loop after step 5 | Make sure `server.insecure` is still `"true"`. If argocd-server serves TLS itself, it redirects to https again behind Traefik |
| `argocd login` hangs or fails | Use `--grpc-web`. Traefik on this Ingress doesn't pass Argo CD's native gRPC through |

## Rotating the CA

The CA is valid for 10 years. To replace it earlier, for example after a
leak: delete the `cert-manager/home-ca` Certificate and Secret, apply step 2
again, delete every Secret it signed (such as `argocd/argocd-server-tls`) so
they're issued again, then remove the old CA from your devices and import
the new one.

## Alternative: Let's Encrypt

If you own a domain whose DNS provider has an API (Cloudflare, Route 53, …),
cert-manager can get publicly trusted certificates with a DNS-01
`ClusterIssuer`, and no device needs any setup. Point a name such as
`argocd.home.example.com` to `10.0.0.125`, create an `acme` ClusterIssuer
with a `dns01` solver for your provider (see the
[cert-manager DNS-01 docs](https://cert-manager.io/docs/configuration/acme/dns01/)),
and use that issuer name and hostname in step 3 instead.
