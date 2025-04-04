# Installing YOURLS on Kubernetes

This guide walks through the process of deploying YOURLS (Your Own URL Shortener) on a Kubernetes cluster using Helm.

## Prerequisites

- A working Kubernetes cluster
- Helm installed on your machine
- Basic knowledge of Kubernetes concepts
- A domain name pointed to your cluster's ingress controller (we'll use `go.example.com` in this guide)

## Installation Steps

### 1. Create a Namespace for YOURLS

First, create a dedicated namespace for YOURLS:

```bash
kubectl create namespace yourls
```

### 2. Configure Values for YOURLS

Create a `values.yaml` file with the following configuration:

```yaml
yourls:
  username: "admin"           # Change to your desired admin username
  password: "securepassword"  # Change to a secure password
  cookieKey: "randomsecurestring"  # Use a random string for cookie encryption
  # The following two settings are critical for proper site URL configuration
  scheme: "https"
  domain: "go.example.com"

mariadb:
  enabled: true               # Use built-in MariaDB
  auth:
    username: "yourlsdb"      # Database username
    password: "dbpassword"    # Database password
    database: "yourls"        # Database name
    rootPassword: "rootpass"  # Root password for MariaDB

persistence:
  enabled: true               # Enable persistent storage
```

### 3. Install YOURLS using Helm

Deploy YOURLS using the Helm chart with your custom values:

```bash
helm upgrade -f values.yaml --install yourls yourls/yourls --namespace yourls
```

### 4. Set Up TLS with cert-manager

Install cert-manager to handle TLS certificates:

```bash
kubectl create namespace cert-manager
helm install cert-manager bitnami/cert-manager --namespace cert-manager --set installCRDs=true
```

Create a ClusterIssuer for Let's Encrypt:

```yaml
# letsencrypt-prod.yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: admin@example.com  # Change to your email
    privateKeySecretRef:
      name: letsencrypt-prod
    solvers:
    - http01:
        ingress:
          class: nginx
```

Apply the ClusterIssuer:

```bash
kubectl apply -f letsencrypt-prod.yaml
```

### 5. Configure Ingress for YOURLS

Create an ingress configuration for YOURLS:

```yaml
# yourls-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: yourls
  namespace: yourls
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - go.example.com
      secretName: go-example-com-tls
  rules:
    - host: go.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: yourls
                port:
                  number: 80
```

Apply the ingress configuration:

```bash
kubectl apply -f yourls-ingress.yaml
```

## Testing Your Installation

### 1. Verify Pod Status

Check that the YOURLS and MariaDB pods are running:

```bash
kubectl get pods -n yourls
```

Expected output:
```
NAME                      READY   STATUS    RESTARTS   AGE
yourls-xxxxxxxxxx-xxxxx   1/1     Running   0          3m
yourls-mariadb-0          1/1     Running   0          3m
```

### 2. Check Environment Variables

Verify that the YOURLS_SITE environment variable is set correctly:

```bash
kubectl exec -it -n yourls deploy/yourls -- env | grep YOURLS_SITE
```

Expected output:
```
YOURLS_SITE=https://go.example.com
```

### 3. Test TLS Certificate

Verify that the SSL certificate is properly issued:

```bash
kubectl get certificate -n yourls
```

Test the HTTPS connection:

```bash
curl -Iv https://go.example.com
```

You should see a successful HTTPS response with a valid certificate.

### 4. Access YOURLS Admin Interface

Open your browser and navigate to `https://go.example.com/admin` to access the YOURLS admin interface.

Login using the credentials you specified in the values.yaml file.

## Troubleshooting

### YOURLS_SITE Not Set Correctly

If the YOURLS_SITE environment variable is not being set correctly, make sure you've configured both `scheme` and `domain` in your values.yaml:

```yaml
yourls:
  scheme: "https"
  domain: "go.example.com"
```

### Ingress Not Working

Check the status of your ingress:

```bash
kubectl describe ingress yourls -n yourls
```

Ensure your DNS records are correctly pointing to your Kubernetes cluster's ingress controller.

### Certificate Issues

Check certificate status:

```bash
kubectl describe certificate go-example-com-tls -n yourls
```

Look for any errors in the cert-manager logs:

```bash
kubectl logs -n cert-manager -l app=cert-manager
```

## Uninstallation

To completely remove YOURLS from your cluster:

```bash
# Uninstall the YOURLS Helm release
helm uninstall yourls -n yourls

# Delete any persistent volume claims
kubectl delete pvc --all -n yourls

# Delete the namespace
kubectl delete namespace yourls
```

Optional: Remove cert-manager if it's no longer needed:

```bash
helm uninstall cert-manager -n cert-manager
kubectl delete namespace cert-manager
```

## Notes

- Remember to back up your MariaDB data before making major changes to your deployment
- For production use, consider using external database services instead of the bundled MariaDB
- Keep your Kubernetes cluster and YOURLS installation updated to maintain security
