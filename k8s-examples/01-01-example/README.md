1. Create [mongodb deployment](mongo-deployment.yaml)

1. Check [mongo dockerhub](https://hub.docker.com/_/mongo) for default port [`27017`](mongo-deployment.yaml#L21) and username [`MONGO_INITDB_ROOT_USERNAME`](mongo-deployment.yaml#L23) and password [`MONGO_INITDB_ROOT_PASSWORD`](mongo-deployment.yaml#L28) env variables

1. To store the values of username, password we need to use secret component which lives in k8s
   1. the deployment file will be inside git repository

1. Create [secret file](mongo-secret.yaml)
   1. base64 value2
   ```sh
   echo -n 'username' | base64
   ```
   1. The order of creation matters, so secret needs to be created first.
1. Apply the secret `kubectl apply -f mongo-secret.yaml`
1. [Refrence the secret in the yaml file](mongo-deployment.yaml#L24-27)

1. Apply the deployment `kubectl apply -f mongo-deployment.yaml`

1. create [internal service](mongo-deployment.yaml#L34-44) for mongodb
   
   To check if the service is connect to the right pod:
      1. `kubectl describe service mongodb-service`
         1. It shows the following output: `Endpoints:         172.17.0.4:27017`
      1. `kubectl get pod -o wide`: Check if `172.17.0.4` is the ip address of the pod

1. Create [mongoexpress](mongoexpress.yaml) deployment
1. Check [mongoexpress dockerhub](https://hub.docker.com/_/mongo-express) for default port, username, password, server
1. For `ME_CONFIG_MONGODB_SERVER` we are going to create a [ConfigMap](mongo-configmap.yaml)
   1. The server name is the name of [mongodb service](mongo-deployment.yaml#L37)
1. Apply configmap first then apply the mongoexpress
1. Access mongoexpress from the browser
   1. Create external service