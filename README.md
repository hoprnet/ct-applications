# ct-applications

## Postgres

### Create Sealed secret

Download the Postgres client cert, client cert key, and CA key from the Secrets Manager. Update the value of `ENVIRONMENT_NAME` as needed.

```shell
export ENVIRONMENT_NAME=staging #TODO - change to the name of your respective environment

mkdir -p tmp && cd tmp

kubectl create secret generic postgres-certs --namespace ${ENVIRONMENT_NAME} --dry-run=client --type=kubernetes.io/tls --from-file=tls.crt=postgres-${ENVIRONMENT_NAME}-client-cert.pem --from-file=tls.key=postgres-${ENVIRONMENT_NAME}-client-key.pem --from-file=ca.crt=postgres-${ENVIRONMENT_NAME}-ca.pem -o yaml | kubeseal --controller-name=sealed-secrets --controller-namespace=sealed-secrets --format yaml > postgres-certs.yaml

export PGHOST=$(gcloud sql instances describe ${ENVIRONMENT_NAME} --format=json | jq -r '.ipAddresses[0].ipAddress')
export PGPASSWORD=$(gcloud secrets versions access 1 --secret=postgres_${ENVIRONMENT_NAME}_ctdapp)
kubectl create secret generic postgres-credentials --namespace ${ENVIRONMENT_NAME} --dry-run=client --from-literal=PGPORT=5432 --from-literal=PGHOST=${PGHOST}  --from-literal=PGUSER=ctdapp  --from-literal=PGDATABASE=ctdapp --from-literal=PGPASSWORD=${PGPASSWORD} -o yaml | kubeseal --controller-name=sealed-secrets --controller-namespace=sealed-secrets --format yaml > postgres-credentials.yaml
```

## RabbitMQ

Update the value of `ENVIRONMENT_NAME` as needed.

### Create Sealed Secret

```shell
export ENVIRONMENT_NAME=staging #TODO - change to the name of your respective environment
export RABBITMQ_CTAPP_USERNAME=ctapp-${ENVIRONMENT_NAME}
export RABBITMQ_CTAPP_PASSWORD=$(pwgen -s 64 1)
export RABBITMQ_CTAPP_URL="amqp://${RABBITMQ_CTAPP_USERNAME}:${RABBITMQ_CTAPP_PASSWORD}@rabbitmq-ha-cluster.rabbitmq.svc/${RABBITMQ_CTAPP_USERNAME}"
kubectl create secret generic ctapp-rabbitmq-${ENVIRONMENT_NAME} --namespace rabbitmq --dry-run=client --from-literal=CELERY_BROKER_URL=${RABBITMQ_CTAPP_URL} --from-literal=username=${RABBITMQ_CTAPP_USERNAME} --from-literal=password=${RABBITMQ_CTAPP_PASSWORD} -o yaml | kubeseal --controller-name=sealed-secrets --controller-namespace=sealed-secrets --format yaml > /tmp/sealed-secret-rabbitmq.yaml
export RABBITMQ_CTAPP_USERNAME_ENCRYPTED=$(yq ".spec.encryptedData.username" /tmp/sealed-secret-rabbitmq.yaml)
export RABBITMQ_CTAPP_PASSWORD_ENCRYPTED=$(yq ".spec.encryptedData.password" /tmp/sealed-secret-rabbitmq.yaml)
export RABBITMQ_CTAPP_URL_ENCRYPTED=$(yq ".spec.encryptedData.CELERY_BROKER_URL" /tmp/sealed-secret-rabbitmq.yaml)
yq -i e ".spec.encryptedData.username |= \"${RABBITMQ_CTAPP_USERNAME_ENCRYPTED}\"" ${ENVIRONMENT_NAME}/rabbitmq/rabbitmq.yaml
yq -i e ".spec.encryptedData.password |= \"${RABBITMQ_CTAPP_PASSWORD_ENCRYPTED}\"" ${ENVIRONMENT_NAME}/rabbitmq/rabbitmq.yaml
yq -i e ".spec.encryptedData.CELERY_BROKER_URL |= \"${RABBITMQ_CTAPP_URL_ENCRYPTED}\"" ${ENVIRONMENT_NAME}/rabbitmq/rabbitmq.yaml
```

### Test sending messages

```shell
export ENVIRONMENT_NAME=staging #TODO - change to the name of your respective environment
export RABBITMQ_HOST=rabbitmq-ha-cluster.rabbitmq.svc
export RABBITMQ_CTAPP_USERNAME=$(kubectl get secret -n ${ENVIRONMENT_NAME} ctapp-rabbitmq-${ENVIRONMENT_NAME} -o jsonpath="{.data.username}" | base64 --decode)
export RABBITMQ_CTAPP_PASSWORD=$(kubectl get secret -n ${ENVIRONMENT_NAME} ctapp-rabbitmq-${ENVIRONMENT_NAME} -o jsonpath="{.data.password}" | base64 --decode)
export RABBITMQ_CTAPP_URL="amqp://${RABBITMQ_CTAPP_USERNAME}:${RABBITMQ_CTAPP_PASSWORD}@${RABBITMQ_HOST}/${RABBITMQ_CTAPP_USERNAME}"

kubectl run perf-test -n ${ENVIRONMENT_NAME} --image=pivotalrabbitmq/perf-test -- --uri "${RABBITMQ_CTAPP_URL}"
kubectl logs -f -n ${ENVIRONMENT_NAME} perf-test
kubectl delete pod -n ${ENVIRONMENT_NAME} perf-test
```