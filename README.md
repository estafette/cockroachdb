# Cockroachdb

This repository is no longer in use. It is now installed from the https://github.com/estafette/estafette-ci Helm chart instead.

## Creating the cockroachdb cluster

To install cockroachdb run:

```bash
export APP_NAME=cockroachdb
export NAMESPACE=estafette
export TEAM_NAME=estafette
export HOSTNAMES=***
export COCKROACHDB_VERSION=v20.1.2
export REPLICAS=3
export CPU_REQUEST=3000m
export CPU_LIMIT=8000m
export MEMORY_REQUEST=16384Mi
export MEMORY_LIMIT=16384Mi
export STORAGE_CLASS=fast
export PVC_SIZE=50Gi
export LOCALITY=continent=europe,country=belgium,region=europe-west1,zone=europe-west1-c

cat kubernetes.yaml | envsubst \$APP_NAME,\$NAMESPACE,\$TEAM_NAME,\$HOSTNAMES,\
\$COCKROACHDB_VERSION,\$REPLICAS,\$CPU_REQUEST,\$CPU_LIMIT,\$MEMORY_REQUEST,\
\$MEMORY_LIMIT,\$STORAGE_CLASS,\$PVC_SIZE,\$LOCALITY | kubectl apply -f -
```

For each pod of the statefulset starting up it will create a _certificate signing request_ (CSR) in Kubernetes, which you'll have to approve in order for the _statefulset_ to complete startup:

Check if the first CSR is there with:

```bash
kubectl get csr
```

And then approve it with

```bash
kubectl certificate approve <namespace>.node.cockroachdb-0
```

This will let the first pod continue startup and create a Kubernetes secret with the certificate for that pod. Then the next pod will create a csr, you'll have to approve and then the next.

```bash
kubectl certificate approve <namespace>.node.cockroachdb-1
kubectl certificate approve <namespace>.node.cockroachdb-2
```

Now that the cluster is up and running we want to create a user for ourselve to use the admin interface and users for our client applications.

## Connect a client pod to the cluster

In order to create users and grant roles you need to run a client pod with the _root_ user. Do so by running:

```bash
kubectl delete po cockroachdb-client-secure -n <namespace> --ignore-not-found
cat secure-client.yaml | \
APP_NAME=cockroachdb NAMESPACE=<namespace> USER=root envsubst \$APP_NAME,\$NAMESPACE,\$USER | \
kubectl apply -f - -n <namespace>
```

The first time you do this for the _root_ user you'll have to approve the _certificate signing request_ (CSR).

Check if the CSR is there with:

```bash
kubectl get csr
```

And then approve it with

```bash
kubectl certificate approve <namespace>.client.root
```

You can then _exec_ into the pod with:

```bash
kubectl exec cockroachdb-client-secure -n <namespace> -ti -- /cockroach/cockroach sql \
--certs-dir=/cockroach-certs/ --host cockroachdb-public --database <database>
```

### Create admin users

Now that you're on the _sql_ command line create a user with:

```sql
CREATE USER <username> WITH PASSWORD '<password>';
```

For this user to be able to use the admin interface the _admin_ role needs to be granted:

```sql
GRANT admin TO <username>;
```

Now you can use the username and password to log in to the admin gui.

### Create application user

For applications to connect a password is not needed, it uses a client side certificate to authenticate instead. To create a user without password run:

```sql
CREATE USER <username>;
```

Depending on what this user is allowed to do against various databases and tables it's best to create new roles with just the permissions you want it to have and then grant the role to the user:

```sql
GRANT <role> TO <username>;
```

For example a schema migration tool with user `migrator` needs to have a lot of permissions, so for this create the `db_admin_role`:

```sql
CREATE ROLE db_admin_role;
GRANT ALL ON DATABASE <database> TO db_admin_role;
GRANT ALL ON TABLE <database>.* TO db_admin_role;

CREATE USER migrator;
GRANT db_admin_role TO migrator;
```

For a client application that isn't allowed to change the schema itself, but is allowed to do CRUD queries on the database tables:

```sql
CREATE ROLE db_user_role;
GRANT SELECT,INSERT,DELETE,UPDATE ON DATABASE <database> TO db_user_role;
GRANT SELECT,INSERT,DELETE,UPDATE ON TABLE <database>.* TO db_user_role;

CREATE USER myapp;
GRANT db_user_role TO myapp;
```

Next we create a client-side certificate for the application to connect securely.

## Create application user certificate

For the application user to securely connect to the cluster you'll have to create a client-side certificate. You can do so by first running:

```bash
kubectl delete job cockroachdb-client-cert-init-for-user -n <namespace> --ignore-not-found
cat init-client-secret.yaml | \
APP_NAME=cockroachdb NAMESPACE=<namespace> USER=<user> envsubst \$APP_NAME,\$NAMESPACE,\$USER | \
kubectl apply -f - -n <namespace>
```

Once this job has started check the CSR and approve it:

```bash
kubectl get csr
```

Approve the csr with

```bash
kubectl certificate approve <namespace>.client.<user>
```

Now that you have the secret you can mount this into your client application and connect to the database with the following connection string:

```
postgresql://<user>@cockroachdb-public:26257/<database>?sslmode=verify-full&sslrootcert=/var/run/secrets/kubernetes.io/serviceaccount/ca.crt&sslcert=<client certificate mount folder>/cert&sslkey=<client certificate mount folder>/key
```