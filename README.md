# rbw Hardware Token Unlock

## Idea

I dont want to enter my long Master Password every time, but still keep everything secure.

By using my hardware token with its hmac-secret extension, i can get it to output a string that i can use as a password for my vault.

This project provides a custom `pinentry` helper for unlocking `rbw` with a
FIDO2 hardware token. When a token is present, the script derives the rbw unlock
secret from a resident FIDO2 credential using the `hmac-secret` extension. When
no token is present, it can fall back to a separate rbw profile that stores the
same secret.

The intended setup is:

- A FIDO2 resident credential for `rbw.local`.
- The credential has `hmac-secret` enabled.
- The credential is registered with `credProtect = UV_REQUIRED`, so PIN/UV is
  required for this credential even if global `alwaysUv` is disabled.
- `rbw` is configured to use this script as its pinentry program.

## Files

- `rbw-smart-pinentry`: Assuan-compatible pinentry helper used by `rbw`.
- `README.md`: Setup, usage, and security notes.

## Dependencies

Install the tools used by the script:

```sh
fido2-token
fido2-cred
fido2-assert
pinentry
expect
rbw
uwsm
notify-send
```

`notify-send` is optional. The script still works without desktop
notifications.

## Register The Token Credential

Create a protected resident credential:

```sh
DEVICE=$(fido2-token -L | head -n 1 | awk '{print $1}' | tr -d :)
TMP=$(mktemp -d)
trap 'rm -rf "$TMP"' EXIT

{
    head -c 32 /dev/urandom | base64 -w 0
    echo
    echo "rbw.local"
    echo "rbw vaultwarden"
    head -c 32 /dev/urandom | base64 -w 0
    echo
} > "$TMP/cred_param"

fido2-cred -M -r -h -v -c 3 -i "$TMP/cred_param" "$DEVICE" > "$TMP/cred"
cat "$TMP/cred"
```

Important flags:

- `-r`: create a resident/discoverable credential.
- `-h`: enable the FIDO2 `hmac-secret` extension.
- `-v`: request user verification during registration.
- `-c 3`: set `credProtect` to `FIDO_CRED_PROT_UV_REQUIRED`.

## Derive The Vault Password

After registration, derive the secret once and set the rbw/Bitwarden password to
the printed value:

```sh
DEVICE=$(fido2-token -L | head -n 1 | awk '{print $1}' | tr -d :)
TMP=$(mktemp -d)
trap 'rm -rf "$TMP"' EXIT

{
    head -c 32 /dev/urandom | base64 -w 0
    echo
    echo "rbw.local"
    echo -n "rbw-service-account-salt-32bytes" | base64 -w 0
    echo
} > "$TMP/assert.txt"

fido2-assert -G -r -v -h -i "$TMP/assert.txt" "$DEVICE" \
    | tr -d '\r' \
    | grep -E '^[A-Za-z0-9+/=]+$' \
    | tail -n 1
```

If multiple `rbw.local` resident credentials exist, clean up the old ones before
depending on resident discovery.

## Install The Pinentry Script

Make the helper executable and place it somewhere stable:

```sh
chmod +x rbw-smart-pinentry
mkdir -p ~/.local/bin
cp rbw-smart-pinentry ~/.local/bin/rbw-smart-pinentry
```

Configure `rbw` to use it. The exact file location depends on your rbw setup,
but the important setting is:

```toml
pinentry = "/home/thomasf/.local/bin/rbw-smart-pinentry"
```

## Fallback Vault

When no FIDO2 token is detected, the script uses:

```sh
RBW_PROFILE=fallback uwsm app -- rbw get "rbw vaultwarden"
```

That fallback item should contain the same unlock secret. If the fallback item
does not exist or the fallback vault cannot be unlocked, the script returns an
Assuan error and sends a desktop notification when available.

In my case, the fallback vault is just my main vault, that contains the password to my "rbw" vault. I created an Organization with my main vault and my rbw user as members and shared all my passwords with it. I highly recommend doing this with an Org, or have some other form of fallback, as loosing the token would mean loosing access to the vault.

## How Unlock Works

The script builds an assertion input:

```text
1. random 32-byte client data hash
2. relying party ID: rbw.local
3. blank credential ID because resident discovery is used
4. base64("rbw-service-account-salt-32bytes")
```

Then it runs:

```sh
fido2-assert -G -r -v -h -i assert.txt "$DEVICE"
```

The random client data hash changes on every unlock. The hmac salt is fixed, so
the derived password is stable for the same credential.

Conceptually, the token returns:

```text
HMAC-SHA-256(credential_secret, salt)
```

`libfido2` handles the CTAP2 PIN/UV protocol, ECDH session setup, salt
encryption/authentication, and response decryption.

## Security Notes

With `credProtect = UV_REQUIRED`, this specific credential should not be usable
without PIN/UV, even when global `alwaysUv` is disabled on the token. A no-PIN
assertion may return `FIDO_ERR_NO_CREDENTIALS` because the token hides protected
credentials until verification succeeds.

Important caveats:

- Delete old `rbw.local` resident credentials that were not registered with
  `credProtect=3`, or change the vault password so they no longer matter.
- If multiple `rbw.local` credentials exist, `fido2-assert -r` can return
  multiple assertions. The script extracts the last base64-looking output line,
  so duplicate credentials can select the wrong secret.
- Anyone with the token and the PIN can derive the vault password.
- Losing or resetting the token can lock you out unless you keep a recovery path.

## Maintenance Commands

List token capabilities:

```sh
fido2-token -I "$DEVICE"
```

List resident relying parties:

```sh
fido2-token -L -r "$DEVICE"
```

List credentials for `rbw.local`:

```sh
fido2-token -L -k rbw.local "$DEVICE"
```

Delete an old resident credential:

```sh
fido2-token -D -i '<credential id>' "$DEVICE"
```
