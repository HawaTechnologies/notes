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
