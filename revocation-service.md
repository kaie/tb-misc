# Proposal: IMAP-Based Revocation Escrow for OpenPGP Keys in Thunderbird

**Status:** Draft
**Date:** March 2026

---

## Problem Statement

When a Thunderbird user loses access to their OpenPGP secret key material (device failure, accidental deletion, forgotten passphrase, or device theft), they can no longer decrypt incoming encrypted email. Correspondents who have the user's public key will continue to send encrypted messages that the user cannot read. The standard remedy is to revoke the public key, signaling to correspondents and keyservers that the key should no longer be used for encryption.

However, generating a revocation certificate requires possession of the secret key. Without prior preparation, this is unsolvable.

Preparing a revocation certificate in advance and storing it on the user's local filesystem does not help if the entire device is lost or the disk is unrecoverable. Similarly, a backup of the secret key is useless if the password protecting it has been forgotten.

## Proposed Solution

### Overview

The proposal consists of two parts:

1. Thunderbird automatically stores an encrypted revocation certificate in the user's IMAP account. Decryption is possible only using a secret key that is held by the Thunderbird project.
2. The Thunderbird project operates a new helper service, to which encrypted revocation certificates can be submitted. The service requires that the owner of the email address associated with the key being revoked completes an email address verification workflow, and confirms twice that the revocation certificate should be published.

With this approach, neither party alone can trigger revocation. The mechanism requires that the user retains access to their email account.

In the specific scenario where a device is stolen and no other copies of the secret key are available, it serves as a last-resort revocation path — provided the adversary has not destroyed the revocation blob stored on IMAP. The limitations of this scenario are discussed in the Security Analysis section.

### Preparation Phase

For each OpenPGP keypair controlled by the Thunderbird client, it performs the following steps:

1. **Generate a revocation signature** using the available secret key material.
2. **Fetch the Thunderbird Revocation Service's current public key** from a well-known endpoint (e.g., `https://revocation.thunderbird.net/.well-known/revocation-service-key`).
3. **Construct a revocation blob** by encrypting the revocation signature together with a copy of the associated public key to the service's public key. The public key must contain a User ID with an email address, as the service requires this to perform email verification.
4. **Store the blob** as a message in a designated IMAP folder on the user's mail server. The message subject identifies the user key and service key by fingerprint (e.g., "encrypted revocation certificate, for user key $fingerprint encrypted to service key $fingerprint"), and the message date is the time the blob was created. This allows the client to identify and clean up stale blobs after rotation.

### Maintenance Phase

The client periodically (e.g., weekly) polls the service's key endpoint. If the service wishes to rotate to a new encryption key, the service publishes the new key. The client then re-generates a fresh revocation signature using the locally available secret key, encrypts it to the new service key, and uploads the updated blob to IMAP. The old blob (identifiable by the service key fingerprint in its subject) should be retained on IMAP for a reasonable transition period before deletion.

### Revocation Phase

When a user has lost their secret key and wishes to revoke, the following flow is triggered:

1. **User initiates revocation.** This could happen through a Thunderbird UI prompt (e.g., when the client detects it cannot decrypt because the secret key is not available, and the user fails to restore the secret key, it could ask the user whether they want to revoke their key), or through a web-based portal provided by the Thunderbird project.
2. **Client requests email verification.** The client (or the user via the web portal) requests a submission token from the Thunderbird Revocation Service, providing the email address. This request is rate-limited per email address.
3. **Service sends submission token.** The service sends an email to the provided address containing a one-time submission token.
4. **Client submits blob with token.** The client retrieves the blob from the IMAP folder and submits it to the service together with the submission token, or the user submits the message from the IMAP server through a web-based portal. Without a valid token, blob submissions are rejected.
5. **Service decrypts and sends first confirmation.** The service validates the token, decrypts the blob using its private key, extracts the revocation signature and the public key, and sends a first confirmation email to the email address found in the public key's User ID. This email contains a confirmation link and a cancellation link.
6. **User confirms immediately.** The user clicks the confirmation link. The service acknowledges receipt and informs the user that a second confirmation email will be sent in 24 hours.
7. **Time delay.** The service waits 24 hours.
8. **Service sends second confirmation.** The service sends a second email containing both a confirmation link and a cancellation link.
9. **User finalizes.** The user clicks confirm or cancel. If confirmed, the service publishes the revocation signature to the relevant keyservers (e.g., keys.openpgp.org). If cancelled, the revocation is aborted.

## Design Goals

The mechanism described above was designed to satisfy the following requirements:

**No secret material required at revocation time.** A user who has lost all key material must still be able to trigger revocation of their public key.

**Stateless service operation.** The Thunderbird project should not need to maintain a per-user database of escrowed revocation certificates. This eliminates operational burden, reduces the attack surface, and avoids creating a centralized target for mass revocation attacks.

**No unilateral revocation by Thunderbird.** Neither a compromised server, a malicious insider, nor an external legal order directed at the Thunderbird project should be sufficient to revoke a user's key. The project should not hold enough information to do so.

**Protection against opportunistic abuse.** An attacker who obtains a one-time snapshot of the user's IMAP mailbox (e.g., from a backup leak) should not be able to trigger revocation.

## Security Analysis

### Trust Separation

The IMAP server holds the encrypted blob but cannot decrypt it. The Thunderbird service holds the decryption key but has no blobs. A compromise of either system alone is insufficient to trigger revocation.

This eliminates the mass revocation risk inherent in a pure server-side escrow design, where compromise of the escrow server (or a legal order directed at the operator) could enable revocation of all escrowed keys at once. The service's decryption key should ideally be stored in a hardware security module (HSM) to ensure that a server compromise does not allow an adversary to extract and copy the key.

### Threat Scenarios

**One-time mailbox snapshot.** An attacker who obtains a copy of the user's IMAP data (e.g., from a backup, a forensic image, or a temporary credential leak) gains the encrypted blob. However, they cannot decrypt it without the service's private key. They also cannot request a submission token because they no longer have mailbox access. **Result: safe.**

**Temporary mailbox access.** An attacker who gains brief access to the user's mailbox (e.g., shoulder-surfed webmail session, briefly unlocked phone, short-lived session token) could retrieve the blob, request a submission token, submit the blob, and confirm the first verification email — all within a single session. However, the second confirmation email is sent 24 hours later. Unless the attacker can regain mailbox access after the delay, they cannot finalize the revocation. **Result: safe against short-lived access.**

**Persistent IMAP credential compromise.** An attacker with ongoing read/write access to the user's IMAP account can retrieve the blob, submit it to the service, receive the verification email, confirm revocation, and delete all evidence before the legitimate user notices. The time delay mitigates this only if the user accesses their mailbox through another channel during the delay window. **Result: vulnerable, but acceptable.** If an attacker persistently controls the user's mailbox, revocation of the OpenPGP key is a minor consequence relative to the broader compromise.

**Device theft with loss of access.** An adversary steals the user's device, gaining possession of the secret key. The user wants to revoke but no longer has access to the key material. If the user retains access to their email account (e.g., via webmail or another device), they can retrieve the blob and submit it to the service. However, the adversary — who may have cached IMAP credentials from the stolen device — could preemptively delete the blob from IMAP, or attempt to interfere with the verification flow. **Result: best-effort.** This mechanism provides a revocation path that would not otherwise exist, but cannot guarantee success against an adversary who actively sabotages it. The user should be advised to change their email password immediately upon discovering device theft, which would cut off the adversary's IMAP access and preserve the blob.

**Thunderbird service compromise.** An attacker who compromises the Thunderbird Revocation Service obtains the service's decryption key but no blobs. They would additionally need to compromise individual users' IMAP accounts to obtain blobs to decrypt. **Result: no mass revocation capability.**

**Legal or coercive pressure on Thunderbird.** A legal order compelling the Thunderbird project to revoke a specific user's key cannot be fulfilled, because the project does not hold the encrypted blob. The project can only decrypt blobs that are submitted to it. **Result: resistant to compelled revocation.**

**Denial-of-service through submission flooding.** Because the service's encryption key is public, an adversary can construct valid-looking revocation blobs for fabricated keys tied to any email address. Without countermeasures, an adversary could flood the service with such blobs to trigger rate limits and block the legitimate user's submission. This is mitigated by requiring email verification before blob submission: the user must first request a submission token (step 2), which is delivered to the email address. Only submissions accompanied by a valid token are processed. Since the adversary cannot intercept the token without mailbox access, they cannot submit blobs at all. The rate limit is applied only to the token request step (per email address), which is lightweight and does not involve any cryptographic processing or blob handling. **Result: mitigated.**

## Limitations

**Designed for key loss, not key compromise.** This mechanism helps users who have lost access to their secret key material. It does not attempt to detect or respond to situations where an adversary has obtained a copy of the user's secret key.

**Requires functioning email delivery.** If the user's email provider is unreachable or the account is deleted, the verification email cannot be delivered and revocation cannot proceed through this mechanism.

**Does not protect against persistent mailbox compromise.** As analyzed above, an attacker with ongoing IMAP access can execute the full revocation flow. This is a deliberate tradeoff for simplicity and statelessness.

**Does not protect against blob destruction.** An adversary with IMAP write access can delete the blob, eliminating the revocation path. However, the client can re-upload the blob.

**Depends on Thunderbird service availability.** The service must be online to decrypt blobs and process revocation requests. However, the service is stateless and simple, making it straightforward to operate with high availability.

**Client must be running periodically.** The maintenance phase (re-encryption on key rotation) requires the client to run and have access to the secret key. Users who stop using Thunderbird will eventually have stale blobs that may become undecryptable after multiple service key rotations.
