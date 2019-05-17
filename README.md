To install cockroachdb run:

```
export APP_NAME=cockroachdb
export NAMESPACE=estafette
export TEAM_NAME=estafette
export HOSTNAMES=***
export COCKROACHDB_VERSION=v19.1.0
export REPLICAS=5
export CPU_REQUEST=3000m
export CPU_LIMIT=8000m
export MEMORY_REQUEST=16384Mi
export MEMORY_LIMIT=16384Mi
export STORAGE_CLASS=fast
export PVC_SIZE=10Gi
export LOCALITY=continent=europe,country=belgium,region=europe-west1,zone=europe-west1-c

cat kubernetes.yaml | envsubst \$APP_NAME,\$NAMESPACE,\$TEAM_NAME,\$HOSTNAMES,\$COCKROACHDB_VERSION,\$REPLICAS,\$CPU_REQUEST,\$CPU_LIMIT,\$MEMORY_REQUEST,\$MEMORY_LIMIT,\$STORAGE_CLASS,\$PVC_SIZE,\$LOCALITY | kubectl apply -f -

```

To start a client and connect to the database run

```
kubectl run cockroachdb-client -n estafette -ti --generator=run-pod/v1 --rm=true --image=cockroachdb/cockroach:v2.1.4 --command -- ./cockroach sql --insecure --host=cockroachdb-public --database=estafette_ci_api
```