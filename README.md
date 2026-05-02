# kata_policy_guides

Small task-focused guides for Kata, Confidential Containers, Trustee, and related policy workflows.

Each task lives in its own folder. Each folder can contain:

- a human guide
- an AI execution guide

## Purpose

This repo is for operational guides that explain one policy task at a time.

The current guide covers:

- requiring signed images
- allowlisting exact image digests in Kata policy
- wiring that policy through initdata for Red Hat peer pods
- following the OpenShift Sandboxed Containers 1.12 model

## Prerequisites

These guides assume the platform already exists.

For the current guide, the minimum prerequisites are:

- Intel TDX already works
- Red Hat OpenShift Sandboxed Containers peer pods already work
- the target environment uses the OpenShift Sandboxed Containers 1.12 workflow
- Red Hat build of Trustee is installed
- Trustee uses Intel Trust Authority for attestation
- you can change Trustee configuration and workload manifests
- you know the exact image digests you want to allow
- you have the public key used to verify signed images

These guides are not product installation guides.

## Layout

- `require_signed_images/README.md`: human-oriented task guide
- `require_signed_images/AI.md`: AI-oriented execution spec

## Operating Rules

- treat existing cluster values as live state
- confirm before overwriting values like `INITDATA`
- back up existing values before changing them
- keep task scope narrow
- use exact image digests, not tags
