---
name: "implement-device-binding"
description: "Tie a user's account to one trusted device (activation + device binding) so stolen credentials cannot be used from another phone — right password, wrong device, blocked. This defeats remote account takeover, SIM swap, and token replay, and is stronger than SMS one-time passcodes (regulators are moving this way). At activation, create a hardware-backed device key in the Android Keystore / iOS Secure Enclave (non-extractable), prove it with key attestation verified server-side, and have the server bind the account to that specific device key. Afterward, sign sensitive requests/transactions with the device key and have the server verify they come from the bound device. Layer with biometrics (user presence) and step-up auth for high-risk actions. Critically, protect the activation and re-binding (new/lost phone) flows with strong identity proofing — that is the attacker's real target. Builds on the hardware-key-storage and attestation controls. Use for banking, payments, wallets, and any high-value account."
---

# Implement Activation & Device Binding

Device binding ties an account to one trusted device so that credentials alone are not enough from
anywhere else: an attacker with the stolen password and even the OTP is blocked because their phone is
not the bound device. It combines the hardware-key and attestation controls into a practical anti-fraud
flow, and it beats SIM swap and token replay.

## 0. Clarify before coding
- What must be bound: login sessions, high-value transactions, or the whole account?
- Do you have hardware-backed keys + attestation available (from earlier controls)?
- How will users legitimately move to a new phone (re-binding), and what is the recovery path?
- What identity proofing do you require at activation (the attacker's target)?

## 1. Why bind
- Stolen password/OTP used on another phone is rejected (right password, wrong device).
- Beats **SIM swap** (attacker's new SIM is not the registered device) and **token replay** (a stolen
  token reused from another device fails).
- Stronger than SMS OTP; regulators increasingly require device-level binding over SMS codes.

## 2. The activation + binding flow
1. At activation, the app creates a **device key** in secure hardware (Android Keystore / iOS Secure
   Enclave). The private key never leaves the hardware.
2. The hardware returns the **public key + attestation** (proving it is genuine, hardware-backed).
3. The app sends the public key + attestation to the server, with a **one-time activation code** that
   proves the right user.
4. The server **verifies the attestation and binds the account to that device key** (stores it).
5. Afterward, sensitive requests/transactions are **signed by the device key**; the server checks the
   signature comes from the **bound device** → allow, otherwise deny.

## 3. Make the bond unforgeable
- **Hardware-backed, non-extractable** device key (Keystore / Secure Enclave), so it cannot be copied,
  even on a rooted/jailbroken phone.
- **Attestation at activation, verified server-side** — do not trust the app's claim.
- **Sign sensitive actions** (login, transfers, adding a payee) with the device key so each important
  action is provably from the bound device.
- **Layer** with biometrics (user presence to use the key) and **step-up** for high-risk actions.

## 4. Protect activation and re-binding (the real target)
- A strong bond is worthless if activating a new device is easy — the attacker just activates their own.
  Require **strong identity proofing** at activation (and again at re-binding).
- Handle new phone / lost phone with a controlled re-binding flow that **unbinds the old device**, with
  strong verification; do not make it so easy it becomes the bypass, nor so rigid it locks out users.

## 5. Server-side enforcement
- Record the bound device key per account; reject requests/tokens not signed by it.
- Detect and alert on binding mismatches (new device attempts) and require re-activation.

## Final checklist
- [ ] Device key created in secure hardware at activation; private key non-extractable.
- [ ] Attestation verified server-side; account bound to that specific device key.
- [ ] Sensitive requests/transactions signed by the device key; server verifies the bound device.
- [ ] Biometrics + step-up layered for high-risk actions.
- [ ] Activation and re-binding use strong identity proofing; old device unbound on re-bind.
- [ ] Server rejects unbound-device access and alerts on mismatches.

## Anti-patterns to refuse
- Relying on SMS OTP alone (interceptable, not device-bound).
- Binding to a soft identifier (device name, ad ID) instead of a hardware-backed key.
- Trusting a client-side "this is the bound device" claim without server-verified attestation/signatures.
- Easy, weakly-verified activation or re-binding (the attacker just enrolls their own device).
- Not signing sensitive transactions with the device key (only binding the login).
