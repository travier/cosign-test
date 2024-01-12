# Example and test repo for building and signing containers via GitHub Actions

Example and test repo used to test building container with podman/buildah and
sign them with podman or cosign / sigstore in GitHub Actions.

See the resulting / in progress PRs / issues:
- Old podman version in `ubuntu-latest` (22.04) in GitHub Actions runners:
  - https://github.com/containers/podman/issues/20771
  - https://github.com/redhat-actions/buildah-build/issues/131
  - Workaround: https://github.com/travier/podman-action
- Missing support for specifying signatures in push-to-registry action:
  - https://github.com/redhat-actions/push-to-registry/issues/89
  - https://github.com/redhat-actions/push-to-registry/pull/90
- Missing options in `podman image trust` to set the policy via the command
  line only:
  - https://github.com/containers/podman/issues/16624
  - See below for workaround
- Missing command in podman to easily verify that an image is correctly signed:
  - Issue to be filled

## Verifying sigstore container signatures made with self-managed keys

Verifying with cosign:

```
$ curl -O "https://raw.githubusercontent.com/travier/cosign-test/main/quay.io-travier-containers.pub"
$ cosign verify --key quay.io-travier-containers.pub quay.io/travier/cosign-example:latest-cosign
# Podman does not push the signature to rekor thus cosign can not verify it
$ cosign verify --key quay.io-travier-containers.pub quay.io/travier/cosign-example:latest-podman
```

How to configure signature verification in podman:

```
$ sudo mkdir -p /etc/pki/containers
$ curl -O "https://raw.githubusercontent.com/travier/cosign-test/main/quay.io-travier-containers.pub"
$ sudo cp quay.io-travier-containers.pub /etc/pki/containers/
$ sudo restorecon -RFv /etc/pki/containers

$ cat /etc/containers/registries.d/quay.io-travier.yaml
docker:
  quay.io/travier:
    use-sigstore-attachments: true
$ sudo restorecon -RFv /etc/containers/registries.d/quay.io-travier.yaml

$ cat /etc/containers/policy.json
{
    "default": [
        {
            "type": "reject"
        }
    ],
    "transports": {
        "docker": {
            ...
            "quay.io/travier": [
                {
                    "type": "sigstoreSigned",
                    "keyPath": "/etc/pki/containers/quay.io-travier-containers.pub",
                    "signedIdentity": {
                        "type": "matchRepository"
                    }
                }
            ],
            ...
            "": [
                {
                    "type": "insecureAcceptAnything"
                }
            ]
        },
        ...
    }
}
...
```

## Verifying sigstore container image keyless signatures

Verifying with cosign:

```
$ COSIGN_EXPERIMENTAL=true cosign verify --certificate-identity "https://github.com/travier/cosign-test/.github/workflows/cosign-keyless.yml@refs/heads/main" --certificate-oidc-issuer https://token.actions.githubusercontent.com ghcr.io/travier/cosign-test | jq
```

For keyless signatures (does not work yet, see
[containers/image#2235](https://github.com/containers/image/pull/2235)):

```
$ sudo mkdir -p /etc/pki/containers
$ curl -O "https://raw.githubusercontent.com/sigstore/root-signing/main/targets/fulcio_v1.crt.pem"
$ curl -O "https://raw.githubusercontent.com/sigstore/root-signing/main/targets/rekor.pub"
$ sudo cp fulcio_v1.crt.pem rekor.pub /etc/pki/containers/
$ sudo restorecon -RFv /etc/pki/containers

$ cat /etc/containers/registries.d/ghcr.io-travier.yaml
docker:
  ghcr.io/travier:
    use-sigstore-attachments: true
$ sudo restorecon -RFv /etc/containers/registries.d/ghcr.io-travier.yaml

$ cat /etc/containers/policy.json
{
    "default": [
        {
            "type": "reject"
        }
    ],
    "transports": {
        "docker": {
            ...
            "ghcr.io/travier": [
            {
                "type": "sigstoreSigned",
                "fulcio": {
                    "caPath": "/etc/pki/containers/fulcio_v1.crt.pem",
                    "oidcIssuer": "https://token.actions.githubusercontent.com",
                    "subjectEmail": "https://github.com/travier/cosign-test/.github/workflows/cosign-keyless.yml@refs/heads/main"
                },
                "rekorPublicKeyPath": "/etc/pki/containers/rekor.pub",
                "signedIdentity": {
                    "type": "matchRepository"
                }
            }
            ],
            ...
            "": [
                {
                    "type": "insecureAcceptAnything"
                }
            ]
        },
        ...
    }
}
...
```

## License

See [LICENSE](LICENSE) or [CC0](https://creativecommons.org/public-domain/cc0/).
