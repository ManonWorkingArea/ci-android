# ci-android

Shared GitHub Actions for the app repos in `ManonWorkingArea` — one place to sign Android APKs with
the self-use keystore, so a key change is one edit here plus the repo secrets.

## `sign-apk` (composite action)

Decodes the keystore from the caller's secrets, runs the caller's gradle packaging command with
`-Pandroid.injected.signing.*` (no `build.gradle` change), then verifies the APK certificate digest.

```yaml
      - uses: ManonWorkingArea/ci-android@v1
        with:
          gradle-command: ./gradlew --no-daemon :app:assembleDebug
          apk-path: android/app/build/outputs/apk/debug/app-debug.apk
          working-directory: android
          keystore-base64: ${{ secrets.ANDROID_KEYSTORE_BASE64 }}
          keystore-password: ${{ secrets.ANDROID_KEYSTORE_PASSWORD }}
          key-alias: ${{ secrets.ANDROID_KEY_ALIAS }}
          key-password: ${{ secrets.ANDROID_KEY_PASSWORD }}
```

Requires `actions/setup-java` and `android-actions/setup-android` (with `build-tools`) earlier in the
job — the action does not install a toolchain.

### Why the key is shared

Android only allows an in-place update when the new APK carries the **same** certificate as the
installed one. Debug builds from a runner are signed with a throwaway key generated per run, so they
cannot update an installed app — you would have to uninstall first (losing app data). One stable
self-use key for every app fixes that permanently.

### Rotating the key

1. Generate a new keystore, back up the file and passwords **outside** the machine.
2. `gh secret set ANDROID_KEYSTORE_BASE64/…` on every repo that uses this action.
3. Update `expected-sha256` in `action.yml`, commit, tag `v2`, and bump callers.

Losing the keystore means the already-installed apps can never be updated again — only uninstalled
and reinstalled.
