1. `minikube addons enable ingress`
1. Run [mongoexpress-change-to-internal-service](mongoexpress-change-to-internal-service.yaml) to convert the service into internal service.
1. Create and apply [ingress rule](mongo-ingress.yaml)
1. `kubectl get ingress` should see it, copy the address
1. Add the ip address to /etc/hosts file
