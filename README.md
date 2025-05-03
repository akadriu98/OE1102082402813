# Title

fmipmobile.icloud.com: improper token scoping allows unauthorized iCloud device removal via PET

---

## Summary

An issue exists in Apple’s Find My iPhone (FMiP) backend (`fmipmobile.icloud.com`) where a valid Apple ID and its associated PET (Private Endpoint Token) can be used to list, authorize, and remove devices from the iCloud account without triggering 2FA or ownership validation mechanisms.

This bypass enables unauthorized actors who have access to a session token (PET) to unlink offline devices from a user's iCloud account, effectively defeating Activation Lock protection in certain scenarios.

---

## Impact

An attacker who obtains a PET for an Apple ID can:

- Enumerate all devices tied to the account.
- Authorize per-device access without re-authentication.
- Remove offline devices silently via the `/remove` endpoint.
- Bypass the standard iCloud UI protections and security prompts.

This undermines the security model of Activation Lock and may enable unauthorized resale or misuse of stolen devices.

---

## Components Affected

- `https://fmipmobile.icloud.com/fmipservice/device/initClient`
- `https://fmipmobile.icloud.com/fmipservice/device/authForUserDevice`
- `https://fmipmobile.icloud.com/fmipservice/device/remove`

---

## What is Needed to Reproduce

- A valid Apple ID and password
- A captured PET (Private Endpoint Token) via session proxying or client instrumentation
- At least one offline iCloud-registered device tied to the account

*Note: This was tested using a controlled test account and device.*

---

## Steps to Reproduce

1. **Capture a valid PET** from a real session using a proxy (Burp/Zap) or MITM TLS proxy on a jailbroken iOS device.
2. Use the PET with the Apple ID to authenticate via Basic Auth to:

POST https://fmipmobile.icloud.com/fmipservice/device/initClient

css
Copy
Edit
Header:
Authorization: Basic base64(apple_id:PET)

css
Copy
Edit

3. Parse the JSON response to extract the `content[]` array (device list) and `serverContext`.

4. For each device:
- POST to:
  ```
  https://fmipmobile.icloud.com/fmipservice/device/authForUserDevice
  ```
  JSON Body:
  ```json
  { "authToken": "PET", "device": "<DEVICE_ID>" }
  ```
  → Returns a scoped `authToken` for the device.

5. Use that token to POST to:
https://fmipmobile.icloud.com/fmipservice/device/remove

css
Copy
Edit
Body:
```json
{
  "device": "<DEVICE_ID>",
  "authToken": "<SCOPED_TOKEN>",
  "serverContext": {...},
  "clientContext": { "appVersion": "7.0" }
}
Observe a statusCode: 200 response if the device is offline, indicating successful removal.

Expected Behavior
Device removal should require full authentication including 2FA or trusted device confirmation.

Tokens should be scoped per-device and per-session.

PET should not allow access to sensitive device control operations unless recently verified.

Actual Behavior
PETs allow unrestricted use of initClient, authForUserDevice, and remove.

Offline devices are removable with just a token and Apple ID.

No additional ownership or session validation is enforced.

PoC
A proof-of-concept is available in both PHP and Python and will be attached to this report as a ZIP file with the following files:

poc_device_remove.py – Python implementation

poc_device_remove.php – PHP implementation

demo_log.txt – Terminal output demonstrating successful removal

No personal account or production data is included.

Recommendations
Enforce device ownership revalidation on remove endpoint.

Scope PET tokens to specific endpoints and expiration windows.

Require reauthentication or 2FA for any unlinking/removal request.

Disclosure Policy
This report is submitted in good faith under Apple’s Responsible Disclosure policy. I am open to providing additional technical validation or test accounts if needed.

Thank you,
[Your Name or Alias]
