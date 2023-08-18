### Create Sealed secret

````
export ENVIRONMENT_NAME=staging



kubectl create secret generic postgres-certs --namespace ${ENVIRONMENT_NAME} --dry-run=client --type=kubernetes.io/tls --from-file=tls.crt=postgres-staging-client-cert.pem --from-file=tls.key=postgres-staging-client-key.pem --from-file=ca.crt=postgres-staging-ca.pem -o yaml | kubeseal --controller-name=sealed-secrets --controller-namespace=sealed-secrets --format yaml > sealed-secret-postgres-certs.yaml

export PGHOST=$(gcloud sql instances describe staging --format=json | jq -r '.ipAddresses[0].ipAddress')
export PGPASSWORD=$(gcloud secrets versions access 1 --secret=postgres_staging_ctdapp)
kubectl create secret generic postgres-credentials --namespace ${ENVIRONMENT_NAME} --dry-run=client --from-literal=PGPORT=5432 --from-literal=PGHOST=${PGHOST}  --from-literal=PGUSER=ctdapp  --from-literal=PGDATABASE=ctdapp --from-literal=PGPASSWORD=${PGPASSWORD} -o yaml | kubeseal --controller-name=sealed-secrets --controller-namespace=sealed-secrets --format yaml > sealed-secret-postgres-credentials.yaml

````
