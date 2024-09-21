# CRANE KUBERNETES OPERATOR

**Crane Operator** is a Kubernetes operator that simplifies application deployment by enabling you to write one simple `yaml` configuration file and the operator handles everything else from setting up Pod/Deployments, Services, Volumes, Ingress, SSL(with Cert-Manager) and DNS(using External DNS) and more.

It accomplishes this by defining a specification for an `Application` object that you follow while it handles the rest.

**An Example for deploying a Wordpress Instance**

```yaml
apiVersion: cloud.cranom.tech/v1
kind: Application
metadata:
  name: wordpress
spec:
  app-name: wordpress
  image: wordpress:latest
  ports:
  - internal: 80
    external: 30080

  volumes:
  - volume-name: wordpress-persistent-storage

  # Note: Crane Operator does not hande or create Secrets so you need to define your own Secrets
  envFrom: wordpress-secrets


```