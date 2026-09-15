# syft-enclave-tinfoil

Measured [Tinfoil](https://docs.tinfoil.sh) workload config for the **syft enclave**.

This repo is public because it has to be: a data owner verifies a running enclave
by reading the measurement published in this repo's signed GitHub releases, and
they cannot verify what they cannot see. The enclave's source and container image
live elsewhere — the image appears here only as a `sha256:` digest, which is what
the attestation commits to.

## What is in a release

Publishing a release runs `.github/workflows/tinfoil-release.yml`, which tags the
commit and dispatches the publish workflow. That computes the expected CVM launch
measurement for `tinfoil-config.yml` **offline** (no VM boots, and the container
image is never pulled), creates a GitHub release carrying it, and signs it
keylessly to the Sigstore transparency log via GitHub OIDC.

Each release has two assets:

| Asset | Contents |
|---|---|
| `tinfoil-deployment.json` | the SEV-SNP / TDX measurements, VM shape, kernel cmdline, CVM image hashes, and the full `tinfoil-config.yml` embedded base64 |
| `tinfoil.hash` | the sha256 of that file — and the subject the Sigstore DSSE signed |

## Verifying a release yourself

```bash
curl -sLO https://github.com/OpenMined/syft-enclave-tinfoil/releases/download/<tag>/tinfoil-deployment.json
curl -sL  https://github.com/OpenMined/syft-enclave-tinfoil/releases/download/<tag>/tinfoil.hash
shasum -a 256 tinfoil-deployment.json     # must equal tinfoil.hash
```

That hash is what the Sigstore bundle signed, so anything inside
`tinfoil-deployment.json` — including the embedded config and therefore the pinned
image digest — is covered by the signature. To re-derive the measurement from
scratch, run `sev-snp-measure` / `tdx-measure` over the firmware, kernel, initrd
and `cmdline` the file records.

To check the config validates against the canonical schema:

```bash
go run github.com/tinfoilsh/tinfoil-config/cmd/tinfoil-config@latest tinfoil-config.yml
```

## Changing the config

Do not edit this repo by hand. The canonical copy lives in
[PySyft](https://github.com/OpenMined/PySyft) at
`packages/syft-enclave/tinfoil/tinfoil-config.yml`, so the image and the config
that pins it are reviewed together. From `packages/syft-enclave/` there:

```bash
just tinfoil-release vX.Y.Z    # build + push the image, pin the digest, open a PR here
# merge the PR, then:
just tinfoil-publish vX.Y.Z    # publish the measured, signed release
just tinfoil-deploy vX.Y.Z <enclave-email> <data-owners>
```

Full walkthrough, and what the attestation does and does not prove:
[`packages/syft-enclave/docs/tinfoil.md`](https://github.com/OpenMined/PySyft/blob/dev/packages/syft-enclave/docs/tinfoil.md).

**Releases are permanent.** A release is a signed GitHub release and a
transparency-log entry; it cannot be unpublished. Pick the next version rather
than re-cutting one.
