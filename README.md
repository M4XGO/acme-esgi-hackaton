# ACME Corp — Infrastructure Hackathon 48h

**Mission** : infrastructure d'entreprise fonctionnelle, sécurisée, observable et documentée pour ACME Corp (PME 50 salariés).

## Stack

| Couche | Choix |
|--------|-------|
| Cluster | K3s single-node |
| CNI | Flannel + Multus (macvlan) |
| Ingress / TLS | ingress-nginx + cert-manager (CA self-signed) |
| App métier | Itadaki — React Router 7 + Spring Boot 4 + H2 |
| Identité | OpenLDAP (3 groupes : admins, editors, viewers) |
| Observabilité | Prometheus + Grafana + Alertmanager + Loki + Promtail |
| Réseau visuel | Weave Scope |
| Backup | Velero + Kopia + Minio |

## Prérequis

- Cluster K3s opérationnel, `kubectl` configuré
- Docker Desktop avec `docker login` (pour `make build-push`)
- `helm` installé localement

## Déploiement

```bash
# 1. Configurer les secrets
cp .env.example .env
# Éditer .env avec les mots de passe souhaités

# 2. Installer Multus (une seule fois)
make multus-install

# 3. Infra complète
make all          # namespaces + secrets + ldap + monitoring + backup

# 4. Builder et publier les images
make build-push   # nécessite docker login (Docker Hub m4xgo/)

# 5. Déployer l'application
make app

# 6. Velero + Minio
make velero

# 7. Exposer les UIs via Ingress
make ingresses

# 8. NetworkPolicies EN DERNIER (quand tous les pods sont Running)
kubectl get pods -A
make netpol

# 9. DNS local
make hosts        # injecte les entrées *.acme.test dans /etc/hosts (sudo)
```

## URLs

| Service | URL |
|---------|-----|
| Itadaki (app) | https://itadaki.acme.test |
| Grafana | https://grafana.acme.test |
| Prometheus | https://prometheus.acme.test |
| Alertmanager | https://alertmanager.acme.test |
| Weave Scope | https://scope.acme.test |

> Certificats self-signed — accepter l'avertissement navigateur.

## Comptes de test

Mots de passe stockés dans le Secret K8s `ldap-users-secret` (namespace `services`) :

```bash
kubectl get secret ldap-users-secret -n services \
  -o jsonpath='{.data}' | python3 -c "
import sys,json,base64
d=json.load(sys.stdin)
[print(k+':', base64.b64decode(v).decode()) for k,v in d.items()]
"
```

| Utilisateur | Groupe | Email |
|-------------|--------|-------|
| admin1 | admins | admin1@acme.test |
| editor1 | editors | editor1@acme.test |
| viewer1 | viewers | viewer1@acme.test |

## Commandes utiles

```bash
make secrets          # recréer les Secrets K8s depuis .env
make netpol           # (ré)appliquer les NetworkPolicies
make test-netpol      # prouver le blocage réseau (jury)
make velero-backup-now  # backup immédiat pour la démo
make velero-list      # lister les backups disponibles
make restore          # restaurer depuis le dernier backup Velero
```

## Séquence de démonstration

1. **Architecture & zones** — diagramme dans la doc Word + Weave Scope (https://scope.acme.test)
2. **Identité** — comptes LDAP, 3 groupes, connexion à Itadaki avec rôles différenciés
3. **Application** — Itadaki : CRUD, auth JWT, logs, `/v3/api-docs` (healthcheck)
4. **HTTPS & accès** — ingress-nginx, cert-manager, NetworkPolicies actives
5. **Observabilité** — Grafana (dashboards + Loki logs), alertes Alertmanager
6. **Backup** — `make velero-backup-now`, `make velero-list`, `make restore`
7. **Redéploiement** — `make build-push && make app` depuis zéro

## Développement & maintenance de l'application

### Lancer l'app en local

```bash
# Backend (Spring Boot)
cd ../itadaki/backend
./mvnw spring-boot:run
# → http://localhost:8080 — H2 fichier ./data/itadaki, Swagger UI sur /swagger-ui.html

# Frontend (React Router + Bun)
cd ../itadaki/frontend
bun install
bun run dev
# → http://localhost:5173
```

### Modifier et redéployer

```bash
# 1. Modifier le code dans ../itadaki/backend ou ../itadaki/frontend

# 2. Rebuilder et pousser les images
cd esgi_hackaton
make build-push          # docker build --platform linux/amd64 + docker push

# 3. Forcer le redémarrage des pods (pull Always)
kubectl rollout restart deployment/itadaki-backend -n services
kubectl rollout restart deployment/itadaki-frontend -n services

# Ou tout en une commande :
make app                 # kubectl apply + rollout status
```

### Logs & debug

```bash
# Logs backend live
kubectl logs -f deployment/itadaki-backend -n services

# Logs frontend live
kubectl logs -f deployment/itadaki-frontend -n services

# Statut des pods
kubectl get pods -n services

# Describe en cas de CrashLoopBackOff
kubectl describe pod -l app=itadaki-backend -n services
```

### Secrets — modifier un mot de passe

```bash
# Éditer .env puis regénérer les Secrets K8s
make secrets

# Redémarrer le backend pour prendre en compte le nouveau JWT_SECRET
kubectl rollout restart deployment/itadaki-backend -n services
```

### Base de données H2

```bash
# Console H2 (accès via port-forward)
kubectl port-forward deployment/itadaki-backend 8080:8080 -n services
# → http://localhost:8080/h2-console
# JDBC URL : jdbc:h2:file:/data/itadaki  /  user: sa  /  password: (vide)
```

### Backup & restauration des données

```bash
# Backup immédiat (Velero)
make velero-backup-now

# Lister les backups
make velero-list

# Restaurer depuis un backup
make velero-restore BACKUP=demo-backup-YYYYMMDD-HHMM
```

## Structure du repo

```
Makefile
.env.example
scripts/
  gen-secrets.sh      # génère les Secrets K8s depuis .env
  bootstrap.sh        # ordre complet depuis zéro
  test-netpol.sh      # preuves jury de blocage réseau
k8s/
  namespaces/         # dmz, services, monitoring (labels zone=*)
  multus/             # NADs macvlan (dmz, services, users)
  ldap/               # StatefulSet OpenLDAP + bootstrap LDIF
  itadaki/            # Deployment frontend+backend, Services, Ingress, PVCs
  monitoring/         # loki-values.yaml, alertmanager-rules.yaml, datasource
  networkpolicies/    # default-deny × 3 + allow × 16
  ingresses/          # Grafana, Prometheus, Alertmanager, Weave Scope
  backup/             # Velero schedule, PVC backup
  velero/             # Minio + Helm values Velero
docs/
  test-credentials.md # comptes LDAP jury
```
