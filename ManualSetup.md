# ArgoCD insecure mode
1. Edit the argocd deployment:
```kubectl edit deployment argocd-server -n argocd```

1. Add the --insecure switch to the args:
    - args:
        - \<server executable>
    	- --insecure

1. Restart the argocd deployment:
```kubectl rollout restart deployment argocd-server -n argocd```

# [Optional] Create self-signed certificate for Traefik

Create the cert:
```
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout homelab.key -out homelab.crt \
  -subj "/CN=homelab.local"
```

Apply it as a kubernetes secret:
```
kubectl create secret tls argocd-tls \
  --cert=homelab.crt \
  --key=homelab.key \
  -n argocd
```


	

