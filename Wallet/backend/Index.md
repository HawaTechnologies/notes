# Backend

The backend is the most complex structure here. It involves several elements to work with the
management, allowance and signing process. In order to successfully enumerate the needed elements,
the whole life-cycle of this stack (and the different operation planes) must be defined. On top of
this, front-end applications will be defined to connect to this wallet protocol.

Due to the nature of the application, this will involve:

- HTTPS access.
- A regular database (SQL or not).
- Blockchain access (including ERC-4337 bundlers).

By default, this stack will be managed through a `docker compose` stack. Future implementations
might add Kubernetes, but since this one is designed to work in isolated environments, for now
only the Docker Compose stack will be available.

## The interaction planes

We'll be talking about different interaction planes with this backend stack. The first one is
the Master Plane. Next, the household plane. Finally, the user plane. These work in a kind of
hierarchy, so master plane manages households, the household plane manages vault and users, and
the user plane manages permissions and performs the operations we care about.

### The master plane

The topmost plane. This one has the task of defining _households_. In this context, a household
involves the management of a familiar group, and only the master plane has the right to manage
the households. Typical actions involve managing households and their metadata. On creation, the
vault recovery phrase for the household being created is also specified.

With this in mind, households serve as an IAM domain, and they will be stored in an internal
database for the server (while a database brand must be specified, what database deployment is
chosen is up to the deployment - e.g. if Postgres is used, a stack might use a local Postgres
server or a serverless Amazon Aurora database).

Interacting with this plane is done with direct commands only:

- `docker compose exec ... {container} {command}` if Docker Compose is used.
- `kubectl exec ... {pod} -- {command}` if Kubernetes is used.

### The household plane

A household is, as said earlier, an IAM domain. However, it's not just that: it contains all
the accounts, being them EOAs for deployment, EOAs for authority and policy management, or the
final ERC-4337 accounts.

A household is managed by an implicit superuser (this means: each household on itself has its
own password, as if on itself it were a kind of user). This password is used to make an *envelope*
for the vault root key, which is the internal key material used to encrypt the actual vault. This
is done indirectly (the password is used to derive a key which, in turn, is used to decrypt the
wrapped / enveloped vault root key, and this vault root key is used to, indirectly, encrypt /
decrypt the vault so the operations against the vault can be executed - see the section on
_passwords, keys and encryption_ for more details).

The household plane is managed through a web interface we might call /household/. It relates to
a web session whose login starts by writing the password of the household itself. It is also a
single-user environment (only one "superuser" can be defined, and it is the household itself).

The typical features in the household plane involve:

- Managing the household password.
- Managing, importing or exporting the household vault.
- Managing, importing or exporting the household users.

### The user plane

Once users exist inside the household, the user plane lets users configure different aspects
of the household. This implies:

- Actions related to Parent / Alpha accounts. They manage the policies and allowances.
- Actions related to Children / Beta accounts. They are subject to the policies.
  - There's another type of account: Sigma. They're not subject to policies and limitations,
    but still cannot access the parents' features and management entry points. They're
    intended for grown adults, guests or friends. They can, still, contribute to the usage
    statistics on dApps or contracts, if they want.
- Actions related to RPC and signatures.
- Actions related to ERC-4337 account deployments and maintenance.

This is the most important level.

## Life-cycle of a household server

The first thing is to mount the server and ensure the database exists. If the database is
a documental one (like Mongo DB), then nothing is needed other than setting up the database
server. Otherwise, an initial schema / migration is needed if the underlying database is SQL.

If the database server is mounted as part of the stack, the database server will be already
present and, if it's SQL, an internal migration script is needed.

Once the setup is done and the database becomes available, the master plane will be available.
It might even be needed a setup at master plane level (e.g. a command `initialize` to run the
initial migrations of the database, if it's of SQL type). The master plane, however, is only
accessible at local level (direct commands, and never a web API).

### Setting up the household

The first step is to create a household. It may involve an internal command like this:

```shell
# The household name must be a slug, and will be created with a lot of default data.
household add $HOUSEHOLD_NAME
```

This command will be executed, typically, in-pod or in-container (this depends on the deployment
type) and it must take the password via stdin (this implies taking the appropriate provisions in
either Docker, Kubernetes or whatever is used for deployment). After setting the password, it
must take the vault recovery phrase. In order to make things secure, this command would actually
take four standard input lines:

```
--- Household Plane credentials ---
Password: {stdin line 1}
Confirm: {stdin line 2}
--- Household Plane vault recovery phrase ---
Vault recovery phrase: {stdin line 3}
Confirm: {stdin line 4}
```

For this, the lines 1 and 2 must match, and also the lines 3 and 4 must match.

By this point, the household will be created. The user is totally responsible for backing up the
password and the vault recovery phrase. The password is used to access the household and then
execute their commands (e.g. creating users), but also wraps the vault root key. The vault root
key is used under the hood. Typically, neither the household plane nor the user plane will make
use of this key in a direct way, but user requests will involve a decrypt-use-discard cycle of
the encrypted (wrapped) key from their own records. The decrypt-use-discard cycle involves
something like this:

    key = derive_key(password)
    vault_root_key = decrypt(key, vault_key_envelope)
    vault = decrypt(vault_root_key, vault)
    ... work as needed ...
    memory_clear(vault)
    memory_clear(vault_root_key)

> **Please note**: The user owning the household must safely store the vault recovery phrase, for
it cannot be recovered later. Instead, the phrase is used in scenarios involving household backup
/ restore, or recovery of the household password. All of these are done at the household level.

Other operations here involve destroying a household or modifying its metadata in some way (the
rest of the operations are done in-household, and they assume the users have access to the needed
data: vault recovery phrase, household password, or user passwords). Those commands look like:

Getting an existing household:

```shell
household get $HOUSEHOLD_NAME
```

it dumps the details via stdout.

Editing an existing household:

```shell
household update $HOUSEHOLD_NAME
```

taking stdin:

```
# Must provide valid data:
{whole JSON object - fields to be defined later}
```

Deleting an existing household:

```shell
# I have to decide whether deletion is complete or soft/logical.
household delete $HOUSEHOLD_NAME
```

Undeleting a household (only possible if I delete them soft/locally only):

```shell
household undelete $HOUSEHOLD_NAME
```

Listing households in this node:

```shell
household list
```

where we can support multiple output formats and also paging / querying.

Also, resetting the household password from one of the users is something that must
be done only here:

```shell
household password-reset $HOUSEHOLD_NAME $SOME_IN_HOUSEHOLD_USER
```

taking stdin:

```
Usesrname: {stdin line 1}
Password: {stdin line 2}
New Household Password: {stdin line 3}
Confirm: {stdin line 4}
```

Line 1 in-household user to get the internal vault key from.
Line 2 is its password.

In this case, a fake login is attempted and the internal vault key is discovered.

Then, the new / confirm passwords must match, and the new password is set for the
household itself, also enveloping the vault key in the household itself, thus making
effective the password reset.

This command is the only way to restore the password of the superuser, if it is
lost. The superuser is in charge of rotating the vault key if it also forgot it.
That one is done, however, in the user plane (and described in the corresponding
file).

#### A word on passwords, keys and encryption

1. The vault recovery phrase, as described earlier, is a 24-word BIP-39 mnemonic. It
   is validated as such. This means that it is not an arbitrary recovery passphrase.
2. The BIP-39 mnemonic encodes 256 bits of entropy plus checksum data. From it, the
   standard BIP-39 process derives a 512-bit seed. That seed is then used to derive
   a 256-bit vault root key (32 bytes), which is the key material stored inside the
   password-protected envelopes.
3. The passwords, on the contrary, can be arbitrary-length strings. Encryption keys
   are derived from them when they're valid (for the household or for the respective
   user attempting an operation).
4. When a password-based encryption key is derived this way, it is used to create an
   envelope for that vault root key (or to open that envelope, when doing operations
   over the vault). This implies that each user will have, also, its own copy of the
   vault root key, but always enveloped via the derived key from the user's password.
   The same applies for the household, which has the main copy itself.

### Using the household

Using the household is done via HTTP and, ideally, behind an HTTPS-enabling reverse proxy.
This is because what happens here at login time / recovery time involves credentials and
vault-related secrets, and using pure HTTP means exposing those values.

Now, all requests are household-scoped under the /$HOUSEHOLD_NAME/ prefix.

The first thing to do is a login:

```
POST /$HOUSEHOLD_NAME/login
Content-Type: application/json
{
    "username": "{username or '~'}",
    "password": "{password}"
}
```

Using the `~` username implies logging with the household password, not a regular user.
Otherwise, the username must be valid (case-insensitive matching can be considered) for
the specified household.

Appropriate errors must be returned in the case of an error, but a session id must be
returned on success. Something like this:

```
{
    "session_id": "...", # e.g. 64 opaque and high-entropy hexadecimal digits.
}
```

This `session_id` will be used for all the other HTTP endpoints. For example, when the
user wants to insta-revoke its session id, they call this endpoint:

```
POST /$HOUSEHOLD_NAME/logout
Authorization: Bearer {session_id}
```

and, on success, it has its session id revoked. Other operations will also use the same
`Authorization` header in the same way.

Session ids are sensitive. This is an important matter: Browser extensions will keep the
session id in the background context (in a Manifest V2 / V3 Chrome-like extension), and
never in cookies, local storage, or any page-space code-reachable process. Also, mobile
extensions will have the session id in-memory only, and never persisting it to any kind
of storage. A restart on either application or context will cause the user the need to
login again.

Just remember that, while all the endpoints here require the same Authorization header,
certain endpoints are household plane (i.e. only allowed for sessions created for the
user `~`) and certain endpoints are user plane (i.e. only allowed for session created
for other users), throwing respective 403 errors when trying to hit endpoints in wrong
plane.

Concrete actions will be described in a later section or file.

### Session security

Sessions are bearer-token based. A session token is sensitive: anyone who obtains it can act
as that session until it expires or is revoked. Clients must keep the token only in privileged
application storage. Browser extensions should keep it in the background context, never in
cookies, local storage, or page-accessible code. Mobile clients should keep it in memory only.
Restarting the client or extension context requires logging in again.

The server must not store raw session tokens. Instead, it stores a keyed hash / HMAC of each
session token in Redis. This way, a Redis-only leak does not immediately expose live bearer
tokens. The key used for session-token HMACs should be independent from the key used for
active-vault encryption, or derived from the same mounted secret with a different purpose label.

Redis stores two kinds of session-related data:

- `sessions`: maps a hashed session token to session metadata.
- `active_vaults`: maps a household to the household's active vault root key, encrypted with
  the server-side session-cache secret.

The session metadata looks like this:

```
{
    "user": "...",
    "household_id": "...",
    "created_at": "...",
    "last_used_at": "...",
    "expires_at": "..."
}
```

Sessions use a short sliding TTL, for example 15 minutes. Every authenticated request refreshes
the session TTL.

#### The session-cache secret

The session-cache secret is high-entropy server-side key material. It is used only to derive an
encryption key for temporary vault root keys stored in Redis. This prevents a Redis-only leak
from exposing plaintext vault root keys.

In production, and especially in multi-node / replicated deployments, the session-cache secret
should be mounted as a deployment secret with strict filesystem permissions. It should be
readable only by the server process. Every server replica that shares the same Redis session
state must use the same mounted secret.

For a single-node deployment, the session-cache secret may be left empty / unset. In that case,
the server may generate a crypto-secure random secret on startup and keep it internally. This is
acceptable only if the operator accepts that recreating the container / pod can generate a new
secret, making existing `active_vaults` entries undecryptable and invalidating all current
sessions. A mounted secret avoids this session-loss behavior.

This secret does not protect against full application or container compromise. If an attacker can
read both Redis and the session-cache secret, or can execute code inside the server process, they
can probably decrypt active vault material while sessions are active. The design should therefore
also rely on container hardening, least-privilege runtime permissions, protected secret mounts,
disabled core dumps, no swap for secret-bearing memory when possible, tight Redis network access,
authenticated / encrypted Redis connections when crossing a trust boundary, and careful logging
that never records passwords, session tokens, vault root keys, derived keys, or plaintext vault
contents.

#### Runtime and memory handling

The implementation should use a runtime that allows careful handling of secret bytes. Go can be a
reasonable choice, but it does not make memory handling perfect by itself. Secret values should be
kept in mutable byte slices when practical, wiped in place as soon as they are no longer needed,
and avoided in immutable strings, logs, panic messages, request dumps, metrics, and long-lived
objects. This reduces exposure, but it is still defense in depth rather than a complete guarantee:
garbage collection, compiler copies, stack movement, crash dumps, and debugging tools can still
expose process memory if the runtime or host is compromised.

#### Login flow

When a user logs in successfully:

    1. Generate a long, random session token.
    2. Compute a keyed hash / HMAC of that session token.
    3. Store the session record in Redis under that token hash.
    4. Return the raw session token to the client exactly once.
    5. Atomically ensure that the household has an active vault entry.

The active-vault operation must be atomic across all server replicas. This should be implemented
with Redis transactions / Lua scripts or another distributed lock with clear expiry behavior.

If an active vault already exists for the household:

    1. Refresh its TTL.
    2. Associate the new session with that household's active vault.

If no active vault exists for the household:

    1. Use the current user's password-derived key to unwrap the vault root key.
    2. Read or initialize the session-cache secret.
    3. Derive an encryption key from the session-cache secret.
    4. Encrypt the vault root key with authenticated encryption.
    5. Store the encrypted vault root key in `active_vaults` under the household id.
    6. Clear plaintext key material from memory as soon as practical.

#### Active-vault lifetime

The active-vault lifetime must not depend only on a simple reference count updated by login and
logout. That is fragile because sessions can expire naturally, clients can disappear, Redis keys
can be evicted, and requests can fail halfway through a state transition.

Instead, the server should track which valid sessions belong to each active vault. A practical
Redis shape is:

- `sessions:{session_hash}`: the session record, with a short TTL.
- `household_sessions:{household_id}`: a set or sorted set of active session hashes for that
  household.
- `active_vaults:{household_id}`: the encrypted vault root key and metadata.

The `household_sessions` structure should be cleaned opportunistically whenever the household's
active vault is used: remove session hashes whose `sessions:{session_hash}` key no longer exists.
If the cleaned set becomes empty, delete the `active_vaults:{household_id}` entry. This avoids
stale counters and handles both explicit logout and natural session expiration.

The active-vault TTL should also be short and sliding. It should be refreshed only while at least
one valid session still exists for that household. If an active vault expires before a session
does, the next request using that session must revoke the session and require login again, because
the server no longer has the temporary vault root key needed to serve the request. Typically, this
means that the active vaults have the same TTL the sessions have.

#### Logout flow

Logout revokes the current session first. The server then removes the session hash from the
household's active-session set and cleans expired session hashes from that set. If no valid
sessions remain for the household, it deletes the corresponding active vault entry.

This update must be atomic with respect to other login, logout, and authenticated-request flows
for the same household.

#### Authenticated request flow

Every in-session request follows this template:

    1. Return 401 if no bearer token is provided.
    2. Hash / HMAC the bearer token and retrieve the session entry.
    3. Return 401 if the session does not exist.
    4. Refresh the session TTL.
    5. Retrieve the associated active vault for the session's household.
    6. If the active vault is missing, revoke the session and return 401.
    7. Clean expired session hashes from the household's active-session set.
    8. If the current session is no longer valid after cleanup, return 401.
    9. Refresh the active-vault TTL.
    10. Read or initialize the session-cache secret.
    11. Derive the encryption key from the session-cache secret.
    12. Decrypt the vault root key from the active vault entry.
    13. If decryption fails, delete the active vault entry, revoke the session, and return 401.
    14. Use the vault root key only for the duration of the request.
    15. Clear plaintext key material from memory as soon as practical.
    16. Return the result of the underlying operation.

Decryption failure usually means that the active vault was encrypted with a different
session-cache secret, for example after a container / pod recreation when the secret was generated
internally instead of mounted. Treat this as session invalidation, not as a recoverable vault
operation.

Extra steps in this workflow occur only if the operation includes changes in the vault (e.g. when
a new key is added or so). In this case, after the operation executes, the vault must be modified
and the new changes must be encrypted and stored. All these mentions reinforce the idea of needing
household-level atomicity in operations (so a per-household mutex or so is needed on each request
workflow).

## Exposed features

The features in this server are grouped by categories and involve different roles which, in turn,
involve the already-defined users. These users, among the usual things (username, password, ...),
are characterized by their role (only one role per user):

- Parents: They're users which have free participation on any EVM chain, dapp and contract.
- Children: They're users which have constrained participation on any EVM chain. Parents determine
  which policies are children subject to, and also can modify children accounts to become Adults
  (see next entries).
- Adults: They're adult users which have free participation on any EVM chain, dapp and contract.
  However, they don't have the rights to manage policies of children (neither as group nor
  individuals). They are just unrestricted. Parents still manage the user records, so the Adult
  role can be changed or revoked by them, turning them into children.

Parents are only created by the superuser / household access. The household access has ultimate
power on creating accounts and setting their roles. Parents cannot, however, create any parent
account or modify other parent accounts in any way (especially: removing another parent's role
to make it non-parent). Also, while only the user plane is the place where parents set up policies
to other users, parents cannot change policies on other parent users in the household, while they
can change policies to other users (however: policies have no effect on users that are Adults,
at least until the account is switched back to being Child).

Accounts track their statistics on transactions, being simple ones to other user, simple ones to
a contract, or contract method invocation. Each user can see their own statistics, but parents
can also see statistics on any child account in the same household.

Finally, accounts make transactions and even private bunders can be per-network deployed.

All these features will be described in related files.