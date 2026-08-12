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
the households. Typical actions involve managing households, their metadata, their vaults and
the master passwords.

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
own password, as if on itself it were a kind of user). This password works as an *envelope* for
an internal passphrase that serves to encrypt the actual vault.

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
household add $HOUSEHOLD_NAME
```

This command will be executed, typically, in-pod or in-container (this depends on the deployment
type) and it must take the password via stdin (this implies taking the appropriate provisions in
either Docker, Kubernetes or whatever is used for deployment). After setting the password, it
must take the deployment passphrase. In order to make things secure, this command would actually
take four standard input lines:

```
--- Household Plane credentials ---
Password: {stdin line 1}
Confirm: {stdin line 2}
--- Household Plane vault key ---
Vault key: {stdin line 3}
Confirm: {stdin line 4}
```

For this, the lines 1 and 2 must match, and also the lines 3 and 4 must match.

By this point, the household will be created. The user is totally responsible for backing up the
password and the vault key (both things are needed, but the vault key is the thing to worry more
about, since it holds the only possibility to access the vault data).

If somehow the keys are lost, there must be a mean to recover them. This is done at the user level.

Other operations here involve destroying a household or modifying its metadata in some way. The
rest of the operations are done in-household, and they assume the users have access to the needed
data: vault key, household password, or user passwords. Those commands look like:

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

### Using the household

Using the household is done via HTTP and, ideally, everything behind an HTTPS-enabling proxy.

This is because what happens here at login time / recovery time involves credentials and vault keys.
