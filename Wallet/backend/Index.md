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
the Master Plane. Next, the household plane. Finally, the user plane.

### The master plane

The topmost plane. This one has the task of defining _households_. In this context, a household
involves the management of a familiar group, and only the master plane has the right to manage
the households. Typical actions involve:

- Creating a household, with an admin password. This will create a superuser with name `admin`
  and the chosen password.
- Deleting a household.
- Changing a user's password (be it superuser or not) inside a household.
- Modifying a household's data.

With this in mind, households serve as an IAM domain, and they will be stored in an internal
database for the server (while a database brand must be specified, what database deployment is
chosen is up to the deployment - e.g. if Postgres is used, a stack might use a local Postgres
server or a serverless Amazon Aurora database).

Households can only be managed this way. A superuser cannot manage a household by itself, even
being the top-level user inside of it and actually being the one able to manage all the users
inside the household.

Interacting with this plane is done with direct commands only:

- `docker compose exec ... {container} {command}` if Docker Compose is used.
- `kubectl exec ... {pod} -- {command}` if Kubernetes is used.

## The household plane
