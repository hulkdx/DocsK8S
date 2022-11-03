1. Create [mongodb deployment](mongo-deployment.yaml)

1. Check [mongo dockerhub](https://hub.docker.com/_/mongo) for default port [`27017`](mongo-deployment.yaml#L21) and username [`MONGO_INITDB_ROOT_USERNAME`](mongo-deployment.yaml#L23) and password [`MONGO_INITDB_ROOT_PASSWORD`](mongo-deployment.yaml#L25) env variables

1. To store the values of username, password we need to use secret component which lives in k8s
   1. the deployment file will be inside git repository
