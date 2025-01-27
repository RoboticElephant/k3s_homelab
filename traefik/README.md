# Traefik
This is a reverse proxy that will be able to... TODO

## Installation
You will need to first make sure that you are able to reach the nodes.
```bash
kubectl get nodes
```
Once, you are able to see the nodes, then verify that you have `Helm` installed.
```bash
helm version
```
If you are able to see the nodes and helm is installed, you will need to make sure that `Metal-LB` is installed. When running the `k3s_ansible` playbook, it is automatically set up. You can review this by going to [k3s_ansible](https://gitea.local.roboticelephant.com/roboticelephant/k3s_ansible).

### Namespace
Now that all this is set up, lets create a `namespace` for traefik
```bash
kubectl create namespace traefik
```
Verify that you are able to see the namespace, by running:
```bash
kubectl get namespaces
```

### Repository
Now we want to add traefik to our repositories.
```bash
helm repo add traefik https://helm.traefik.io/traefik
```
After you add the repo, you will want to update your repositories
```bash
helm repo update
```

### Install traefik
Once we have updated then we will want to make any necessary modifications to the `values.yaml` file.

We will install traefik through `helm`. We will want to run the command
```bash
helm install --namespace=<name-for-namespace-that-was-created-earlier> <release-name> <repo> --values=<values-to-change>
# Example
helm install --namespace=traefik traefik traefik/traefik --values=values.yaml
```

You will then want to verify 
```bash
kubectl get svc --all-namespaces -o wide
```
We are getting all the namespaces in our entire kubernetes cluster.
At this point we will be able to see that traefik is up and running and has service type of `LoadBalancer` as well as the external IP address. 

## Middleware
This will be middleware that is applied to our routes. This is created in `default-headers.yaml`. In order to apply it, we will need to run:
```bash
kubectl apply -f default-headers.yaml
```

You can verify that it has been updated by running
```bash
kubectl get middleware
# if not in default namespace, then will need to include the namespace
kubectl get --namespace=traefik middleware
```

## Dashboard
We will now need to install dashboard, but we need basic auth credentials. Therefore, first we will need to install
```bash
sudo apt install apache2-utils
```
This will used so we can use `htpasswd` to create the password.
```bash
htpasswd -nb <username> <password> | openssl base64
# Example
htpasswd -nb elephant password | openssl base64
```
Once you have this, you will want to add it to the file `dashboard/secret-dashboard.yaml`.

Then you will want to apply the file
```bash
kubectl apply -f dashboard/secret-dashboard.yaml
```
To check this you can run
```bash
kubectl get secrets --namespace traefik
```

### Ingress
The dashboard is now enabled in traefik, but we need to create the ingress so we can actually get to it. We now need to route traefik into it. 

We will need to create new middleware that will be used to have the secret for ingress using basic auth
```bash
kubectl apply -f dashboard/middleware.yaml
```
You can verify that it exists running the commands above.

Once you have applied the middleware, we can now apply the ingress
```bash
kubectl apply -f dashboard/ingress.yaml
```

## Cert Manager
The way the docker version does it, is that it references a local certs. This is fine, but doesn't play nicely with highly available (HA) system. Only one pod would be able to interact at any time. Therefore, we will be using `Cert Manager` instead.

Cert Manager can be our issuer and also fetch and retrieve certificates. This will allow the certs to be stored as secrets and we can have multiple pods access the secrets. This allows us to pull certs from cert manager and scale traefik as wide as we want. 

### Namespace
You will want to create a namespace if it doesn't already exist.
```bash
kubectl create namespace cert-manager
```

### Cert CRDs
We will then want apply some CRDs (Custom Resource Definitions). The CRDs allow you to define objects in kubernetes that kubernetes doesn't know about natively. 
```bash
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.16.3/cert-manager.crds.yaml
```

### Install Cert Manager
You will need to install it as follows:
```bash
helm install cert-manager jetstack/cert-manager --namespace cert-manager --values=values.yaml --version v1.16.3
```
If this already exists, then you will need to `upgrade` instead of `install`.

We can then run
```bash
kubectl get pods --namespace cert-manager
```
to see all the pods that are running. 

### Issuer
Now that that is up and running, we want to get certs from LetsEncrypt. You will need to set up your API token in your desired `DNS01` validator. Since I am using Cloudflare, I am referencing [cert-manager docs](https://cert-manager.io/docs/configuration/acme/dns01/cloudflare/#api-tokens) for creating the API Tokens.

#### Cloudflare
Inside of Cloudflare, to create a token go to `User Profile > API Tokens > API Tokens`. Cert-manager recommends the settings:

- Permissions:
    - Zone -> DNS -> Edit
    - Zone -> Zone -> Read
- Zone Resources:
    - Include -> All Zones

##### Cloudflare Secret Token
Once we have created the token and saved it, we will want to apply the token
```bash
kubectl apply -f secret-cf-token.yaml
```

#### Staging
This is where we can get certificates, but they aren't globally trusted. The purpose is to allow you to test the API. It is preferable to get this working first in staging, before going to production. 

Once you have created the cloudflare token and applied it (see `Cloudflare Secret Token` above), then we can apply the stage lets encrypt.
```bash
kubectl apply -f cert-manager/issues/letsnecrypt-staging.yaml
```

Now we will want to apply the certificate yaml
```bash
kubectl apply -f cert-manager/certifications/staging/local-notthedroid-com.yaml
```

You can make sure that it is running as expected by running
```bash
kubectl logs -n cert-manager -f <pod>
# The <pod> will be cert-manager-<alphanumeric>
```
The `-f` command means to follow the logs from the pod. You will need to keep watching this until it no longer throws an error. You could also run
```bash
kubectl get challenges
```
If it tells you there are `No resources found in default namespace` it more than likely means that you have retrieved a certificate. 

#### Production
This is globally trusted endpoint, but has rate limiting applied to it. If you exceed the failure rate that LetsEncrypt has set, then you will lock yourself out for up to a week. 

Once you have created the cloudflare token and applied it (see `Cloudflare Secret Token` above), then we can apply the production lets encrypt.
```bash
kubectl apply -f cert-manager/issues/letsnecrypt-production.yaml
```

Now we will want to apply the certificate yaml
```bash
kubectl apply -f cert-manager/certifications/production/local-notthedroid-com.yaml
```

You can make sure that it is running as expected by running
```bash
kubectl logs -n cert-manager -f <pod>
# The <pod> will be cert-manager-<alphanumeric>
```
The `-f` command means to follow the logs from the pod. You will need to keep watching this until it no longer throws an error. You could also run
```bash
kubectl get challenges
```
If it tells you there are `No resources found in default namespace` it more than likely means that you have retrieved a certificate. 

Now you can verify by running
```bash
kubectl get certificate
```
This should show you that the name should be `True` for Ready. This is a good sign that everything is working as expected. 

## Updating Service
Now that this is working, if you have any services running, you will want to update them to reference the proper secret in the `ingress.yaml` file. 
Specifically make sure the below is added for the secret, which is the certificate.
```yaml
spec:
  ...
  tls:
    secretName: local-notthedroid-com-tls
```

## Verify
You can verify by running
```bash
kubectl describe ingressroutes.traefik.io
```
Then verify that the Secret Name is the one that you expect as per the `Updating Service` above.
