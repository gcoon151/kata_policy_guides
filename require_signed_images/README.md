# How To Allow Only Approved Images In Red Hat Peer Pods

Use this when:

- you already have TDX working
- Red Hat build of Trustee is installed
- Trustee uses Intel Trust Authority for attestation
- you want peer pods to run only approved container images

## What You Are Configuring

There are three different policies.

1. `Kata policy`
This is the runtime policy inside the peer pod.
This is where you allow or deny container images.

2. `Trustee image security policy`
This is the signed-image policy.
This is where you require a valid image signature.

3. `Intel Trust Authority policy`
This checks TDX attestation evidence.
This is not the image allowlist.

If your goal is "only approved images run", do this:

1. Require image signatures with Trustee.
2. Allow only exact image digests with the Kata policy.

## What Success Looks Like

After you finish:

- an unsigned image fails
- a signed but unapproved image fails
- a signed and approved image starts

## Step 1: Create the Trustee Public Key Secret

This stores the public key used to verify signed images.

```bash
oc create secret generic img-sig \
  --from-file=pub-key=./cosign.pub \
  -n trustee-operator-system
```

## Step 2: Create the Trustee Image Signature Policy

Create `security-policy-config.json`:

```json
{
  "default": [],
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

Make sure Trustee uses both secrets:

```yaml
spec:
  kbsSecretResources: ["kbsres1", "security-policy", "img-sig"]
```

## Step 3: Create the Kata Policy

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

Replace the example digests with your real digests.

Important:

- use digests, not tags
- include every image the pod uses
- include init containers and sidecars too

## Step 4: Create `initdata.toml`

Create `initdata.toml`:

```toml
version = "0.1.0"
algorithm = "sha384"

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

What this file does:

- `aa.toml`: attestation settings
- `cdh.toml`: Trustee settings and image signature policy location
- `policy.rego`: Kata runtime policy

## Step 5: Encode `initdata.toml`

Create the value you will put in the pod:

```bash
cat initdata.toml | gzip | base64 -w0 > initdata.txt
```

## Step 6: Calculate the Measured Hash

If `initdata.toml` changes, the measured value changes.

Calculate it:

```bash
hash=$(sha256sum initdata.toml | cut -d' ' -f1)
initial_pcr=0000000000000000000000000000000000000000000000000000000000000000
PCR8_HASH=$(echo -n "$initial_pcr$hash" | xxd -r -p | sha256sum | cut -d' ' -f1)
echo "$PCR8_HASH"
```

Add this new expected value to the reference values used by your attestation flow.

Simple rule:

- if you change `initdata.toml`, update the expected measured value too

## Step 7: Apply the Initdata to the Workload

Add this annotation to the pod:

```yaml
metadata:
  annotations:
    io.katacontainers.config.runtime.cc_init_data: "<contents-of-initdata.txt>"
```

## Step 8: Test

Test 3 cases.

### Test 1: unsigned image

Expected result:

- pod fails

### Test 2: signed image, but digest not in `allowed_images`

Expected result:

- pod fails

### Test 3: signed image, digest is in `allowed_images`

Expected result:

- pod starts

## Common Mistakes

### Mistake 1

Using the Intel Trust Authority policy as the image allowlist.

Do not do that.
Use the Kata policy for the image allowlist.

### Mistake 2

Using tags like `:latest`.

Do not do that.
Use image digests.

### Mistake 3

Forgetting sidecars or init containers.

If their digests are missing, the pod fails.

### Mistake 4

Changing `initdata.toml` and not updating the expected measured value.

That breaks attestation or resource release.

## Minimal Rule To Remember

- Trustee signed-image policy checks signature
- Kata policy checks allowed digest
- ITA policy checks TDX evidence

You need all three pointed at the right job.

## References

- Confidential Containers: Init-Data
  - https://confidentialcontainers.org/docs/features/initdata/
- Confidential Containers: Policies
  - https://confidentialcontainers.org/docs/attestation/policies/
- Red Hat OpenShift Sandboxed Containers 1.11: Deploying confidential containers
  - https://docs.redhat.com/en/documentation/openshift_sandboxed_containers/1.11/html-single/deploying_confidential_containers/index
- Red Hat OpenShift Sandboxed Containers 1.11: Deploying Red Hat build of Trustee
  - https://docs.redhat.com/en/documentation/openshift_sandboxed_containers/1.11/html-single/deploying_red_hat_build_of_trustee/deploying_red_hat_build_of_trustee
- Intel Trust Authority: Attestation policies
  - https://docs.trustauthority.intel.com/main/articles/articles/ita/concept-policies.html
