# Kubernetes Ingress TLS with Cert-Manager and Let's Encrypt

This guide provides step-by-step instructions to configure automatic TLS certificate issuance for Kubernetes Ingress resources using Cert-Manager and Let's Encrypt.

## Prerequisites

- A Kubernetes cluster
- `kubectl` command-line tool configured to communicate with your cluster
- Cert-Manager installed in your cluster
- **Ingress Controller installed in your cluster**: Ensure you have an Ingress Controller, like Nginx or Traefik, installed. If you're using Nginx, you can follow the [official Nginx Ingress Controller installation guide](https://kubernetes.github.io/ingress-nginx/deploy/).
- **DNS A Records**: Ensure you have DNS A records pointing to your Ingress Controller's external IP for each of the domains you want to secure with TLS. This is necessary for Let's Encrypt to verify domain ownership during the ACME challenge process.

## Step 1: Install Cert-Manager

If you haven't installed Cert-Manager in your Kubernetes cluster, follow the [official Cert-Manager installation guide](https://cert-manager.io/docs/installation/kubernetes/).

## Step 2: Create a ClusterIssuer

A ClusterIssuer is a Kubernetes resource that represents a certificate authority from which to obtain certificates. In this case, we'll use Let's Encrypt.

1. Create a file named `cluster-issuer.yaml` with the following content:

    ```yaml
    apiVersion: cert-manager.io/v1
    kind: ClusterIssuer
    metadata:
      name: letsencrypt-cluster-issuer
    spec:
      acme:
        server: https://acme-v02.api.letsencrypt.org/directory # Let's Encrypt ACME server for production
        email: your-email@example.com
        privateKeySecretRef:
          name: letsencrypt-cluster-issuer-key
        solvers:
        - http01:
            ingress:
              class: nginx
    ```

    Replace `your-email@example.com` with your actual email address.

2. Apply the configuration:

    ```bash
    kubectl apply -f cluster-issuer.yaml
    ```

## Step 3: Create Certificates

Create Kubernetes Certificate resources to specify the details about the certificates you want Cert-Manager to manage.

1. Create a file named `certificates.yaml` with the following content:

    ```yaml
    ---
    apiVersion: cert-manager.io/v1
    kind: Certificate
    metadata:
      name: letsencrypt-certificate-one
      namespace: example-namespace
    spec:
      dnsNames:
        - domain.one.com
      secretName: letsencrypt-certificate-secret-one
      issuerRef:
        name: letsencrypt-cluster-issuer
        kind: ClusterIssuer
    ---
    apiVersion: cert-manager.io/v1
    kind: Certificate
    metadata:
      name: letsencrypt-certificate-two
      namespace: example-namespace
    spec:
      dnsNames:
        - domain.two.com
      secretName: letsencrypt-certificate-secret-two
      issuerRef:
        name: letsencrypt-cluster-issuer
        kind: ClusterIssuer
    ```

2. Apply the certificates:

    ```bash
    kubectl apply -f certificates.yaml
    ```

## Step 4: Configure Ingress to Use the Certificates

Modify your Ingress resources to specify the use of the TLS certificates managed by Cert-Manager.

1. Create a file named `ingress.yaml` with the following content:

    ```yaml
    apiVersion: networking.k8s.io/v1
    kind: Ingress
    metadata:
      name: ingress-for-domains
      namespace: example-namespace
      annotations:
        kubernetes.io/ingress.class: nginx
        cert-manager.io/cluster-issuer: "letsencrypt-cluster-issuer"
    spec:
      ingressClassName: nginx
      tls:
      - hosts:
          - domain.one.com
        secretName: letsencrypt-certificate-secret-one
      - hosts:
          - domain.two.com
        secretName: letsencrypt-certificate-secret-two
      rules:
      - host: domain.one.com
        http:
          paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: domain-one-service
                port:
                  number: 3535
      - host: domain.two.com
        http:
          paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: domain-two-service
                port:
                  number: 3535
    ```

2. Apply the Ingress configuration:

    ```bash
    kubectl apply -f ingress.yaml
    ```

## Verification

To verify that the certificates have been successfully issued and are in use:

1. Check the status of the Certificate resources:

    ```bash
    kubectl describe certificate letsencrypt-certificate-one -n example-namespace
    kubectl describe certificate letsencrypt-certificate-two -n example-namespace
    ```

2. Access your domains via HTTPS and verify that the browser shows a valid certificate.

## Conclusion

You have successfully configured your Kubernetes Ingress resources to automatically obtain and renew TLS certificates from Let's Encrypt using Cert-Manager. This setup ensures secure, encrypted traffic to your services.
