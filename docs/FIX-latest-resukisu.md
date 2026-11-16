# Fix: "Root My Pixel" not working with the latest ReSukiSU

## Symptom
On devices with the **latest ReSukiSU Manager** installed, the exploit and
late-load succeed, but root never actually works. The install log shows:

```
[!] Installed manager signature is not trusted; registration skipped: ...
```

## Root cause
The app trusted **one hardcoded ReSukiSU manager signature** (an allowlist):

`app/src/main/kotlin/com/alex193a/rootmypixel/utils/KernelSuInstallChecks.kt`

```kotlin
private val TRUSTED_MANAGER_SIGNATURES = setOf(
    ManagerSignature(
        size = 0x377,
        hash = "d3469712b6214462764a1d8d3e5cbe1d6819a0b629791b9f4101867821f1df64",
    ),
)
```

During install, `registerManager()` reads the real signature of the installed
ReSukiSU APK with `ksud debug get-sign <apk>` and compares it to that set.
The **latest ReSukiSU is signed with a different certificate**, so its
`size`/`hash` no longer match. `isTrustedManagerSignature()` returned `false`,
so the code hit `return` and **never ran**
`ksud kernel dynamic-manager set <size> <hash>`.

Without that registration, the kernel module does not recognise the installed
manager, so the manager cannot grant root to apps — i.e. root "doesn't work".

## What was changed
File: `app/src/main/kotlin/com/alex193a/rootmypixel/feature/install/InstallViewModel.kt`
(function `registerManager`).

The hard `return` on an unknown signature was removed. Behaviour now:

- If the signature **cannot be parsed** → still skip (nothing valid to register).
- If the signature **parses but is not in the known list** → log a notice and
  **register it anyway**, using the exact size/hash read from the installed
  manager APK via `ksud debug get-sign`.
- If the signature **is in the known list** → unchanged (registers as before).

This makes the app work with current and future ReSukiSU releases without
having to hardcode every new signing certificate, while still only ever
registering the signature of the manager the user actually installed.

```kotlin
val signature = KernelSuInstallChecks.parseManagerSignature(signatureResult.output)
if (signature == null) {
    appendLog("[!] Could not parse manager signature; registration skipped: ...")
    return
}
if (!KernelSuInstallChecks.isTrustedManagerSignature(signature)) {
    appendLog("[*] Installed manager signature not in known list; registering installed manager anyway (...)")
}
// ...falls through to: ksud kernel dynamic-manager set <size> <hash>
```

## Note on the security trade-off
The original allowlist was a trust boundary: only an exact, known ReSukiSU
build could be registered. The new behaviour is **trust-on-first-use** — it
trusts whichever ReSukiSU manager the user installed. That is reasonable for a
one-tap rooting tool (the user already chose and sideloaded that manager), but
it is a deliberate loosening of the original check.

### Stricter alternative (optional)
Instead of trust-on-first-use, add the latest ReSukiSU manager's signature to
the allowlist. On a device with the current manager installed, run:

```bash
ksud debug get-sign /path/to/ReSukiSU-manager.apk
# -> size: 0x<...>, hash: <64 hex>
```

Then add that pair to `TRUSTED_MANAGER_SIGNATURES` in
`KernelSuInstallChecks.kt`. This keeps the strict allowlist but must be updated
on every new ReSukiSU signing change.

## Unchanged / verified
- Device profiling and `profiles.json` (e.g. Pixel 7 Pro `cheetah`,
  `CP2A.260705.006`, KMI `android14-6.1`) are fine.
- `ksud` commands (`late-load --kmi`, `kernel dynamic-manager set/get`,
  `debug get-sign`, `debug info`) still match current ReSukiSU.
