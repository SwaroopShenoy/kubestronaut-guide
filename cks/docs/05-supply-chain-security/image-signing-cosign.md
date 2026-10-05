# Image Signing and Verification with Cosign

Up: [CKS hub](../../README.md) · Domain 5 — Supply Chain Security (20%) · Prev: [Image scanning with Trivy](image-scanning-trivy.md) · Next: [Admission control](admission-control.md)

A signature proves who built an image and that it has not changed since. This topic covers signing and verifying images with Cosign, both with keys and without.

## Exam scope

**In scope:** signing an image, verifying a signature, understanding why verification should use a digest, and recognizing the failure modes. Enforcement at admission is covered in [Admission control](admission-control.md).

## Concepts

- A signature is stored against an image **digest** (`sha256:...`). If you give cosign a tag, it looks up the digest the tag points to at that moment and signs that digest. A tag can later be moved to different content; a digest cannot.
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

Signing by tag also works:

```bash
cosign sign --key cosign.key registry.example.com/app:1.0
```

Cosign v3.1.3 prints this warning and carries on:

```
Image reference registry.example.com/app:1.0 uses a tag, not a digest, to identify the image to sign.
This can lead you to sign a different image than the intended one. Please use a
digest (example.com/ubuntu@sha256:abc123...) rather than tag
(example.com/ubuntu:latest) for the input to cosign. The ability to refer to
images by tag will be removed in a future release.
```

So the tag form is accepted today, but it is discouraged and may stop working in a later release. The risk is a race: if the tag moves between your build and the sign command, you sign the wrong image. Using the digest removes that doubt, and it is what the warning recommends.

Verify:

```bash
cosign verify --key cosign.pub registry.example.com/app@${DIGEST}
echo "exit=$?"
```

On success cosign prints the verified claims as JSON and exits 0. On failure it prints an error and exits non-zero. Scripts should test the exit code, not the presence of specific text.

Verifying by tag also works, and prints no warning. The tag is resolved when the command runs, so the result describes whatever the tag points to at that moment, which may not be the image that runs later. If the tag has moved to an unsigned image, verification fails. Use digests in anything that enforces policy.

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
| "no matching signatures" | Image not signed, or signed by a different key, or the tag now points to a newly built, unsigned image |
| `UNAUTHORIZED` or `denied` when signing | Not logged in to the registry; cosign reads the same credentials as `docker login` (or use `cosign login`) |
| `MANIFEST_UNKNOWN` or "accessing entity" when signing | The image has not been pushed, or the tag is mistyped; cosign signs images in a registry, not local-only images |
| Warning "uses a tag, not a digest" | A tag was passed; the digest it pointed to was signed. Pass the digest to be certain |
| Keyless passes for any repo | Identity not pinned in `--certificate-identity` |
| Private key leaked | Rotate: generate a new pair, re-sign current images, revoke trust in policies |

## Common mistakes

- Signing by tag. It works, but the signature lands on whatever the tag points to at that moment. Sign the digest.
- Storing `cosign.key` in the repository. Store it in the CI secret store, and keep the password separate.
- Verifying in CI but not at admission. A pod can still be created by hand.

## Quick reference

```bash
cosign generate-key-pair
cosign sign --key cosign.key <registry>/<image>@<digest>
cosign verify --key cosign.pub <registry>/<image>@<digest>
cosign tree <registry>/<image>@<digest>
```

---

Prev: [Image scanning with Trivy](image-scanning-trivy.md) · Next: [Admission control](admission-control.md)  
<sub>© 2026 Swaroop Shenoy · Licensed under [CC BY 4.0](../../../LICENSE)</sub>
