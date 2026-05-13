# ingress-controller

Infrastructure Kubernetes partagée entre toutes les apps du VPS. Remplace l'ancien `nginx-edge-proxy` (Docker).

## Architecture

```
                        ┌─────────────────────────────────────────────┐
                        │               PROD (k3s)                    │ 
                        │                                             │
  Internet              │   ┌──────────────────────────────────────┐  │
  (80 / 443) ─────────────▶   Traefik  (intégré à k3s)                
                        │   │  Ingress Controller                  │  │
                        │   └──────────┬───────────────────────────┘  │
                        │              │ hostname match               │
                        │   ┌──────────▼──────────────────────────┐   │
                        │   │         cert-manager                │   │
                        │   │  (provisionne TLS automatiquement)  │   │
                        │   └──────────┬──────────────────────────┘   │
                        │              │ ClusterIssuer                │
                        │   ┌──────────▼──────────────────────────┐   │
                        │   │   Let's Encrypt (ACME HTTP-01)      │   │
                        │   └─────────────────────────────────────┘   │
                        │                                             │
                        │   ┌──────────┐  ┌──────────┐  ┌──────────┐  │
                        │   │  app-A   │  │  app-B   │  │  app-N   │  │
                        │   │namespace │  │namespace │  │namespace │  │
                        │   │          │  │          │  │          │  │
                        │   │ Service  │  │ Service  │  │ Service  │  │
                        │   │ClusterIP │  │ClusterIP │  │ClusterIP │  │
                        │   │  Pods    │  │  Pods    │  │  Pods    │  │
                        │   │Ingress ──┘  │Ingress ──┘  │Ingress ──┘  │
                        │   │  .yaml      │  .yaml      │  .yaml      │
                        │   └──────────┘  └──────────┘  └──────────┘  │
                        └─────────────────────────────────────────────┘

                        ┌─────────────────────────────────────────────┐
                        │            DEV LOCAL (kind)                 │
                        │                                             │
  localhost             │   ┌──────────────────────────────────────┐  │
  :8080 / :8443 ───────────▶  Traefik  (DaemonSet via Helm)       
                        │   │  hostPort 80/443 → kind node         │  │
                        │   └──────────┬───────────────────────────┘  │
                        │              │ pas de cert-manager          │
                        │              │ pas de TLS                   │
                        │   ┌──────────▼──────────────────────────┐   │
                        │   │        Apps (HTTP only)             │   │
                        │   │  ssl-redirect: "false"              │   │
                        │   └─────────────────────────────────────┘   │
                        └─────────────────────────────────────────────┘
```

**Ce repo contient :** Traefik (prod k3s), cert-manager, ClusterIssuer Let's Encrypt.  
**Chaque repo d'app contient :** son propre `k8s/ingress.yaml` avec ses règles de routing.

---

## Responsabilités par repo

| Ce repo (`ingress-controller`) | Chaque repo d'app |
|---|---|
| Traefik (prod k3s — déjà inclus) | `k8s/ingress.yaml` avec hostname |
| cert-manager (install) | Annotation `cert-manager.io/cluster-issuer: letsencrypt-prod` |
| `cluster-issuer.yaml` (Let's Encrypt) | Son namespace, ses Pods, son Service |
| Helm values Traefik pour kind (dev) | `kind-config.yaml` (port mapping) |

---

## Prod (k3s)

### 1. Traefik

Traefik est **déjà installé** avec k3s — rien à faire.

### 2. cert-manager

```bash
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/latest/download/cert-manager.yaml
kubectl wait --namespace cert-manager \
  --for=condition=ready pod \
  --selector=app.kubernetes.io/instance=cert-manager \
  --timeout=120s
```

### 3. ClusterIssuer Let's Encrypt

```bash
kubectl apply -f cluster-issuer.yaml
kubectl get clusterissuer letsencrypt-prod
```

### 4. Déployer les apps

Chaque app s'occupe de son namespace et de son Ingress. Les certificats TLS sont provisionnés automatiquement par cert-manager au premier accès.

---

## Dev local (kind)

### 1. Créer le cluster kind

Utilise le `kind-config.yaml` du repo de l'app (mappage ports 80→8080, 443→8443).

```bash
kind create cluster --name dev --config kind-config.yaml
```

### 2. Installer Traefik via Helm

```bash
helm repo add traefik https://traefik.github.io/charts
helm repo update
helm install traefik traefik/traefik \
  --namespace traefik \
  --create-namespace \
  -f install/traefik-kind-values.yaml
```

> En local, pas de cert-manager ni de TLS. Désactiver le ssl-redirect dans les Ingress des apps (`nginx.ingress.kubernetes.io/ssl-redirect: "false"`).

---

## Flux TLS (prod)

```
Ingress resource (app repo)
  └── annotation: cert-manager.io/cluster-issuer: letsencrypt-prod
        ↓
  cert-manager détecte l'annotation
        ↓
  ClusterIssuer "letsencrypt-prod" (cluster-issuer.yaml)
        ↓
  Challenge ACME HTTP-01 via Traefik
        ↓
  Let's Encrypt valide → émet le certificat
        ↓
  Secret TLS stocké dans le namespace de l'app
        ↓
  Traefik sert HTTPS avec ce certificat
```


