# Thunderbird Key Backup and Recovery Strategy

*Draft proposal version 2 — April 2026*

## Overview

This document proposes a new backup and recovery mechanism for cryptographic keys managed by Thunderbird. It describes a design that does not yet exist in Thunderbird and is intended as a basis for discussion and implementation planning. The primary goal is to allow full recovery of all application private keys using only two things: access to the user's email accounts and a recovery code saved by the user.

The document additionally proposes Cross-Device Trust Sync as an optional extension that reuses the infrastructure established for backup. This feature has lower priority and would be implemented separately. It is described here because the backup architecture directly enables it, and decisions made during backup design affect how it could be implemented.

---

## How It Works — A Brief Summary

Thunderbird generates a one-time 256-bit random seed, encoded as a 64-uppercase-character Passphrase using a custom hexadecimal alphabet (`0–9 H K N P W X`). The characters have been chosen to avoid characters that look similar to digits. The Passphrase is presented to the user in a human-friendly form suitable for writing down, which we'll describe as the user's Recovery Code.

This Passphrase is used to symmetrically encrypt a freshly generated post-quantum OpenPGP recovery keypair. This asymmetric keypair is introduced as an intermediate layer so that Thunderbird can create backups automatically using only the public key, without ever requiring the Passphrase to be present on the system during normal operation. The encrypted keypair and the encrypted backups of all application private keys are stored on the user's IMAP servers. The backups are signed before encryption using a signing subkey whose secret component remains on the local device. Recovery requires only the saved Recovery Code and access to the email account: the Recovery Code yields the Passphrase, which decrypts the recovery keypair, which decrypts the backup archives, and the signatures confirm that the archives have not been tampered with.

The backup infrastructure also enables an optional Cross-Device Trust Sync feature, described separately later in this document.

---

*The following analogy may help illustrate the concept.*

Each server holds two things: a small lockbox and a large lockbox. The large lockbox contains your email secrets. The small lockbox contains the key to the large one — and only your saved recovery code can open the small lockbox. To recover, you open the small lockbox with your recovery code, take out the key, and use it to unlock the large one.

Before sealing the large lockbox, Thunderbird stamps it with a personal seal that stays on your computer. Thunderbird can later verify the seal is genuine — so if someone tampers with the box's contents, the broken seal will be detected and the box rejected.

As an optional extension, the same seal and the same servers could also be used to share trust decisions between your devices — described later in this document.

In technical terms: the small lockbox corresponds to `recovery-privkey-encrypted`, which contains `recovery-privkey`, the large lockbox corresponds to the encrypted backup archive, and the personal seal corresponds to `data-signkey`.

---

## Key Concepts and Terminology

| Term | Description |
|---|---|
| **Recovery Code** | The human-friendly presentation of the Passphrase, structured for safe transcription; consists of 16 data groups and one checksum line, each formatted as 5 characters — a group identifier followed by 4 Passphrase characters — using the `0–9 H K N P W X` alphabet plus the checksum line identifier `M` |
| **Passphrase** | 64 characters from the `0–9 H K N P W X` alphabet, encoding 256 bits of entropy; used as the passphrase for OpenPGP symmetric encryption; never shown to the user directly — the user-facing form is always the Recovery Code |
| **recovery-pubkey** | Post-Quantum Cryptography Recovery Public Key; used exclusively for encrypting backups |
| **recovery-privkey** | Post-Quantum Cryptography Recovery Private Key; the corresponding secret key, never stored in plaintext; contains two subkeys: a PQC encryption subkey and a signing subkey |
| **data-signkey** | The signing subkey of `recovery-privkey`; its secret component is stored locally on the device and used to sign backup archives |
| **recovery-privkey-encrypted** | Symmetrically Encrypted PQC Recovery Private Key; the full `recovery-privkey` (including both subkeys) encrypted with the Passphrase via OpenPGP symmetric encryption |
| **Application Keys** | Private key material managed by Thunderbird that is subject to backup; this includes OpenPGP keypairs, S/MIME private keys, and optionally other secrets Thunderbird chooses to include |
| **recovery-pubkey-id** | The fingerprint of the currently active `recovery-pubkey`; embedded in the Subject of the Recovery Message on the IMAP server |
| **Recovery Message** | An email message stored on each backup-enabled IMAP server with Subject `thunderbird-recovery <recovery-pubkey-id>` and `recovery-privkey-encrypted` as its payload; combines key identification and encrypted key material in a single artifact |
| **Device UUID** | A random unique identifier generated once per device during initial setup and stored locally; used to identify the device's backup archive on the IMAP server |

---

## Initial Setup (Performed Once)

### Step 1: Generate the Passphrase and Recovery Code

Thunderbird generates 256 bits of cryptographically random entropy and encodes it as a 64-character Passphrase using the alphabet `0 1 2 3 4 5 6 7 8 9 H K N P W X`. This alphabet is used throughout — for the Passphrase and for the Recovery Code — for consistency. The letters `H K N P W X` replace the standard hex digits `A–F` with visually unambiguous alternatives, chosen to avoid handwriting confusion between letters and digits.

The Passphrase is then formatted into a Recovery Code for human transcription: 17 lines of 5 characters each. The 17 lines consist of 16 data groups and one checksum line. Each data group is identified by its own alphabet symbol in the first character position, making entry order-independent — Thunderbird slots each group by reading its identifier. The checksum line is identified by `M`, a character outside the 16-symbol alphabet, and contains a 16-bit checksum (first 16 bits of SHA256 of the entropy) encoded as 4 alphabet symbols. It is placed first so the format is easily recognisable when a paper backup is unfolded.

Example layout:
```
M XXXX
0 XXXX
1 XXXX
...
X XXXX
```

The screen displays all 17 lines simultaneously. A copy button places a compact single-line version on the clipboard, suitable for pasting into a password manager: 17 five-character tokens separated by spaces, with no internal space within each token (e.g. `MXXXX 0XXXX 1XXXX ... XXXX`).

The user is instructed to record the Recovery Code and store it securely. This is the only time it is displayed. The recommended storage method is writing it down on paper. Users may also choose to store it in a password manager, in which case the application should clearly inform them that the security of their backup is then bounded by the security of that password manager. This is the user's own informed decision to make.

Thunderbird also generates a random UUID for the device and stores it locally. This is a technical identifier used internally to distinguish backup archives from different devices on the IMAP server. The user does not need to be aware of or manage this value.

### Step 2: Encrypt Using the Passphrase

The 64-character Passphrase generated in Step 1 is used directly as the passphrase for OpenPGP symmetric encryption. OpenPGP tools expect a string passphrase rather than raw bytes, and the Passphrase satisfies this requirement.

To recover the Passphrase from the Recovery Code, Thunderbird strips the group identifiers, discards the checksum line, and concatenates the 4 payload characters from each of the 16 data groups.

For example, the recovery code `M4884 00123 14567 289HK 3NPWX 40123 54567 689HK 7NPWX 80123 94567 H89HK KNPWX N0123 P4567 W89HK XNPWX` results in this passphrase: `0123456789HKNPWX0123456789HKNPWX0123456789HKNPWX0123456789HKNPWX`.

### Step 3: Generate the Recovery Keypair

Thunderbird generates a fresh OpenPGP keypair with two subkeys: a post-quantum encryption subkey and a signing subkey (`data-signkey`). This keypair is generated independently using random entropy — it is not derived from the Passphrase. The keypair is never used for email communication. Its roles are exclusively:

- **Encryption subkey:** encrypting and decrypting backup archives
- **Signing subkey:** signing backup archives to allow tamper detection during recovery

The secret component of `data-signkey` is stored locally on the device under normal Thunderbird key protection, as it is needed regularly for signing operations. The remainder of `recovery-privkey` is not kept locally in plaintext.

### Step 4: Encrypt the Recovery Private Key

`recovery-privkey` is encrypted using OpenPGP symmetric encryption with the Passphrase. The resulting ciphertext, `recovery-privkey-encrypted`, is stored on the local machine. The plaintext `recovery-privkey` and the Passphrase are then discarded from memory.

### Step 5: Store Keys

The following are stored persistently on the local machine:

| Item | Location |
|---|---|
| `recovery-privkey-encrypted` | Local machine + all backup-enabled IMAP servers (inside the Recovery Message) |
| `recovery-pubkey` | Local machine |

On each backup-enabled IMAP server, Thunderbird stores a Recovery Message with Subject `thunderbird-recovery <recovery-pubkey-id>` and `recovery-privkey-encrypted` as its payload. This single message serves as both the key identifier and the encrypted key store.

---

## Regular Backup Process (Automated)

### Backup Process

For data isolation purposes, Thunderbird stores backup archives separately per account, keeping secrets from different contexts — for example personal and work accounts — in their respective locations.

Thunderbird manages one backup archive per device per IMAP account. Only keys whose OpenPGP UID or S/MIME subject address matches the email address of that account are included. A key associated with multiple accounts may therefore appear in multiple archives across different IMAP servers. Keys that cannot be attributed to any configured account are only backed up on the designated Trust Sync server, if one is configured.

For each backup-enabled IMAP account, Thunderbird periodically performs the following steps:

1. Collects all application private keys attributable to this account that are stored on this device.
2. Signs the archive using `data-signkey`.
3. Encrypts the signed archive using `recovery-pubkey` via OpenPGP public-key encryption.
4. Uploads the encrypted archive to the backup folder on that account's IMAP server as an email message with the Subject `backup <device-uuid>` and the current timestamp as the message date. Thunderbird may clean up older messages with the same Subject.
5. Ensures the Recovery Message is present on that IMAP server, creating or updating it if necessary.

Each backup-enabled IMAP server always holds:
- The Recovery Message (Subject: `thunderbird-recovery <recovery-pubkey-id>`, payload: `recovery-privkey-encrypted`)
- One encrypted backup archive per enrolled device, containing only the keys attributable to that account

The designated Trust Sync server additionally holds one encrypted backup archive per enrolled device containing all secret keys from all accounts, including unattributed keys.

No plaintext key material ever leaves the local machine.

---

## Recovery Process

**Prerequisites:** Access to each email account whose backups are needed for recovery, and the saved Recovery Code.

1. Thunderbird scans the backup folders of all accessible IMAP accounts for Recovery Messages and backup archives. It identifies the active `recovery-pubkey-id` from the Subject of the Recovery Message and downloads `recovery-privkey-encrypted` from its payload.
2. The user enters their Recovery Code.
3. Thunderbird derives the Passphrase from the Recovery Code by stripping group identifiers and the checksum line, then concatenating the 16 groups of 4 payload characters.
4. Thunderbird verifies that the checksum matches. If a failure is detected, it informs the user and stops the recovery operation.
5. Thunderbird decrypts `recovery-privkey-encrypted` using the Passphrase, obtaining `recovery-privkey` in memory.
6. Thunderbird decrypts each backup archive using `recovery-privkey`.
7. Thunderbird verifies the signature on each archive using the public component of `data-signkey`, extracted from `recovery-privkey`. Any archive that fails verification is rejected.
8. Thunderbird merges the application private keys from all verified archives, restoring the full key set across all previously enrolled devices and accounts.
9. `recovery-privkey` and the Passphrase are erased from memory.
10. `recovery-privkey-encrypted` is retained on the local machine for future backup cycles.

After recovery, Thunderbird resumes normal operation. No re-enrollment or re-generation of keys is required.

---

## Optional Proposal: Cross-Device Trust Sync

> **Note:** This section describes an optional feature proposal that reuses the backup infrastructure. It has lower priority than the backup mechanism and would be designed and implemented separately. The terminology specific to this proposal is defined within this section.

| Term | Description |
|---|---|
| **Trust Sync Blob** | A signed, unencrypted OpenPGP message stored on the IMAP server containing the user's trust decisions for a specific correspondent |

### Purpose

The backup infrastructure established in this document — IMAP-based storage, the recovery keypair, and the `data-signkey` — can also serve a second purpose: synchronising trust decisions across the user's devices. When a user marks a correspondent's public key as verified or makes another trust decision in Thunderbird, that decision is local to the device it was made on. Cross-Device Trust Sync would allow these decisions to be shared with other devices belonging to the same user, authenticated using `data-signkey`, and adopted automatically without requiring any direct device-to-device communication.

**Cross-Device Trust Sync is disabled by default.** Because Trust Sync Blobs are stored unencrypted, this feature requires explicit opt-in. The user must designate exactly one IMAP account for Trust Sync storage and acknowledge the privacy implications described below before the feature is activated.

### Privacy Considerations

Trust Sync Blobs are stored unencrypted on the designated IMAP server. The following information is therefore visible to anyone with administrative access to that server, or to any party who gains unauthorised access to the account:

- The email addresses of all correspondents for whom the user has made trust decisions
- The public keys of those correspondents
- The nature and timing of trust decisions (e.g. that a key was marked as verified or revoked)

The designated IMAP server effectively learns a subset of the user's social graph — specifically those correspondents whose keys the user has explicitly evaluated. This may be broader than what the server could infer from the inbox alone, since trust decisions may cover correspondents the user has not yet exchanged email with on that account.

A user who wishes to keep personal and professional contacts separate should avoid designating a work account as the Trust Sync server. Since Trust Sync data is stored unencrypted, this consideration is more significant than for backup archives, which are encrypted.

### Design Rationale: Why Not Local OpenPGP Signatures?

OpenPGP supports local (non-exportable) certifications on public keys, which could in principle serve as a trust signal. However, this approach was deliberately not adopted for Cross-Device Trust Sync. Trust decisions are explicit, structured data — they include a trust level, a timestamp, and potentially a revocation — and mapping these semantics onto the presence or absence of a key signature is imprecise. More critically, synchronising trust state via signature presence and revocation on public keys requires relying on correct and consistent handling of those states across all clients involved — something we prefer not to depend on. The Trust Sync Blob model treats trust decisions as first-class data with explicit versioning and revocation support, and allows greater flexibility by carrying arbitrary structured metadata about a trust decision independently of the OpenPGP key format.

Each Trust Sync Blob covers all trust decisions for a single correspondent, identified by email address. It is stored on the user's designated Trust Sync IMAP server as an RFC 2822 message in a dedicated folder. Blobs are signed using `data-signkey` but are not encrypted, allowing any Thunderbird instance with knowledge of `data-signkey`'s public component to verify and read them without additional key material.

Each blob contains:

- The correspondent's email address
- All current trust decisions for that correspondent (including explicit revocations of prior decisions)
- A copy of all relevant public keys for that correspondent
- A monotonically increasing version integer, scoped per correspondent
- The signature, whose creation timestamp serves as the authoritative update time

The Subject header of the IMAP message contains the correspondent's email address in plaintext, consistent with the privacy tradeoff described above.

### Producing a Trust Sync Blob

When the user makes or changes a trust decision for a correspondent on any device:

1. Thunderbird collects all current trust decisions for that correspondent into a single blob.
2. The version integer is incremented by one relative to the locally cached version.
3. The blob is signed with `data-signkey`.
4. The signed blob is uploaded to the IMAP Trust Sync folder, replacing the previous blob for that correspondent.

### Consuming a Trust Sync Blob

When a secondary device checks for updates:

1. Thunderbird scans the Trust Sync folder on the IMAP server.
2. For each blob found, it checks the version integer against its local cache.
3. If the server version does not exceed the locally cached version, the blob is ignored.
4. Otherwise, Thunderbird verifies the signature using the public component of `data-signkey`. If verification fails, the blob is rejected.
5. All trust decisions in the blob are applied locally, including any trust removals.

### Rollback Protection

Each Thunderbird instance caches the last-seen version integer per correspondent. An adversary with IMAP access could replace a current blob with an older one, but Thunderbird will reject any blob whose version integer does not exceed the locally cached value. The older blob is ignored rather than applied.

---

## Security Properties

**Backup:**
- **Passphrase entropy:** 256 bits; quantum-resistant for symmetric encryption contexts.
- **Recovery keypair:** Post-quantum OpenPGP keypair; resistant to attacks by quantum computers.
- **No single point of failure:** Loss of the local machine is recoverable via IMAP + Recovery Code. Each account's keys can be recovered independently from that account's IMAP server. A designated Trust Sync server enables full recovery of all keys across all accounts from a single server. Compromise of any IMAP server does not expose plaintext keys.
- **`recovery-privkey` is never stored in plaintext:** It exists in memory only during backup decryption and is cleared immediately afterward.
- **Backup integrity:** Each backup archive is signed with `data-signkey` before encryption, allowing tampering to be detected during recovery.
- **Backup keypair isolation:** `recovery-pubkey`/`recovery-privkey` are never used for email communication, limiting their exposure and attack surface.
- **Per-account key scoping:** Each IMAP server only receives backup archives containing keys attributable to that account. Keys that cannot be attributed to any configured account are only backed up on the designated Trust Sync server, if one is configured.

**Cross-Device Trust Sync (optional proposal):**
- **Trust Sync authenticity:** Trust Sync Blobs are signed with `data-signkey`; a secondary device will not accept any trust decision that cannot be verified against that key.

---

## What the User Must Safeguard

| Item | Where | Consequence if lost |
|---|---|---|
| Recovery code (17-line code) | Written on paper, or stored in a password manager | Cannot decrypt `recovery-privkey-encrypted` → cannot recover any backup |
| Access to each email account | Email provider | Keys attributable to that account cannot be recovered, unless a Trust Sync server is configured and accessible |
| Access to the designated Trust Sync server (if configured) | Email provider | Unattributed keys and the full cross-account backup cannot be recovered |

Loss of the recovery code or loss of all account access each independently prevent recovery from backup. Loss of individual accounts reduces the set of recoverable keys to those backed up on the remaining accessible accounts. If no Trust Sync server is configured, keys that cannot be attributed to any account are not backed up and cannot be recovered.

---

## Appendix A: Re-Enrollment After Loss of Recovery Code *(requires further design)*

> **Note:** This section identifies a necessary feature and its key design challenges. The exact mechanism requires a dedicated design pass and is not yet fully specified.

If the user loses their recovery code but still has a working local Thunderbird installation, no data loss has occurred — all key material remains intact on the device. However, the backup on the IMAP server can no longer be decrypted in a recovery scenario, so the user should promptly re-enroll with a new recovery keypair and a new recovery code.

### Server-Side Key Identifier

To coordinate key rotation across devices, the active `recovery-pubkey-id` is embedded in the Subject of the Recovery Message stored on each IMAP server. A device may only participate in backup and Cross-Device Trust Sync if its locally configured `recovery-pubkey` fingerprint matches the `recovery-pubkey-id` found in the Recovery Message. A mismatch causes the device to suspend participation and prompt the user to complete the re-enrollment flow.

### Re-Enrollment Flow

1. The user explicitly initiates re-enrollment on one device, acknowledging that all other devices must be manually rejoined afterward.
2. Thunderbird generates a new 256-bit entropy value and a new recovery code, a new `recovery-pubkey`/`recovery-privkey` keypair including a new `data-signkey`, and a new `recovery-privkey-encrypted`.
3. A new Recovery Message, a fresh backup archive, and all re-signed Trust Sync Blobs are uploaded to each IMAP server, replacing all previous content.
4. The user records the new recovery code.
5. On each secondary device, the user initiates a "use existing recovery code" workflow, entering the new recovery code. The device fetches the server-side key identifier, verifies it matches the newly derived `recovery-pubkey`, and resumes normal participation.

Any device that is not explicitly rejoined will detect the key mismatch and suspend backup and sync participation until the user completes the workflow on that device. A device that is permanently forgotten simply ceases to participate.

---

## Appendix B: Multi-Device Secret Sync *(requires further design)*

> **Note:** This section identifies likely user needs that will need to be addressed. No design decisions are made here.

### Multi-Device Sync via the Backup Mechanism

The backup architecture described in this document already technically enables multi-device synchronisation of secret keys, without requiring any additional infrastructure. When a user acquires a new secret key on one device, the application could notify the user on their secondary devices that a new key is available in the backup. The user would then initiate a sync by entering their recovery code on the secondary device, which decrypts the latest backup and imports the new key. This is a manual process, but it is a natural consequence of the existing architecture.

### Demand for Easier Sync

Users who are asked to enter a recovery code on a secondary device will predictably request a more convenient mechanism, such as QR code scanning. This is a reasonable expectation, but it introduces a significant risk: if the QR code encodes the recovery code directly, it could be photographed and uploaded to cloud storage, exposing the secret to unintended parties and defeating the security model.

For this reason, any future convenience mechanism for cross-device sync should never encode the recovery code directly in a scannable code. Instead, such a mechanism should use a temporary secure channel — the QR code would encode only ephemeral connection parameters for that channel, and the actual secret material would be transferred through it. The detailed design of such a mechanism is left for a future design pass.

---

## Appendix C: Additional Security Considerations

### User Trust and Opt-Out

Some users may feel uncomfortable uploading their secret key material to an IMAP server, even when it is strongly encrypted and the design has been carefully reviewed. They may not trust the encryption model, or they may be concerned about the consequences of an implementation bug. This is a legitimate position and should be respected.

The mechanism proposed in this document should therefore be presented as optional and convenient, not mandatory. Users who prefer not to use it should be able to turn it off entirely and rely on a classic manual backup approach — for example, exporting their secret keys to an encrypted file and storing it themselves.

### Recovery Code Display

A related concern applies to the initial display of the Recovery Code. Users may be tempted to photograph it for convenience. A photograph stored in cloud-synced storage would compromise the security of the entire backup system. Thunderbird should warn users about this risk and ask them not to use photographic capture.

### Implementation Quality Requirements

The archive creation and encryption pipeline is a security-critical code path. A bug that accidentally produces an unencrypted backup archive, or that transmits key material before encryption is applied, would silently compromise all of the user's secrets. The consequences of such a bug would be severe and potentially undetectable by the user.

This code path must be subject to thorough automated testing, including tests that explicitly verify the output is encrypted and contains no plaintext key material. Any change to the archive creation or encryption logic should be treated with the same scrutiny as a change to a cryptographic primitive.
