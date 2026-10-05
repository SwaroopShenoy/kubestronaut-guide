# Image Signing and Verification with Cosign

Up: [CKS hub](../../CKS_2026_Complete_Crash_Course.md) · Domain 5 — Supply Chain Security (20%) · Prev: [Image scanning with Trivy](image-scanning-trivy.md) · Next: [Admission control](admission-control.md)

## Exam scope

**In scope:** signing an image, verifying a signature, understanding why verification should use a digest, and recognizing the failure modes. Enforcement at admission is covered in [Admission control](admission-control.md).

## Concepts

- Signing binds a signature to an image **digest** (`sha256:...`), not to a tag. A tag can be moved; a digest cannot.
- Key-based: a private key signs, a public key verifies. Protect the private key.
- Keyless: CI proves its identity with an OIDC token; the signature is recorded in a transparency log. No long-lived key to leak.

## Key-based signing

```bash
cosign generate-key-pair           # writes cosign.key (encrypted, prompts for a password) and cosign.pub
```

Sign by digest:

```bash
DIGEST=$(crane digest registry.example.com/app:1.0)   # or: docker inspect --format '{{index .RepoDigests 0}}'
cosign sign --key cosign.key registry.example.com/app@${DIGEST}
```

Verify:

```bash
cosign verify --key cosign.pub registry.example.com/app@${DIGEST}
echo "exit=$?"
```

On success cosign prints the verified claims as JSON and exits 0. On failure it prints an error and exits non-zero. Scripts should test the exit code, not the presence of specific text.

Verifying by tag still works but resolves the tag at verification time, so a moved tag can pass the wrong image. Use digests in anything that enforces policy.

## Keyless signing in CI

The CI job needs `id-token: write`:

```yaml
permissions:
  id-token: write
  contents: read
steps:
- uses: sigstore/cosign-installer@v3
- run: cosign sign --yes registry.example.com/app@${DIGEST}
```

Verify, pinning the identity that is allowed to sign:

```bash
cosign verify \
  --certificate-identity-regexp 'https://github.com/my-org/my-repo/.github/workflows/.*' \
  --certificate-oidc-issuer https://token.actions.githubusercontent.com \
  registry.example.com/app@${DIGEST}
```

Always pin `--certificate-identity` or the regexp and the issuer. Without them, verification accepts any valid keyless signature.

Do not use `GITHUB_TOKEN` as an identity token for signing. The workflow must request an OIDC token through the runner, which `id-token: write` enables.

## Storing and finding signatures

Cosign stores signatures in the same registry as the image, under a derived tag. To inspect:

```bash
cosign tree registry.example.com/app@${DIGEST}
```

## Attestations and SBOMs (concept)

```bash
cosign attest --key cosign.key --type cyclonedx --predicate sbom.cdx.json registry.example.com/app@${DIGEST}
cosign verify-attestation --key cosign.pub --type cyclonedx registry.example.com/app@${DIGEST}
```

The older `cosign attach sbom` command is deprecated in favor of attestations.

## Failure modes

| Symptom | Cause |
|---|---|
| "no matching signatures" | Image not signed, or signed by a different key, or verified by tag after rebuild |
| Passes with wrong key | Verified by tag, which was moved; verify by digest |
| Keyless passes for any repo | Identity not pinned in `--certificate-identity` |
| Private key leaked | Rotate: generate a new pair, re-sign current images, revoke trust in policies |

## Common mistakes

- Signing the tag instead of the digest.
- Storing `cosign.key` in the repository. Store it in the CI secret store, and keep the password separate.
- Verifying in CI but not at admission. A pod can still be created by hand.

## Quick reference

```bash
cosign generate-key-pair
cosign sign --key cosign.key <registry>/<image>@<digest>
cosign verify --key cosign.pub <registry>/<image>@<digest>
cosign tree <registry>/<image>@<digest>
```
