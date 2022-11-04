1. `minikube addons enable ingress`
2. Above code does that
3. Run [mongoexpress-change-to-internal-service](mongoexpress-change-to-internal-service.yaml) to convert the service into internal service.
4. Create and apply [ingress rule](mongo-ingress.yaml)
5. `kubectl get ingress` should see it, copy the address
6. Add the ip address to /etc/hosts file