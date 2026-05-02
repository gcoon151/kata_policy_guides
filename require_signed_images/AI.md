# AI Execution Spec: Allow Only Approved Images In Red Hat Peer Pods

## Objective

Configure Red Hat peer pods so:

- unsigned images fail
- signed but unapproved images fail
- signed and approved images start

## Preconditions

- TDX already works
- Red Hat build of Trustee is installed
- Trustee uses Intel Trust Authority for attestation
- OpenShift Sandboxed Containers version is 1.12
- peer pods are already functional

## Safety Rules

- do not overwrite existing cluster values without user confirmation
- before changing any existing value, back it up
- if a change breaks the workload, restore the previous value

Values that require confirmation and backup before change:

- `INITDATA`
- existing pod annotations for confidential containers
- Trustee-related secrets already used by the cluster
- existing Trustee configuration that exposes KBS resources

## Policy Roles

- `Kata policy`: runtime image allowlist
- `Trustee image security policy`: image signature enforcement
- `Intel Trust Authority policy`: attestation evidence validation

Constraint:

- do not use ITA policy as the image allowlist
- do not use Kata policy as the signature verifier

## Required Inputs

- image signing public key: `cosign.pub`
- Trustee URL: `<trustee-url>`
- Trustee CA or service certificate: `<trustee-ca-or-service-cert>`
- approved image digests:
  - `icr.io/acme/payments-api@sha256:...`
  - `icr.io/acme/sidecar-helper@sha256:...`

## Required Artifacts

- KBS secret: `img-sig`
- file: `security-policy-config.json`
- KBS secret: `security-policy`
- file: `policy.rego`
- file: `initdata.toml`
- file: `initdata.txt`
- expected measured value: `PCR8_HASH`

## Procedure

### 0. Confirm and back up current cluster values

Before changing any existing cluster value:

1. identify the current value
2. show the proposed replacement to the user
3. get confirmation
4. save a backup that can be restored

Minimum backup scope:

- current `INITDATA` value if it already exists
- current workload YAML if pod annotations will change
- current Trustee secret definitions if they already exist
- current Trustee configuration if KBS resource exposure will change

Do not continue until backup is complete.

### 1. Create image-signing key secret

```bash
oc create secret generic img-sig \
  --from-file=pub-key=./cosign.pub \
  -n trustee-operator-system
```

### 2. Create Trustee image signature policy

Create `security-policy-config.json`:

```json
{
  "default": [
    {
      "type": "reject"
    }
  ],
  "transports": {
    "docker": {
      "icr.io/acme/payments-api:stable": [
        {
          "type": "sigstoreSigned",
          "keyPath": "kbs:///default/img-sig/pub-key"
        }
      ]
    }
  }
}
```

Create the KBS secret:

```bash
oc create secret generic security-policy \
  --from-file=osc=./security-policy-config.json \
  -n trustee-operator-system
```

Ensure Trustee exposes both resources:

```bash
oc edit kbsconfig <kbsconfig-name> -n trustee-operator-system
```

Required result:

```yaml
spec:
  kbsSecretResources:
    - kbsres1
    - img-sig
    - security-policy
```

### 3. Create Kata runtime policy

Create `policy.rego`:

```rego
package agent_policy

default CreateSandboxRequest := true
default DestroySandboxRequest := true
default GuestDetailsRequest := true
default StartContainerRequest := true
default StatsContainerRequest := true
default RemoveContainerRequest := true

default ExecProcessRequest := false
default ReadStreamRequest := false
default CopyFileRequest := false

default CreateContainerRequest := false

allowed_images := {
  "icr.io/acme/payments-api@sha256:1111111111111111111111111111111111111111111111111111111111111111",
  "icr.io/acme/sidecar-helper@sha256:2222222222222222222222222222222222222222222222222222222222222222"
}

CreateContainerRequest if {
  some storage in input.storages
  storage.source in allowed_images
}
```

Replace example digests with real digests.

Invariants:

- use digests only
- include every image used by the pod
- include init containers
- include sidecars

### 4. Build initdata

Create `initdata.toml`:

```toml
version = "0.1.0"
algorithm = "sha256"

[data]
"aa.toml" = '''
[token_configs]
[token_configs.coco_as]
url = "https://<trustee-url>"

[token_configs.kbs]
url = "https://<trustee-url>"
'''

"cdh.toml" = '''
socket = "unix:///run/confidential-containers/cdh.sock"
credentials = []

[kbc]
name = "cc_kbc"
url = "https://<trustee-url>"
kbs_cert = """
-----BEGIN CERTIFICATE-----
<trustee-ca-or-service-cert>
-----END CERTIFICATE-----
"""

[image]
image_security_policy_uri = "kbs:///default/security-policy/osc"
'''

"policy.rego" = '''
package agent_policy

default CreateSandboxRequest := true
default DestroySandboxRequest := true
default GuestDetailsRequest := true
default StartContainerRequest := true
default StatsContainerRequest := true
default RemoveContainerRequest := true
default ExecProcessRequest := false
default ReadStreamRequest := false
default CopyFileRequest := false
default CreateContainerRequest := false

allowed_images := {
  "icr.io/acme/payments-api@sha256:1111111111111111111111111111111111111111111111111111111111111111",
  "icr.io/acme/sidecar-helper@sha256:2222222222222222222222222222222222222222222222222222222222222222"
}

CreateContainerRequest if {
  some storage in input.storages
  storage.source in allowed_images
}
'''
```

If Trustee uses `insecure_http = true`, remove the `kbs_cert` block.

### 5. Encode initdata

```bash
cat initdata.toml | gzip | base64 -w0 > initdata.txt
```

### 6. Compute measured value

```bash
hash=$(sha256sum initdata.toml | cut -d' ' -f1)
initial_pcr=0000000000000000000000000000000000000000000000000000000000000000
PCR8_HASH=$(echo -n "$initial_pcr$hash" | xxd -r -p | sha256sum | cut -d' ' -f1)
echo "$PCR8_HASH"
```

Requirement:

- register the new expected measured value in the attestation reference path

Rule:

- any `initdata.toml` change requires a new expected measurement
- any `initdata.toml` change requires a restart of the `cloud-api-adaptor` DaemonSet

### 7. Apply initdata to workload

Preferred method:

- apply `initdata.txt` to a test workload annotation first
- do not replace cluster-wide `INITDATA` first if a narrower test path exists

Add this pod annotation:

```yaml
metadata:
  annotations:
    io.katacontainers.config.hypervisor.cc_init_data: "<contents-of-initdata.txt>"
spec:
  runtimeClassName: kata-remote
```

If initdata changed, restart the `cloud-api-adaptor` DaemonSet before testing.

## Verification

### Case 1

Input:

- unsigned image

Expected result:

- pod fails

### Case 2

Input:

- signed image
- digest not in `allowed_images`

Expected result:

- pod fails

### Case 3

Input:

- signed image
- digest in `allowed_images`

Expected result:

- pod starts

## Failure Mapping

- unsigned image starts:
  - Trustee image security policy is missing, wrong, or not reachable

- signed unapproved image starts:
  - Kata policy is missing, weak, or does not match actual image references

- approved image fails:
  - digest mismatch
  - missing sidecar or init-container digest
  - bad Trustee URL
  - bad KBS certificate
  - stale expected measurement

- workload fails after `initdata.toml` change:
  - expected measurement was not updated
  - `cloud-api-adaptor` DaemonSet was not restarted

- workload fails after replacing an existing cluster value:
  - restore the backup immediately
  - compare old and new `INITDATA`, annotations, secrets, or Trustee config

## Completion Criteria

Configuration is complete only if all are true:

- any overwritten cluster value was user-confirmed first
- backups exist for any replaced cluster value
- `img-sig` secret exists
- `security-policy` secret exists
- Trustee exposes both resources
- generated `KbsConfig` contains `img-sig` and `security-policy` in `spec.kbsSecretResources`
- `policy.rego` allows only approved digests
- `initdata.toml` contains `aa.toml`, `cdh.toml`, and `policy.rego`
- `initdata.txt` is applied to the workload
- expected measurement matches current `initdata.toml`
- `cloud-api-adaptor` DaemonSet was restarted after any initdata change
- verification cases 1, 2, and 3 all produce the expected result
