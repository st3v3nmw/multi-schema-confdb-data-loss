# Confdb multi-schema data loss

> **Fixed.** This bug was resolved in snapd on June 16, 2025 in commit
> [4ebded6](https://github.com/canonical/snapd/commit/4ebded6c2ee28f4f062274ab4774423fa9349e23)
> ("o/confdbstate: fix data loss when writing to a new schema under an existing account").

Writing a value to one confdb schema causes all data stored in a different confdb
schema under the same account to be lost. The bug triggers whenever one schema
has data and the other being written to has no existing data.

## Step 1: Acknowledge the confdb-schema assertions

Fetch the account and account-key assertions from the Store so snapd can verify
the signatures, then acknowledge both schema assertions:

```bash
snap known --remote account account-id=f22PSauKuNkwQTM9Wz67ZCjNACuSjjhN > /tmp/account.assert
snap ack /tmp/account.assert

snap known --remote account-key public-key-sha3-384=-vtoKUiMBQW08EG3HlyFYUxxa7Svz2xhg6baPpjUyDvH0MPeCfogBQoAaneZuPbp > /tmp/account-key.assert
snap ack /tmp/account-key.assert

snap ack foo.assert
snap ack bar.assert
```

## Step 2: Build and install the custodian snaps

Build from inside each snap directory:

```bash
cd snap-foo && snapcraft pack && cd ..
cd snap-bar && snapcraft pack && cd ..
```

Install both with `--dangerous` and connect their plugs:

```bash
snap install snap-foo/repro-foo_1.0_amd64.snap --dangerous
snap connect repro-foo:foo-admin

snap install snap-bar/repro-bar_1.0_amd64.snap --dangerous
snap connect repro-bar:bar-admin
```

Confirm both plugs are connected:

```bash
snap connections repro-foo
snap connections repro-bar
```

Both should show `confdb repro-foo:foo-admin :confdb manual` (and the bar
equivalent).

## Step 3: Reproduce

Write a value into foo and read it back:

```bash
snap set f22PSauKuNkwQTM9Wz67ZCjNACuSjjhN/foo/admin data.value="hello-from-a"
snap get f22PSauKuNkwQTM9Wz67ZCjNACuSjjhN/foo/admin data.value
# hello-from-a
```

Now write a value into bar for the first time:

```bash
snap set f22PSauKuNkwQTM9Wz67ZCjNACuSjjhN/bar/admin data.value="hello-from-b"
```

Read foo again:

```bash
snap get f22PSauKuNkwQTM9Wz67ZCjNACuSjjhN/foo/admin data.value
# error: cannot get "data.value" through .../foo/admin: no data
```

## Resetting

To re-run the repro from a clean state, unset both values:

```bash
snap unset f22PSauKuNkwQTM9Wz67ZCjNACuSjjhN/foo/admin data.value
snap unset f22PSauKuNkwQTM9Wz67ZCjNACuSjjhN/bar/admin data.value
```

Then repeat Step 3.
