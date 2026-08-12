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

### About the security of sessions
