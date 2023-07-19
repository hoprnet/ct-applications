### Create CTApp User

````
export ENVIRONMENT_NAME=staging
export RABBITMQ_CTAPP_USERNAME=ctapp-${ENVIRONMENT_NAME}
export RABBITMQ_CTAPP_PASSWORD=$(pwgen -s 64 1)
export RABBITMQ_CTAPP_URL="amqp://${RABBITMQ_CTAPP_USERNAME}:${RABBITMQ_CTAPP_PASSWORD}@rabbitmq-ha-cluster.rabbitmq.svc/${RABBITMQ_CTAPP_USERNAME}"
kubectl create secret generic ctapp-rabbitmq-${ENVIRONMENT_NAME} --namespace ${ENVIRONMENT_NAME} --dry-run=client --from-literal=CELERY_BROKER_URL=${RABBITMQ_CTAPP_URL} --from-literal=username=${RABBITMQ_CTAPP_USERNAME} --from-literal=password=${RABBITMQ_CTAPP_PASSWORD} -o yaml | kubeseal --controller-name=sealed-secrets --controller-namespace=sealed-secrets --format yaml > /tmp/sealed-secret-rabbitmq.yaml
export RABBITMQ_CTAPP_USERNAME_ENCRYPTED=$(yq ".spec.encryptedData.username" /tmp/sealed-secret-rabbitmq.yaml)
export RABBITMQ_CTAPP_PASSWORD_ENCRYPTED=$(yq ".spec.encryptedData.password" /tmp/sealed-secret-rabbitmq.yaml)
export RABBITMQ_CTAPP_URL_ENCRYPTED=$(yq ".spec.encryptedData.CELERY_BROKER_URL" /tmp/sealed-secret-rabbitmq.yaml)
yq -i e ".spec.encryptedData.username |= \"${RABBITMQ_CTAPP_USERNAME_ENCRYPTED}\"" ${ENVIRONMENT_NAME}/rabbitmq/sealed-secret-ctapp.yaml
yq -i e ".spec.encryptedData.password |= \"${RABBITMQ_CTAPP_PASSWORD_ENCRYPTED}\"" ${ENVIRONMENT_NAME}/rabbitmq/sealed-secret-ctapp.yaml
yq -i e ".spec.encryptedData.CELERY_BROKER_URL |= \"${RABBITMQ_CTAPP_URL_ENCRYPTED}\"" ${ENVIRONMENT_NAME}/rabbitmq/sealed-secret-ctapp.yaml
````



Test sending messages

````
export ENVIRONMENT_NAME=staging
export RABBITMQ_HOST=rabbitmq-ha-cluster.rabbitmq.svc
export RABBITMQ_CTAPP_USERNAME=$(kubectl get secret -n ${ENVIRONMENT_NAME} ctapp-rabbitmq-${ENVIRONMENT_NAME} -o jsonpath="{.data.username}" | base64 --decode)
export RABBITMQ_CTAPP_PASSWORD=$(kubectl get secret -n ${ENVIRONMENT_NAME} ctapp-rabbitmq-${ENVIRONMENT_NAME} -o jsonpath="{.data.password}" | base64 --decode)
export RABBITMQ_CTAPP_URL="amqp://${RABBITMQ_CTAPP_USERNAME}:${RABBITMQ_CTAPP_PASSWORD}@${RABBITMQ_HOST}/${RABBITMQ_CTAPP_USERNAME}"

kubectl run perf-test -n ${ENVIRONMENT_NAME} --image=pivotalrabbitmq/perf-test -- --uri "${RABBITMQ_CTAPP_URL}"
kubectl logs -f -n ${ENVIRONMENT_NAME} perf-test
kubectl delete pod -n ${ENVIRONMENT_NAME} perf-test
````