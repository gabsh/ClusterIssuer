# ingress-controller

Infrastructure Kubernetes partagée entre toutes les apps du VPS. Remplace l'ancien `nginx-edge-proxy` (Docker).

## Architecture

```
Internet (80/443)
  ↓
Traefik (Ingress Controller)
  ↓  hostname match via Ingress resource dans chaque namespace d'app
Service ClusterIP de chaque app
  ↓
Pods
```

**Ce repo contient :** Traefik (prod k3s), cert-manager, ClusterIssuer Let's Encrypt.  
**Chaque repo d'app contient :** son propre `k8s/ingress.yaml` avec ses règles de routing.

---

## Prod (k3s)

### 1. Traefik
Traefik est **déjà installé** avec k3s — rien à faire.

### 2. cert-manager

```bash
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/latest/download/cert-manager.yaml
kubectl wait --namespace cert-manager --for=condition=ready pod --selector=app.kubernetes.io/instance=cert-manager --timeout=120s
```

### 3. ClusterIssuer Let's Encrypt

```bash
kubectl apply -f cluster-issuer.yaml
```

Vérifie qu'il est prêt :
```bash
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

## Ajouter une nouvelle app

1. Créer un namespace pour l'app : `kubectl create namespace <app>`
2. Ajouter un `k8s/ingress.yaml` dans le repo de l'app avec le bon hostname
3. L'annotation `cert-manager.io/cluster-issuer: letsencrypt-prod` dans l'Ingress suffit pour avoir le TLS automatiquement
