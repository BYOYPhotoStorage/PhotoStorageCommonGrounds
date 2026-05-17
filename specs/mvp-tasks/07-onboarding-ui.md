# Task 07 — Onboarding UI

> Read [`../mvp-implementation-overview.md`](../mvp-implementation-overview.md) first.

## Goal

A single-screen credential entry form. On "Connect", validate against B2 and persist credentials, then navigate to the gallery.

## Files you own

- `app/src/main/java/com/hriyaan/photostorage/ui/OnboardingActivity.kt` (overwrites the stub from Task 01)
- `app/src/main/res/layout/activity_onboarding.xml`
- Any Material/string resources you need (strings should land in `res/values/strings.xml` — Task 01 created the file; just append).

## Layout

Per the architecture doc:

```
LinearLayout (vertical, padding 16dp)
├── TextInputLayout
│   └── TextInputEditText: hint "B2 Application Key ID"     id=@+id/keyIdInput
├── TextInputLayout (passwordToggleEnabled)
│   └── TextInputEditText: hint "B2 Application Key"        id=@+id/appKeyInput, inputType=textPassword
├── TextInputLayout
│   └── TextInputEditText: hint "B2 Bucket Name"            id=@+id/bucketInput
├── MaterialButton: text "Connect"                          id=@+id/connectButton
├── ProgressBar (indeterminate, gone by default)            id=@+id/progressBar
└── TextView (visibility=gone)                              id=@+id/errorText
```

Use Material3 `TextInputLayout` styling. Reasonable spacing; this is a one-page form, not a design exercise.

## Behavior

```kotlin
class OnboardingActivity : AppCompatActivity() {
    private val prefsStore by lazy { (application as PhotoBackupApp).prefsStore }

    onClick(connectButton):
        1. read inputs; if any blank → show inline error, return
        2. disable button, show progress, hide errorText
        3. lifecycleScope.launch(Dispatchers.IO):
             val creds = B2Credentials(keyId, appKey, bucket)
             val client = S3ClientFactory.create(creds, S3Config.forBucket(bucket))
             val uploader = S3Uploader(client, bucket)
             val result = uploader.validateCredentials()
             uploader.close()
             withContext(Dispatchers.Main):
                 result.onSuccess {
                     prefsStore.saveCredentials(creds)
                     startActivity(Intent(this@OnboardingActivity, GalleryActivity::class.java))
                     finish()
                 }
                 result.onFailure { e ->
                     showError(messageFor(e))
                     re-enable button, hide progress
                 }
}
```

### Error-message mapping

| Cause                                   | Toast / errorText                                   |
|-----------------------------------------|-----------------------------------------------------|
| `S3Exception` 403 / `InvalidAccessKeyId`| "Invalid Application Key ID."                       |
| `S3Exception` 403 / `SignatureDoesNotMatch` | "Invalid Application Key."                       |
| `S3Exception` 404 / `NoSuchBucket`      | "Bucket not found."                                 |
| `IOException` / network                 | "Couldn't reach Backblaze. Check your connection."  |
| anything else                           | "Could not connect: ${e.javaClass.simpleName}"      |

Keep messages short. Don't include the raw stack trace. Don't surface `e.message` directly — some include the access key.

## Dependencies (by interface — see overview)

- `PrefsStore` (Task 03)
- `B2Credentials` (Task 03)
- `S3Config`, `S3ClientFactory`, `S3Uploader` (Task 06)
- `PhotoBackupApp.prefsStore` (Task 09 wires this — until then, you can construct `PrefsStore(applicationContext)` inline; Task 09 will replace the call)

## Constraints

- Don't construct an `S3Client` until the user taps "Connect" — building one is non-trivial work.
- Don't store credentials in `PrefsStore` until validation succeeds. (One save call, after success.)
- Make the Connect button non-clickable while a request is in flight. Re-enable on failure or after navigation.
- Do not log credentials. Even on error.

## Acceptance criteria

- [ ] Launching the activity shows three input fields and a "Connect" button.
- [ ] Tapping Connect with one or more empty fields shows an inline error and does not call B2.
- [ ] With valid credentials, the activity stores them via `PrefsStore` and starts `GalleryActivity`, then finishes itself.
- [ ] With invalid credentials, the activity shows an error matching the table above, re-enables the button, and does NOT save anything to `PrefsStore`.
- [ ] No instance of `creds.applicationKey` appears in any `Log.*` call or in any user-visible error string.

## Out of scope

- Bucket-region picker (DEFAULT_REGION is fine for MVP).
- "Test connection" without saving.
- Visual polish beyond Material defaults.
