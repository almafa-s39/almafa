# Postfix + Dovecot: Aliases, LDAP Mail Routing, and Sieve Vacation (Debian 13.3)

This document covers two authentication scenarios for a Postfix +
Dovecot mail setup on Debian 13.3: local (PAM) users and virtual (LDAP)
users. It includes alias/mailing-list expansion and a Sieve-based
vacation auto-responder for each scenario.

Debian 13 (Trixie) ships **Dovecot 2.4**, which introduced a
non-backward-compatible configuration format compared to Dovecot 2.3.
Two configuration issues in the draft this document was built from would
have prevented the service from starting or working correctly on
Dovecot 2.4 — both are called out and corrected below.

> [!WARNING]
> Two changes were required versus the original draft:
> 1. The LDAP `passdb`/`userdb` blocks used the Dovecot 2.3-style
>    `args = /etc/dovecot/dovecot-ldap.conf.ext` pattern. **`args` is not
>    a valid Dovecot 2.4 setting** — external `.conf.ext` files were
>    replaced by inline settings. As written, this fails to parse and
>    the `dovecot` service will not start.
> 2. The Sieve-enable line `mail_plugins = $mail_plugins sieve` is
>    Dovecot 2.3 syntax. In 2.4 it must use the new boolean-list block
>    form, `mail_plugins { sieve = yes }`, or the plugin is silently not
>    loaded and vacation rules are ignored.
>
> Both are corrected in Sections 2.2 and 3.1/3.3 below.

## 1. Scenario 1: PAM Authentication (Local Users)

Local users map directly to system accounts via PAM. This is Debian's
default Dovecot authentication method, so no `passdb`/`userdb` changes
are required for this scenario.

### 1.1 Aliases and Basic Mailing Lists

For local users, alias and mailing-list expansion is handled entirely
by `/etc/aliases`, which Postfix reads by default. This part of the
original draft is correct as written and needed no changes.

```bash
# /etc/aliases
# Basic Alias
john.doe: jdoe

# Basic Mailing List (Alias Expansion)
sysadmins: jdoe, root, alice
network-team: :include:/etc/postfix/network-team-list
```

**Command Breakdown & Explanation:**

- `john.doe: jdoe`: A simple one-to-one alias, delivering mail
  addressed to `john.doe` to the local account `jdoe`.
- `sysadmins: jdoe, root, alice`: A mailing list expanded inline —
  mail to `sysadmins` is delivered to all three listed recipients.
- `network-team: :include:/etc/postfix/network-team-list`: Expands to
  the list of usernames contained in the referenced file, one per line,
  rather than being hard-coded in `/etc/aliases` itself.

```bash
newaliases
```

> [!IMPORTANT]
> Always run `newaliases` after editing `/etc/aliases`. Postfix reads
> the compiled `.db` version of this file, not the plaintext file
> directly — edits will not take effect until it is rebuilt.

### 1.2 Vacation Auto-Responder (Dovecot Sieve)

For Postfix to hand a message to Dovecot so the Sieve `vacation` rule
can fire, local delivery must go through Dovecot's LMTP service rather
than Postfix's built-in local delivery agent.

Postfix Configuration (`/etc/postfix/main.cf`):

```ini
mailbox_transport = lmtp:unix:private/dovecot-lmtp
```

This line is correct as originally written and needs no change — it
routes all local delivery through Dovecot's LMTP socket.

Dovecot Configuration (`/etc/dovecot/conf.d/20-lmtp.conf`):

```ini
protocol lmtp {
  mail_plugins {
    sieve = yes
  }
}
```

**Command Breakdown & Explanation:**

- `protocol lmtp { ... }`: Scopes the enclosed settings to Dovecot's
  LMTP service specifically.
- `mail_plugins { sieve = yes }`: Enables the Sieve plugin for LMTP
  delivery, using Dovecot 2.4's boolean-list setting syntax. The
  originally drafted `mail_plugins = $mail_plugins sieve` line is 2.3
  syntax; `$setting` variable references now require an explicit
  `$SET:` prefix and are no longer needed for this particular idiom,
  since the new boolean-list type replaces it directly.

```bash
~/.dovecot.sieve
```

Local users can still place a `.dovecot.sieve` file directly in their
home directory with no further Dovecot configuration required: if no
Sieve script storage is explicitly configured, Dovecot 2.4
auto-detects a `~/.dovecot.sieve` file (or a `~/sieve/` directory) for
the `personal` script storage. This part of the original draft's intent
holds up under 2.4 once the plugin is actually loaded correctly above.

```sieve
# ~/.dovecot.sieve
require ["vacation"];
vacation
  :days 7
  :subject "Out of Office"
  "I am currently away from the keyboard. For urgent infrastructure issues, contact the on-call admin.";
```

> [!TIP]
> The Sieve script syntax itself is unaffected by the Dovecot 2.3 → 2.4
> config format change — only the surrounding Dovecot *configuration*
> that enables Sieve changed. The script above was correct as originally
> written.

## 2. Scenario 2: LDAP Authentication (Virtual Users)

### 2.1 Dovecot 2.4 LDAP Authentication

The `dovecot_config_version` / `dovecot_storage_version` requirement in
the original draft is correct — Dovecot 2.4 refuses to start without
these as the first settings in `dovecot.conf`. The `passdb`/`userdb`
blocks that followed, however, used the removed `args =` file-include
pattern and needed to be rewritten using inline 2.4 settings.

> [!IMPORTANT]
> Install the `dovecot-ldap` package in addition to `dovecot-core` —
> the LDAP authentication driver ships as a separate package on Debian
> and is not included by default:
> `apt install dovecot-ldap`

Dovecot Configuration (`/etc/dovecot/dovecot.conf`):

```ini
# Mandatory in Debian 13 (Dovecot 2.4+)
dovecot_config_version = 2.4.0
dovecot_storage_version = 2.4.0

# LDAP connection settings (global, shared by passdb and userdb)
ldap_uris = ldap://ldap.yourdomain.local
ldap_auth_dn = cn=dovecot-bind,ou=System,dc=yourdomain,dc=local
ldap_auth_dn_password = CHANGE-ME-BIND-PASSWORD
ldap_base = ou=People,dc=yourdomain,dc=local

# Password lookup
passdb ldap {
  filter = (&(objectClass=posixAccount)(uid=%{user}))
  fields {
    user = %{ldap:uid}
    password = %{ldap:userPassword}
  }
}

# User lookup (home directory, uid/gid)
userdb ldap {
  filter = (&(objectClass=posixAccount)(uid=%{user}))
  fields {
    home = %{ldap:homeDirectory}
    uid = %{ldap:uidNumber}
    gid = %{ldap:gidNumber}
  }
}
```

**Command Breakdown & Explanation:**

- `ldap_uris`, `ldap_auth_dn`, `ldap_auth_dn_password`, `ldap_base`:
  Global LDAP connection settings, shared by both the `passdb` and
  `userdb` blocks below them. This replaces the separate
  `dovecot-ldap.conf.ext` file entirely — there is no external file to
  maintain in Dovecot 2.4.
- `passdb ldap { filter = ...; fields { ... } }`: Performs a password
  lookup directly against LDAP. `filter` locates the user's LDAP entry;
  `fields` maps LDAP attributes (via the `%{ldap:attrName}` syntax) to
  the internal Dovecot fields `user` and `password`.
- `userdb ldap { filter = ...; fields { ... } }`: Performs the
  companion user-info lookup, mapping `homeDirectory`, `uidNumber`, and
  `gidNumber` from LDAP to Dovecot's `home`, `uid`, and `gid` fields.

> [!CAUTION]
> `passdb_ldap_bind = yes` (authentication binds) is generally preferred
> over the password-lookup approach shown above for security — it never
> exposes password hashes to the Dovecot process, since LDAP itself
> verifies the credential. The example above uses password lookups
> because it's a closer match to the original draft's intent; switch to
> `passdb_ldap_bind = yes` with a `passdb_ldap_filter` if your security
> policy requires it.

### 2.2 Aliases and Mailing Lists via LDAP

Postfix queries LDAP directly to expand aliases and mailing lists for
virtual users, using its own independent `ldap:` table driver — this is
unrelated to Dovecot's LDAP configuration and did not change with the
Dovecot 2.4 upgrade.

Postfix Configuration (`/etc/postfix/main.cf`):

```ini
virtual_alias_maps = ldap:/etc/postfix/ldap-aliases.cf, ldap:/etc/postfix/ldap-groups.cf
```

Postfix LDAP Alias Config (`/etc/postfix/ldap-aliases.cf`):

```ini
server_host = ldap://ldap.yourdomain.local
search_base = ou=People,dc=yourdomain,dc=local
query_filter = (mailAlias=%s)
result_attribute = mail
```

This single-recipient alias lookup is correct as originally written —
a single `result_attribute` returning one address per match needs no
changes.

> [!WARNING]
> The originally drafted mailing-list config used
> `result_attribute = memberUid` against a `posixGroup`-style entry.
> `memberUid` holds bare usernames (e.g. `jdoe`), **not** email
> addresses, so Postfix would try to deliver to a literal recipient
> address like `jdoe` rather than `jdoe@yourdomain.local` — mail
> to the list would bounce or misroute. The corrected version below uses
> a `groupOfNames`-style entry instead, whose `member` attribute holds
> full DNs that Postfix can recurse into to fetch each member's real
> `mail` attribute.

Postfix LDAP Mailing List Config (`/etc/postfix/ldap-groups.cf`) — corrected for `groupOfNames`:

```ini
server_host = ldap://ldap.yourdomain.local
search_base = ou=Groups,dc=yourdomain,dc=local
query_filter = (mail=%s)
result_attribute =
special_result_attribute = member
leaf_result_attribute = mail
```

**Command Breakdown & Explanation:**

- `query_filter = (mail=%s)`: Finds the group entry whose `mail`
  attribute matches the address being expanded (e.g. `network-team@yourdomain.local`).
- `result_attribute = ` (empty): Disables direct attribute expansion at
  the group level, since group membership here is expressed through DN
  references rather than inline address strings.
- `special_result_attribute = member`: Tells Postfix that the `member`
  attribute contains DNs to recurse into — this is how `groupOfNames`
  membership is expanded (Postfix looks up each referenced DN in turn).
- `leaf_result_attribute = mail`: Once recursion reaches each member's
  own LDAP entry (a "leaf", not another group), return that entry's
  `mail` attribute as the final expanded address.

> [!NOTE]
> This corrected version assumes your directory's mail groups use the
> `groupOfNames` schema (`member` attribute holding full member DNs). If
> your directory can only use `posixGroup`/`memberUid`, expanding
> straight to valid email addresses needs an extra layer — either a
> secondary LDAP table that maps each `uid` to its `mail` attribute
> before mail delivery, or migrating the mail-enabled groups to
> `groupOfNames`. Confirm which schema your directory actually uses
> before deploying this section.

### 2.3 Vacation Auto-Responder for Virtual Users

The LMTP plugin-enable configuration is identical to Scenario 1
(Section 1.2) — use the same corrected `protocol lmtp { mail_plugins {
sieve = yes } }` block in `/etc/dovecot/conf.d/20-lmtp.conf`.

Because LDAP-backed virtual users don't have standard `/home/user`
directories, their Sieve scripts live under their virtual mail root
instead (the `home` value returned by the `userdb ldap` lookup in
Section 2.1, e.g. `/var/vmail/domain.com/user/`).

> [!WARNING]
> The originally drafted `90-sieve.conf` used a `plugin { sieve = ...;
> sieve_dir = ... }` block. **The `plugin { }` section no longer exists
> in Dovecot 2.4** — all former plugin settings are now ordinary global
> settings, configured through the named `sieve_script` filter shown
> below instead.

Dovecot Sieve Config (`/etc/dovecot/conf.d/90-sieve.conf`) — corrected:

```ini
sieve_script personal {
  driver = file
  path = ~/sieve
  active_path = ~/.dovecot.sieve
}
```

**Command Breakdown & Explanation:**

- `sieve_script personal { ... }`: Declares a named Sieve script
  storage of type `personal` (the default type for this filter name),
  which the LMTP service consults to find the user's active vacation
  script.
- `driver = file`: Uses the local filesystem storage driver — the
  default, but shown explicitly for clarity.
- `path = ~/sieve`: The `~` resolves against the user's `home` value,
  which for virtual users comes from the `userdb ldap` lookup in
  Section 2.1 (e.g. `/var/vmail/domain.com/user/sieve`), not a real
  `/home` directory.
- `active_path = ~/.dovecot.sieve`: The symlink Dovecot follows to
  determine which script in `path` is currently active.

## 3. Verification and Troubleshooting

### 3.1 Verify Dovecot configuration syntax

```bash
doveconf -n
```

What it checks and variables to look for:

- **Exit status / output**: Should print the effective configuration
  without any `Error:` or `Fatal:` lines. A parse error here (e.g. an
  unrecognized `args` setting, or a `plugin { }` block) will name the
  offending file and line — this is the fastest way to catch the two
  issues corrected in this document if they were reintroduced.

### 3.2 Verify Postfix configuration syntax

```bash
postfix check
```

What it checks and variables to look for:

- **Output**: Should return no output at all on success. Any line
  starting with `postfix:` indicates a configuration problem, commonly
  a missing file referenced by `virtual_alias_maps` or a permissions
  issue on `/etc/postfix/ldap-*.cf`.

### 3.3 Verify Dovecot LDAP passdb/userdb lookups

```bash
doveadm auth test <username>
doveadm user <username>
```

What it checks and variables to look for:

- **`doveadm auth test`**: Prompts for a password and reports
  `passdb: <username> auth succeeded` on success, confirming the
  `passdb ldap` filter and `fields` mapping in Section 2.1 are correct.
- **`doveadm user`**: Prints the resolved `home`, `uid`, and `gid`
  values from the `userdb ldap` lookup — confirm these match the
  expected values for a known test account.

### 3.4 Verify LMTP delivery and Sieve execution

```bash
postmap -q jdoe ldap:/etc/postfix/ldap-aliases.cf
echo "Test vacation" | sendmail jdoe
journalctl -u dovecot -f
```

What it checks and variables to look for:

- **`postmap -q`**: Should return the expected resolved address for a
  known alias, confirming the Postfix `ldap-aliases.cf` query is
  reaching the directory correctly.
- **journalctl output after `sendmail`**: Look for a line confirming
  the Sieve plugin loaded (`sieve: ... loaded`) and, if a vacation
  script is active, a `sieve: msgid=...: stored mail into mailbox
  'INBOX'` or vacation-specific log line. No mention of `sieve` in the
  log at all means the plugin is still not being loaded — recheck
  Section 1.2 / 2.3's `mail_plugins { sieve = yes }` block.

### 3.5 Verify mailing-list group expansion (LDAP)

```bash
postmap -q network-team@yourdomain.local ldap:/etc/postfix/ldap-groups.cf
```

What it checks and variables to look for:

- **Output**: Must be a comma-separated list of valid, fully-qualified
  email addresses for each group member — not bare usernames. If the
  output contains values that look like `uid` strings rather than
  addresses, the directory is using `posixGroup`/`memberUid` and needs
  the schema change or secondary lookup noted in Section 2.2.

> [!NOTE]
> LDAP schema details (whether your directory uses `posixGroup` or
> `groupOfNames` for mail groups, and exact attribute names like
> `mailAlias`) are site-specific. Confirm your actual schema with your
> directory administrator before deploying Section 2.2 — the corrected
> config assumes `groupOfNames`, which is an assumption made to produce
> a working example, not a fact about your specific directory.
```