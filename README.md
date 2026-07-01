# Signed & Sealed

Cryptographic proof of container image provenance, enforced at the Kubernetes admission layer — not just checked, blocked.
This project proves one concrete thing end to end: **if an unsigned or tampered image is pushed to this cluster, it is actually rejected, not just flagged.**

## Stack

- GitHub Actions (build + push)
- Cosign (keyless signing via Sigstore — Fulcio + Rekor)
- Kyverno (Kubernetes admission enforcement)
- kind / Kubernetes
- Docker Hub

## Architecture

```
git push main
  └── GitHub Actions (build-sign.yml)
        ├── docker buildx build → SHA-tagged image (no :latest)
        ├── push to Docker Hub
        └── cosign sign --keyless
              ├── OIDC token requested from GitHub's identity provider
              ├── exchanged with Fulcio for a short-lived signing certificate
              └── signature + certificate recorded in Rekor (public transparency log)

kubectl apply / pod admission
  └── Kyverno ClusterPolicy (require-signed-images)
        └── verifyImages rule
              ├── keyless attestor, issuer = token.actions.githubusercontent.com
              ├── certificate subject locked to this repo's workflow on refs/heads/main
              └── verifyDigest: true → rejects mutable tag references
```

## Why keyless signing

No private key to generate, store, rotate, or accidentally leak through a misconfigured log line. Trust is anchored to GitHub's OIDC identity provider instead.

When the workflow runs, Cosign requests an OIDC token describing the exact workflow, repository, and branch that's executing. It exchanges that token with Fulcio, Sigstore's certificate authority, for an X.509 certificate valid for roughly ten minutes. Cosign signs the image digest with that certificate, and the signature plus certificate are recorded in Rekor, a public append-only transparency log. Verification later doesn't depend on the signing key still existing — it depends on the Rekor entry and certificate chain.

## Key implementation details

**No `:latest` tags.** Every image is built and signed by Git SHA. Tags are mutable — signing a digest ties the signature permanently to one immutable artifact.

**Certificate subject locked to `main`.** The Kyverno policy only trusts signatures where the certificate subject matches this exact workflow file running on `refs/heads/main`. A signature produced from a fork or a feature branch fails verification.

**`verifyDigest: true`.** Without this, Kyverno would accept a signature attached to a tag reference, even though tags can be repointed to a different (unsigned) image after the fact. This flag forces digest-pinned references only.

**`id-token: write` permission.** The GitHub Actions job needs this explicitly for Cosign to request the OIDC token. Omitting it fails the signing step with a vague, easy-to-miss error.

## Proof of work

### 1. Signature verification (Cosign / Rekor)

```
cosign verify \
  --certificate-identity-regexp "https://github.com/Eliab-K/signed-and-sealed/.github/workflows/build-sign.yml@refs/heads/main" \
  --certificate-oidc-issuer "https://token.actions.githubusercontent.com" \
  eliabkuta/signed-and-sealed@sha256:e41067c02791848019d230ba10c807219e5445a593ac8747ce218aa636730485
```

Result: verification succeeded. Certificate subject, issuer, and Rekor transparency log entry all confirmed the image was signed by this exact workflow, on `main`, at the recorded commit (`cac28d84d8e7fb0803edbda613f8e217b1899559`).

### 2. Unsigned image — blocked at admission

```
$ kubectl run unsigned-test --image=eliabkuta/signed-and-sealed:unsigned-test

Error from server: admission webhook "mutate.kyverno.svc-fail" denied the request:
resource Pod/default/unsigned-test was blocked due to the following policies
require-signed-images:
  verify-signature: 'failed to verify image docker.io/eliabkuta/signed-and-sealed:unsigned-test:
    .attestors[0].entries[0].keyless: no signatures found'
```

### 3. Signed image — admitted normally

```
$ kubectl run signed-test --image=eliabkuta/signed-and-sealed@sha256:e41067c02791848019d230ba10c807219e5445a593ac8747ce218aa636730485
pod/signed-test created

$ kubectl get pod signed-test
NAME          READY   STATUS    RESTARTS   AGE
signed-test   1/1     Running   0          8s
```

## Reproducing this

1. Fork or clone this repo.
2. Create a Docker Hub repository and an access token with **Read & Write** permissions.
3. Add two GitHub repository secrets: `DOCKERHUB_USERNAME`, `DOCKERHUB_TOKEN`.
4. Push to `main` — the `build-sign.yml` workflow builds, pushes, and signs the image automatically.
5. Install Kyverno on a Kubernetes cluster:
   ```
   helm repo add kyverno https://kyverno.github.io/kyverno/
   helm install kyverno kyverno/kyverno -n kyverno --create-namespace
   ```
6. Apply the policy:
   ```
   kubectl apply -f kyverno-policy.yaml
   ```
7. Test both cases — an unsigned image should be rejected, the signed image should be admitted.

## What this doesn't cover (and why that's an honest scope, not an oversight)

This project is deliberately scoped to signing + admission enforcement. It does not include:

- **SBOM generation** (e.g. via Syft) attached as a verifiable attestation
- **SLSA provenance** recording build context (which repo, which workflow, which commit) as a separate attestation layer
- **A separate verification stage** — right now trust is established at admission time only; a stronger architecture would verify again post-deploy, in a different execution context than signing

Those are the natural next layers: SBOM and provenance close the gap between "this image is signed" and "this image contains exactly what the source would produce, built by exactly this pipeline." Signing alone proves *who* built it; provenance proves *how*.

## Why this matters

Signing an image proves who built it. It says nothing about whether the cluster receiving that image actually checks — or whether the check is enforced rather than just logged. This project closes that specific gap for a payments/fintech-adjacent context: proving what's actually running in production, verifiably, with zero private keys to manage.
