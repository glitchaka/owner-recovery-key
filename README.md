# owner-recovery-key

Reusable Rust library for privileged OWNER and RECOVERY authentication, elevation, recovery, key rotation, and one-time credential consumption.

SST is the first intended consumer, but the library should remain application-agnostic.

## Scope

`owner-recovery-key` answers two privileged questions:

> May this specific running application instance elevate to OWNER now?

and:

> If OWNER access has been lost, may this installation safely recover and establish new OWNER credentials?

It is deliberately separate from:

- `disk-seal`, which binds a copy to its physical storage medium;
- `user-activation`, which authorizes the immutable normal user for a specific installation.

## Credential classes

The model uses three distinct credential roles:

```text
OWNER KEY
RECOVERY KEY
LICENSE AUTHORITY KEY
```

### OWNER KEY

Provides privileged OWNER access to an authentic installation after a successful secure elevation procedure.

### RECOVERY KEY

Used only to recover OWNER control when the normal OWNER credential or administrative installation has been lost or rendered unusable.

### LICENSE AUTHORITY KEY

Separate from OWNER and RECOVERY credentials. It signs user activations/licenses and should not automatically be granted merely because a session is OWNER.

The three roles must not share the same private secret.

## One-time credential rule

OWNER and RECOVERY credentials are **single-use credentials**.

A successful use consumes the presented credential and rotates to a successor generation.

Example:

```text
OWNER-0001 -> CONSUMED
OWNER-0002 -> CONSUMED
OWNER-0003 -> ACTIVE
```

Likewise:

```text
RECOVERY-0001 -> CONSUMED
RECOVERY-0002 -> ACTIVE
```

A credential must never be marked consumed merely because it was presented.

## Elevation state machine

Presenting a valid OWNER credential does **not** immediately make the application OWNER.

Recommended state machine:

```text
USER
  |
  | valid OWNER credential presented
  v
OWNER_PENDING
  |
  | secure self-scan passes
  | challenge-response succeeds
  | successor state is safely committed
  v
OWNER
```

`OWNER_PENDING` is intentionally non-privileged. It exists only to complete the elevation protocol.

## Mandatory self-scan before OWNER

The application must not consider OWNER elevation complete until it has performed a self-integrity and environment scan.

The scan should be extensible, but SST's intended checks include:

- valid `disk-seal` identity;
- executable integrity measurement;
- integrity of security-critical modules/components;
- coherent installation identity;
- coherent credential generation and revocation state;
- detection of known/debugger state where feasible;
- detection of suspicious instrumentation or process injection where feasible;
- process/parent/session coherence checks where useful;
- exclusive elevation session;
- other application-specific integrity policy.

No implementation should claim that user-mode software can prove with certainty that it is completely unobserved. The goal is to reject detectable suspicious conditions and, more importantly, to ensure copied elevation material is not reusable elsewhere.

## Anti-copy elevation protocol

OWNER elevation must use an ephemeral challenge bound to the current installation and current running session.

Conceptually:

```text
challenge = HASH(
    installation_id
  || disk_seal_identity
  || executable_measurement
  || elevation_session_id
  || random_nonce
  || timestamp
  || owner_key_generation
)
```

The OWNER credential proves authorization for **that challenge**.

An elevation proof copied from one SST instance must not authorize another SST instance, another support device, or another later session.

There must be no reusable generic `owner_activation.dat` blob that can simply be copied between installations.

## Observation / instrumentation rule

If the self-scan detects suspicious debugging, instrumentation, injection, integrity failure, or another policy violation during `OWNER_PENDING`:

```text
OWNER ELEVATION ABORTED
OWNER KEY NOT CONSUMED
```

A failed, interrupted, or suspicious elevation must leave the presented OWNER credential usable.

The same rule applies to RECOVERY credentials.

## Exclusive elevation sessions

Only one elevation operation should be active for a given installation/session identity at a time.

Each attempt receives an ephemeral:

```text
elevation_session_id
```

Challenge data and privilege transition are scoped to that session.

## Credential consumption

The application itself performs credential destruction **only after it has legitimately reached the privilege state required to do so**.

The intended rule is:

> The shell cannot consume a privileged credential until the shell has passed its self-scan and completed the privilege transition that authorizes the consumption operation.

A credential is not merely deleted. The system should retain audit metadata while making the original secret cryptographically unrecoverable.

Conceptual consumed record:

```text
key_id       = OWNER-0003
generation   = 3
status       = CONSUMED
used_at      = ...
fingerprint  = SHA-256(...)
credential   = irrecoverable ciphertext or no recoverable secret material
```

A practical design may use cryptographic erasure: encrypt remaining secret material with a fresh ephemeral destruction key, securely discard that key, and retain only non-secret audit metadata.

The system must not pretend that overwriting a file alone invalidates pre-existing external copies. Reuse prevention comes from generation/state tracking and challenge binding.

## Transactional rotation

Credential rotation must be atomic from the application's point of view.

For OWNER:

```text
1. verify presented OWNER credential
2. enter OWNER_PENDING
3. perform secure self-scan
4. complete challenge-response
5. prepare successor OWNER credential/state
6. durably commit successor state
7. transition to OWNER
8. mark previous OWNER generation CONSUMED
9. make previous secret cryptographically unrecoverable
10. confirm success
```

If the successor cannot be committed, the old credential must not be consumed.

A crash or power loss must not leave the owner permanently locked out.

## Recovery flow

RECOVERY exists specifically to avoid permanent loss of control.

Recommended flow:

```text
RECOVERY KEY presented
        |
        v
RECOVERY_PENDING
        |
        | self-scan
        | challenge-response
        | recovery policy validation
        v
rotate/revoke lost OWNER state
create new OWNER credential
create successor RECOVERY credential
durably commit both
consume old RECOVERY credential
        |
        v
OWNER
```

A successful recovery may revoke a lost OWNER generation without invalidating unrelated user activations.

## Recovery independence

The overall system must never have a single physical point of failure.

In particular:

- loss of one OWNER device must not permanently remove OWNER access;
- loss of one RECOVERY copy must not be catastrophic if policy permits redundant offline recovery material;
- the central registry must not be the only source of truth;
- recovery procedures should support signed offline registry backups;
- recovery design must permit reconstruction after hardware replacement.

Whether a specific product binds a particular recovery artifact to `disk-seal` is a policy decision, but there must always be a documented path that avoids permanent owner lockout.

## Revocation and generations

Credential metadata should support at least:

```text
ACTIVE
CONSUMED
REVOKED
```

Example:

```text
OWNER-KEY-17   CONSUMED
OWNER-KEY-18   REVOKED
OWNER-KEY-19   ACTIVE
```

The application should reject any proof made with a credential generation known to be consumed or revoked.

## Persistent OWNER installation vs session elevation

Two concepts must remain separate:

1. **Installation ownership metadata** — this sealed installation is designated as an OWNER installation.
2. **Current session privilege** — this running process has successfully authenticated and completed OWNER elevation now.

Owning an installation must not automatically imply that every process instance is currently elevated.

## API direction

Conceptual Rust API:

```rust
let pending = owner_recovery_key::OwnerElevation::begin(
    owner_credential,
    installation_identity,
)?;

let scan = pending.self_scan(&policy)?;
let challenge = pending.challenge()?;
let proof = credential.respond(&challenge)?;
let owner = pending.complete(proof, scan)?;

owner.consume_presented_credential()?;
```

Recovery should expose a separate flow, not a flag on normal OWNER elevation.

Possible modules/traits:

```text
owner
recovery
credential
challenge
self_scan
integrity
rotation
revocation
state
storage
audit
crypto
```

## Security rules

1. Valid credential presentation alone never grants OWNER.
2. OWNER requires successful self-scan and challenge-response.
3. OWNER/RECOVERY proofs are session-bound and installation-bound.
4. Copied activation/elevation material must not be reusable.
5. Suspicious observation/instrumentation aborts elevation when detected.
6. Failed or aborted elevation does not consume the key.
7. Only a legitimately privileged shell may execute the credential-consumption operation.
8. OWNER and RECOVERY credentials are one-use and rotate on success.
9. The successor must be durably committed before the predecessor is consumed.
10. Reuse prevention depends on generation/state tracking, not file deletion alone.
11. OWNER, RECOVERY, and LICENSE AUTHORITY private keys remain separate.
12. A recovery path must exist that prevents permanent lockout of the legitimate owner.
13. Private signing/owner secrets must never be committed to application source repositories.

## Relationship to the SST security model

```text
disk-seal
    -> authentic physical copy / storage medium

user-activation
    -> immutable named user authorized for that installation

owner-recovery-key
    -> privileged OWNER elevation and disaster recovery
```

Together these form three independent layers: physical copy protection, user authorization, and privileged elevation/recovery.
