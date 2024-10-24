# HashiCorp Vault Secrets Operator (VSO) PKI Tutorial 

The purpose of this tutorial is to show automated certificate rotation with a working example.  Here we will generate a TLS certificate and apply this to the ngnix sample hello-world application

These are the software versions used in this tutorial

```shell-session
Minikube Version: v1.31.2
Ubuntu Version 22.04 LTS
Helm Version: v3.12.3
```


**To begin first Complete The VSO tutorial** (https://developer.hashicorp.com/vault/tutorials/kubernetes/vault-secrets-operator) **up until "Deploy and sync a secret".**  

Next Configure PKI secrets engine

```shell-session
kubectl exec --stdin=true --tty=true vault-0 -n vault -- /bin/sh
```

```shell-session
vault secrets enable pki
```

```shell-session
vault secrets tune -max-lease-ttl=8760h pki
```

```shell-session
vault write pki/root/generate/internal \
    common_name=example.com \
    ttl=8760h
```

```shell-session
vault write pki/config/urls \
    issuing_certificates="http://vault.default:8200/v1/pki/ca" \
    crl_distribution_points="http://vault.default:8200/v1/pki/crl"
```

```shell-session
vault write pki/roles/example-dot-com \
    allowed_domains=example.com \
    allow_subdomains=true \
    max_ttl=72h
```
```shell-session
vault policy write pki - <<EOF
path "pki*"                        { capabilities = ["create", "read", "update", "list"] }
EOF
```

```shell-session
vault write auth/demo-auth-mount/role/issuer \
    bound_service_account_names=issuer \
    bound_service_account_namespaces=default \
    policies=pki \
    ttl=20m
```

```shell-session
exit
```

**Now we setup the authentication for VSO & create the PKI Secret**

```shell-session
cat > vault-auth-static2.yaml <<EOF 
apiVersion: secrets.hashicorp.com/v1beta1
kind: VaultAuth
metadata:
  name: static-auth2
  namespace: default
spec:
  method: kubernetes
  mount: demo-auth-mount
  kubernetes:
    role: issuer
    serviceAccount: issuer
    audiences:
      - vault
EOF
```

```shell-session
kubectl apply -f vault-auth-static2.yaml
```

```shell-session
cat > vault-pki-secret.yaml <<EOF 
apiVersion: secrets.hashicorp.com/v1beta1
kind: VaultPKISecret
metadata:
  name: pki-test1
  namespace: default
spec:
  vaultAuthRef: static-auth2
  namespace: default
  mount: pki
  role: example-dot-com
  destination:
    create: true
    name: demo-example-com-tls
    type: kubernetes.io/tls
  commonName: demo.example.com
  format: pem
  revoke: true
  clear: true
  expiryOffset: 30s
  ttl: 5m
EOF
```

```shell-session
kubectl apply -f vault-pki-secret.yaml
```



Create the issuer k8s service account

```shell-session
kubectl create serviceaccount issuer
```


Now we will enable ingress on Minikube

```shell-session
minikube addons enable ingress
```

We will create a deplyoment called ngnix-demo

```shell-session
kubectl create deployment nginx-demo --image=nginxdemos/hello
```

Verify the deployment is ready

```shell-session
kubectl get deployment
```
Expose the deployment

```shell-session
kubectl expose deployment nginx-demo --port=80
```

Apply the Ingress to allow traffic into our application

```shell-session
cat > ingress.yaml <<EOF 
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: nginx-demo
spec:
  rules:
    - host: demo.example.com
      http:
        paths:
          - pathType: Prefix
            path: "/"
            backend:
             service:
               name: nginx-demo
               port:
                 number: 80
EOF
```

Apply the ingress

```shell-session
kubectl apply -f ingress.yaml
```
Now we need to get the ingress IP address to update our hosts file

```shell-session
kubectl get ingress
```

Output

```shell-session
NAME         CLASS   HOSTS              ADDRESS          PORTS     AGE
nginx-demo   nginx   demo.example.com   192.168.59.100   80, 443   15m
```

Update our hosts file to point example.com to the address in the output of the command above

```shell-session
sudo nano /etc/hosts
```

At the bottom of the file add, replacing the IP address below with the output of the command "kubectl get ingress" 

```shell-session
192.168.59.100 demo.example.com
```

Now use curl to inspect the certificate.  Note the CN=Kubernetes Ingress Controller Fake Certificate.  This is a default certificate, we will replace this in the following steps.

```shell-session
curl --insecure -vvI https://demo.example.com 2>&1 | awk 'BEGIN { cert=0 } /^\* SSL connection/ { cert=1 } /^\*/ { if (cert) print }'
```

The Output should look somthing like this

```shell-session
* SSL connection using TLSv1.3 / TLS_AES_256_GCM_SHA384
* ALPN, server accepted to use h2
* Server certificate:
*  subject: O=Acme Co; CN=Kubernetes Ingress Controller Fake Certificate
*  start date: Aug 22 15:37:42 2023 GMT
*  expire date: Aug 21 15:37:42 2024 GMT
*  issuer: O=Acme Co; CN=Kubernetes Ingress Controller Fake Certificate

```

**Now we will apply the Ingress patch to use TLS and the HashiCorp Vault Generated Certificate**

```shell-session
cat > ingress-patch.yaml <<EOF
spec:
  tls:
  - hosts:
    - demo.example.com
    secretName: demo-example-com-tls
EOF
```

We will apply the patch to the nginx-demo ingress controller

```shell-session
kubectl patch ingress nginx-demo --patch-file=ingress-patch.yaml
```

We can run the curl command again to see CN=example.com

```shell-session
curl --insecure -vvI https://demo.example.com 2>&1 | awk 'BEGIN { cert=0 } /^\* SSL connection/ { cert=1 } /^\*/ { if (cert) print }'
```

Output

```shell-session
* SSL connection using TLSv1.3 / TLS_AES_256_GCM_SHA384
* ALPN, server accepted to use h2
* Server certificate:
*  subject: CN=demo.example.com
*  start date: Aug 22 19:28:47 2023 GMT
*  expire date: Aug 25 19:29:17 2023 GMT
*  issuer: CN=example.com
```

We can see now we are now using the Vault issued certificate.  We can also view this via a web browser, it will also be rotated automatically.

**To View the Vault UI**

Setup port forwarding on a new Terminal Window

```shell-session
kubectl port-forward svc/vault 8200:8200 -n vault
```
The Vault UI will now be available on http://127.0.0.1:8200 & the token is "root"

You can take a look at the PKI secrets engine configuration and the certificates generated by Vault
