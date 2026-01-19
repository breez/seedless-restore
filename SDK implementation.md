# SDK Implementation

## PRF (Pseudo-Random Function) Integration

### Overview
Each WebAuthn passkey is bound to a specific Relying Party Identifier (rpID), which can be a domain or application identifier. To enable cross-application passkey sharing using this SDK, we utilize a centralized rpID: `keys.breez.technology`.

### Cross-Platform Configuration
To allow multiple applications and websites to share the same passkey, the domain `https://keys.breez.technology` must serve three special configuration files:

### 1. Related Origins Configuration
**File Location:** `/.well-known/webauthn`

**Content-Type:** `application/json`

**Purpose:** Declares which origins are allowed to use the centralized rpID for WebAuthn operations.

**Configuration Example:**
```json
{
  "related_origins": [
    "https://keys.breez.technology",
    "https://app2.example.com",
    "android:apk-key-hash:B6:16:AD:FE:C5:C6:D3:4C:93:01:5B:4A:79:20:21:4E:62:43:AB:29:28:EE:34:9A:F2:46:55:4B:54:FC:42:DF",
    "ios:bundle-id:com.example.passkeyprf"
  ]
}
```

**Notes:**
- Web origins must use HTTPS
- Android apps are identified by APK key hash
- iOS apps are identified by bundle ID
- The centralized rpID (`keys.breez.technology`) must serve this file

### 2. Android Configuration (Asset Links)
**File Location:** `/.well-known/assetlinks.json`

**Purpose:** Declares digital asset links between the web domain and Android applications, enabling passkey sharing.

**Configuration Example:**
```json
[
  {
    "relation": [
      "delegate_permission/common.handle_all_urls",
      "delegate_permission/common.get_login_creds"
    ],
    "target": {
      "namespace": "web",
      "site": "https://keys.breez.technology"
    }
  },
  {
    "relation": [
      "delegate_permission/common.handle_all_urls",
      "delegate_permission/common.get_login_creds"
    ],
    "target": {
      "namespace": "android_app",
      "package_name": "com.example.passkeyprf",
      "sha256_cert_fingerprints": [
        "B6:16:AD:FE:C5:C6:D3:4C:93:01:5B:4A:79:20:21:4E:62:43:AB:29:28:EE:34:9A:F2:46:55:4B:54:FC:42:DF"
      ]
    }
  }
]
```

**Notes:**
- The Android application with package name `com.example.passkeyprf` must be signed with the certificate matching the specified SHA256 fingerprint
- Multiple Android apps can be listed by adding additional entries

### 3. iOS Configuration (Apple App Site Association)
**File Location:** `/.well-known/apple-app-site-association`

**Purpose:** Establishes a connection between the website and iOS applications for Universal Links and passkey sharing.

**Configuration Example:**
```json
{
  "applinks": {
    "details": [
      {
        "appIDs": [
          "TEAMID.com.example.passkeyprf"
        ],
        "components": [
          {
            "/": "/*",
            "comment": "Matches all paths"
          }
        ]
      }
    ]
  },
  "webcredentials": {
    "apps": [
      "TEAMID.com.example.passkeyprf"
    ]
  }
}
```

**Notes:**
- Replace `TEAMID` with your Apple Developer Team ID
- The `webcredentials` section is essential for passkey sharing with iOS apps
- The app must have the Associated Domains capability enabled with `webcredentials:keys.breez.technology`

### Integration with Partner Portal

The Partner Portal enables users to register their applications and websites for passkey sharing. Each user can add one or more Android apps, iOS apps, or websites, providing the necessary configuration information for each platform.

### Passkey Migration Considerations

#### 1. Cross-Platform Migration (CXP Support)
Passkey migration between different platforms (iOS, Android, and web) requires support for the CXP (Credential Provider) protocol:
- **iOS**: Currently supports CXP for exporting passkeys
- **Android**: Expected to support CXP in future versions (potentially Android 17)
- **Web**: CXP support varies by browser and platform

#### 2. Domain (rpID) Modification
The ability to change the domain (rpID) associated with a passkey varies by password manager:
- **Native Managers**: Neither iOS Password Manager nor Android's Google Password Manager support domain modification
- **Third-party Managers**:
  - **Bitwarden**: Supports CXP on iOS but does not support the PRF extension
  - **1Password**: Supports the PRF extension but does not support CXP, even on iOS
  - **KeePass**: Does not support either CXP or the PRF extension



## NOSTR Integration for Salt List Storage

To ensure reliable storage of salt lists across the network, we utilize NOSTR with redundancy measures:

### Breez-Managed Relay
Breez maintains a dedicated relay where authenticated users can publish events. To enable publishing to Breez's relay, users of the Breez SDK must:

1. **Register Public Key in Partner Portal**: Users of the SDK must add a NOSTR public key in the Breez Partner Portal to authorize publishing to the Breez-managed relay.

2. **NIP-42 Authentication**: Once the public key is registered, the app using the SDK can use the private key corresponding to the public key registered in the Partner Portal to authenticate using the [NIP-42 (HTTP Authentication)](https://github.com/nostr-protocol/nips/blob/master/42.md) protocol.

This authentication mechanism ensures secure and authorized access to the relay, guaranteeing at least one reliable storage location.

### NIP-65 Implementation
The SDK implements [NIP-65 (Relay List Metadata)](https://github.com/nostr-protocol/nips/blob/master/65.md) to discover and connect to active relays. The SDK retrieves a current list of recommended relays from Breez server. The client will update the event containing the list of relais when Breez server returns an update list and will also ensure that the salt events are published in all the relais from this list.

### Current Relay Configuration
The default relay list includes:
- `wss://relay.nostr.watch`
- `wss://relaypag.es`
- `wss://monitorlizard.nostr1.com`
- `wss://relay.damus.io`
- `wss://relay.nostr.band`
- `wss://relay.primal.net`

This multi-relay approach ensures data redundancy and improves availability across the NOSTR network.
