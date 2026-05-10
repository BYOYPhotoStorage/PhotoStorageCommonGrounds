# Task 03 — Credential Storage

> Read [`../mvp-implementation-overview.md`](../mvp-implementation-overview.md) first.

## Goal

Securely persist the user's B2 credentials on device using `EncryptedSharedPreferences`.

## Files you own

- `app/src/main/java/com/photobackup/app/data/PrefsStore.kt`
- (Optional) `app/src/test/java/com/photobackup/app/data/PrefsStoreTest.kt`

## Public contract

```kotlin
package com.photobackup.app.data

data class B2Credentials(
    val keyId: String,
    val applicationKey: String,
    val bucketName: String
)

class PrefsStore(context: Context) {
    fun saveCredentials(creds: B2Credentials)
    fun getCredentials(): B2Credentials?
    fun clearCredentials()
    fun hasCredentials(): Boolean
}
```

`getCredentials()` returns null if any of the three fields is missing or blank. `hasCredentials()` is a cheap boolean check used by `MainActivity` (Task 09) to decide routing.

## Implementation

Use the snippet from the architecture doc:

```kotlin
private val prefs = EncryptedSharedPreferences.create(
    context,
    "b2_credentials",
    MasterKey.Builder(context)
        .setKeyScheme(MasterKey.KeyScheme.AES256_GCM)
        .build(),
    EncryptedSharedPreferences.PrefKeyEncryptionScheme.AES256_SIV,
    EncryptedSharedPreferences.PrefValueEncryptionScheme.AES256_GCM
)
```

Keys: `key_id`, `application_key`, `bucket_name`.

`saveCredentials` writes all three with `apply()`. `clearCredentials` calls `prefs.edit().clear().apply()`.

## Constraints

- **Never log credentials.** No `Log.d(..., creds.applicationKey)` anywhere — not even in error paths. If you need to log a failure, log `creds.keyId` only (which is a public-ish identifier on B2).
- Constructor must accept any `Context` — most consumers pass `applicationContext`.
- Treat blank strings as "missing" so `getCredentials()` doesn't return a half-formed object.

## Acceptance criteria

- [ ] After `saveCredentials(creds)` and process restart (simulated by creating a new `PrefsStore` instance with the same context), `getCredentials()` returns the same `B2Credentials`.
- [ ] After `clearCredentials()`, `hasCredentials()` is false and `getCredentials()` is null.
- [ ] Inspecting the on-device file at `/data/data/com.photobackup.app/shared_prefs/b2_credentials.xml` shows encrypted blobs, not plaintext keys.
- [ ] No reference to `applicationKey` appears in any `Log.*` call.

## Out of scope

- Key rotation. Re-onboarding overwrites the stored values.
- Biometric protection on read.
- Multiple credential profiles.
