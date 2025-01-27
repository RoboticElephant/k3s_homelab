# Rancher
We have already set up our rancher service before we set up traefik. We want to be able to retroactively set up rancher to use traefik's certs. Therefore, we will need to do the following.

## Upgrade
You will need to get your current Rancher version using
```bash
helm ls -n cattle-system
```
This is needed to be able to supply in your `helm upgrade` command.

```bash
helm upgrade rancher rancher-latest/rancher --namespace cattle-system --set hostname=rancher.local.notthedroid.com --set ingress.tls.source=local-notthedroid-com-tls --version v2.10.1
```

### What are we doing?
We are setting the hostname as what how we want to access this. We are setting the `ingress.tls.source` to the secret that we created when setting up Traefik. Our production secret is `local-notthedroid-com-tls`. 

By performing the upgrade we are overwriting the information with the updated information.

## DNS
You will need to make sure that you add the URL to the proper location in your DNS set up.
