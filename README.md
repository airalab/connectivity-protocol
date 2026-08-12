# Sensors.social Protocol

## Goals

- One shared structure for Altruist firmware, connectivity layer and sensors.social UI.
- **Protobuf** on the wire (MCU-friendly; high performance, no JSON canonicalization).

## Data Flow Architecture

```
┌────────────┐                ┌─────────────────┐              ┌─────────────────┐
│  IoT       │   Signed       │  Connectivity   │              │  Backend/Map    │
│  Sensor    │──────────────> │  Layer          │────────────> │  Infrastructure │
│  Device    │   Envelope     │  (Pass-through) │              │                 │
└────────────┘                └─────────────────┘              └─────────────────┘
     │                              │                                  │
     │ Measures                     │ Validates                        │ Decodes
     │ Serializes                   │ Signature                        │ Processes
     │ Signs                        │ Forwards                         │ Stores
     └──────────────────────────────┴──────────────────────────────────┘
```

## Message Anatomy

```
┌────────────────────────────────────────────────────────────────────┐
│                         TELEMETRY MESSAGE                          │
├────────────────────────────────────────────────────────────────────┤
│                                                                    │
│  Metadata                                                          │
│  ┌────────────────────────────────────────────┐                    │
│  │ Owner: 4CvP46mxFm54eBbTMFayHK7...          │                    │
│  └────────────────────────────────────────────┘                    │
│                                                                    │
│  Payload (choose one: Urban OR Insight)                            │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                                                             │   │
│  │  Public Data (visible to everyone)                          │   │
│  │  ┌────────────┐ ┌────────────┐ ┌────────────┐               │   │
│  │  │ Temp: 21°C │ │ PM2.5: 8.1 │ │ Location   │  ...          │   │
│  │  └────────────┘ └────────────┘ └────────────┘               │   │
│  │                                                             │   │
│  │  Protected Data (encrypted for specific recipients)         │   │
│  │  ┌──────────────────────────────────────────┐               │   │
│  │  │ 🔒 Encrypted for City Authority          │               │   │
│  │  │    Contains: Noise levels, detailed GPS  │               │   │
│  │  └──────────────────────────────────────────┘               │   │
│  │  ┌──────────────────────────────────────────┐               │   │
│  │  │ 🔒 Encrypted for Device Owner            │               │   │
│  │  │    Contains: All sensor readings         │               │   │
│  │  └──────────────────────────────────────────┘               │   │
│  │                                                             │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                    │
└────────────────────────────────────────────────────────────────────┘
```

## Versioning

Breaking schema changes go in a new folder/package (`v2/`, `v3/`, …).  
There is no `schema_version` field inside `Message` — the path / `package sensors.social.v1` is the version.

## Signed Envelope

Telemetry message wrapped in a `SignedEnvelope` that provides:
- **`sensor_id`** — Unique device identifier (persistent across restarts)
- **`timestamp`** — UTC timestamp when measurement was taken (ISO 8601 format)
- **`nonce`** — Random value to prevent replay attacks (e.g., UUID v4)
- **`signature`** — Ed25519 signature
- **`message`** — The actual measurements data

## Encryption

Selective data sharing is supported through public and protected measurement sections.

### Privacy Models

```
┌─────────────────────────────────────────────────────────────────────┐
│  MODEL 1: Fully Public                                              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Public: [Temp, PM2.5, Location]        🌍 Everyone can see         │
│  Protected: [ ]                                                     │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│  MODEL 2: Fully Private                                             │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Public: [ ]                                                        │
│  Protected: [🔒 Encrypted(All sensors)]  🔐 Only owner can see      │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│  MODEL 3: Mixed (Most Common)                                       │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Public: [Temp, PM2.5, Approx Location]  🌍 Everyone can see        │
│  Protected: [                                                       │
│    🔒 Encrypted(Precise GPS, Noise) → City Authority                │
│    🔒 Encrypted(All sensors)        → Device Owner                  │
│  ]                                                                  │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Data Sharing Model

Sensors can implement flexible privacy by:
1. **Fully public**: Only populate the `public` array with Urban/Insight messages, leave `protected` empty
2. **Fully private**: Leave `public` empty, put all measurements in `protected` sections (ciphertext contains serialized UrbanProtected/InsightProtected)
3. **Mixed sharing**: Share basic metrics publicly, detailed metrics in protected sections
4. **Multiple recipients**: Create separate protected sections for different users/groups

### Encryption Process

```
┌─────────────────────────────────────────────────────────────────────┐
│              How Protected Data is Encrypted                        │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Step 1: Prepare Measurements                                       │
│  ┌────────────────────────────────────────────┐                     │
│  │ Urban: [Noise: 85dB, GPS: (51.5, -0.1)]    │                     │
│  └────────────────────────────────────────────┘                     │
│                    ▼                                                │
│  Step 2: Wrap in UrbanProtected                                     │
│  ┌────────────────────────────────────────────┐                     │
│  │ UrbanProtected {                           │                     │
│  │   protected: [Urban, Urban]                │                     │
│  │ }                                          │                     │
│  └────────────────────────────────────────────┘                     │
│                    ▼                                                │
│  Step 3: Serialize to Binary                                        │
│  ┌────────────────────────────────────────────┐                     │
│  │ [0x0A, 0x12, 0x08, 0x15, ...]              │  Protobuf bytes     │
│  └────────────────────────────────────────────┘                     │
│                    ▼                                                │
│  Step 4: Key Agreement (ECDH)                                       │
│  ┌──────────────────┐     ┌──────────────────┐                      │
│  │ Sensor Private   │  +  │ Recipient Public │  → Shared Secret     │
│  │ Key              │     │ Key              │                      │
│  └──────────────────┘     └──────────────────┘                      │
│                    ▼                                                │
│  Step 5: Encrypt with AEAD (XChaCha20-Poly1305)                     │
│  ┌────────────────────────────────────────────┐                     │
│  │ Ciphertext + Auth Tag                      │  🔒 Encrypted       │
│  └────────────────────────────────────────────┘                     │
│                    ▼                                                │
│  Step 6: Add to Protected Section                                   │
│  ┌────────────────────────────────────────────┐                     │
│  │ EncryptedData {                            │                     │
│  │   version: 1                               │                     │
│  │   algorithm: "xchacha20"                   │                     │
│  │   from: <sensor_pubkey>                    │                     │
│  │   nonce: <random_24_bytes>                 │                     │
│  │   ciphertext: <encrypted_data>             │                     │
│  │ }                                          │                     │
│  └────────────────────────────────────────────┘                     │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

1. **Create wrapper**: Place Urban/Insight measurements in UrbanProtected/InsightProtected wrapper
2. **Serialize**: Convert wrapper to protobuf binary format
3. **Key Agreement**: ECDH between sender's private key and recipient's public key (from `meta.owner`)
4. **Key Derivation**: HKDF-SHA256 to derive encryption key from shared secret
5. **AEAD Encryption**: Encrypt serialized protobuf with chosen algorithm
6. **Authentication**: 16-byte Poly1305 or GCM tag appended to ciphertext

### Supported Algorithms

```
  AES-GCM-256 (Hardware)  ████████████████████████████  2-3 GB/s
  XChaCha20-Poly1305      ██████                        ~600 MB/s
  
  ✅ Use AES-GCM on devices with hardware acceleration
  ✅ Use XChaCha20 on MCUs / mobile devices
```

### Access Revocation

Revocation only applies going forward.

A protected section, once published, is stored in IPFS and its hash is anchored in the
chain. A recipient who held the key keeps everything they were able to decrypt up to the
moment access was taken back. Rotating keys or dropping a recipient from future messages
withdraws access to what comes next — not to what has already been released.

Applications built on this format should present the distinction to the user, rather than
implying that sharing can be undone.

## Partial Decoding

Protobuf supports **zero-copy pass-through** at the connectivity layer:

```
┌─────────────────────────────────────────────────────────────────────┐
│                   Layer Responsibilities                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  FIRMWARE (Creates & Signs)                                         │
│  ┌────────────────────────────────────────────┐                     │
│  │ 1. Measure sensors                         │                     │
│  │ 2. Create Message                          │                     │
│  │ 3. Serialize to bytes                      │                     │
│  │ 4. Wrap in SignedEnvelope                  │                     │
│  │ 5. Sign with Ed25519                       │                     │
│  └────────────────────────────────────────────┘                     │
│                         │                                           │
│                         ▼                                           │
│  CONNECTIVITY LAYER (Pass-through)                                  │
│  ┌────────────────────────────────────────────┐                     │
│  │ 1. Verify Ed25519 signature    ✓           │                     │
│  │ 2. Check timestamp freshness   ✓           │                     │
│  │ 3. Forward envelop             →           │  (No parsing!)      │
│  │                                            │                     │
│  │ ❌ Does NOT decode Message                 │                     │
│  │ ❌ Does NOT parse measurements             │                     │
│  └────────────────────────────────────────────┘                     │
│                         │                                           │
│                         ▼                                           │
│  BACKEND/MAP (Full Processing)                                      │
│  ┌────────────────────────────────────────────┐                     │
│  │ 1. Deserialize Message                     │                     │
│  │ 2. Extract metadata                        │                     │
│  │ 3. Process public measurements             │                     │
│  │ 4. Try decrypt protected sections          │                     │
│  │ 5. Store in database                       │                     │
│  │ 6. Update map visualization                │                     │
│  └────────────────────────────────────────────┘                     │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```
