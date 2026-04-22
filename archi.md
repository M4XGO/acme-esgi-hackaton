# Architecture — ACME Corp Hackathon (Itadaki)

## Stack déployée

| Couche | Technologie | Namespace |
|--------|-------------|-----------|
| CNI | Flannel (K3s défaut) | cluster-wide |
| Réseau secondaire | Multus (macvlan sur eth0) | cluster-wide |
| Visualisation réseau | Weave Scope :4040 | weave |
| Ingress controller | ingress-nginx (NodePort → LoadBalancer) | ingress-nginx |
| TLS | cert-manager + ca-issuer (self-signed CA) | cluster-wide |
| Frontend | Next.js :3000 | services |
| Backend | Java Spring Boot :8080 | services |
| Base de données | H2 (fichier persistant sur PVC) | services |
| Annuaire | OpenLDAP :389 (StatefulSet) | services |
| Logs | Loki + Promtail | monitoring |
| Métriques | Prometheus + Grafana + Alertmanager | monitoring |
| GitOps | ArgoCD | argocd |
| Backup K8s | Velero + Minio S3 | velero |

## URLs exposées via Ingress

| Service | URL | Port backend |
|---------|-----|-------------|
| Itadaki (app) | `https://itadaki.acme.test` | frontend:3000 / backend:8080 |
| Grafana | `https://grafana.acme.test` | :3000 |
| Prometheus | `https://prometheus.acme.test` | :9090 |
| Alertmanager | `https://alertmanager.acme.test` | :9093 |
| ArgoCD | `https://argocd.acme.test` | :8080 |
| Weave Scope | `https://scope.acme.test` | :4040 |

> DNS : entrée wildcard dans `/etc/hosts` → IP du nœud K3s (`make hosts`)
> Ingress-nginx : NodePort 30376 (HTTPS) / 31300 (HTTP), exposé via K3s ServiceLB

## Schéma d'architecture général

```mermaid
graph TD
    subgraph client["💻 Client (Mac)"]
        USER([Utilisateur\n*.acme.test])
    end

    subgraph ingress_ns["namespace: ingress-nginx"]
        INGRESS["ingress-nginx\nNodePort :30376 HTTPS\nNodePort :31300 HTTP\nServiceLB → :443/:80"]
    end

    subgraph services["namespace: services"]
        FRONT["itadaki-frontend\nNext.js :3000"]
        BACK["itadaki-backend\nJava :8080"]
        LDAP["openldap\n:389"]
        H2[("PVC itadaki-h2\n2Gi")]
        LDAP_D[("PVC ldap-data\n1Gi")]
        BK_PVC[("PVC backup\n10Gi")]
        CRON["CronJob h2-backup\n@ 2h"]
    end

    subgraph monitoring["namespace: monitoring"]
        GRAFANA["Grafana\n:3000"]
        PROM["Prometheus\n:9090"]
        AM["Alertmanager\n:9093"]
        LOKI["Loki\n:3100"]
        PROMTAIL["Promtail"]
    end

    subgraph argocd_ns["namespace: argocd"]
        ARGO["ArgoCD\n:8080"]
    end

    subgraph weave_ns["namespace: weave"]
        SCOPE["Weave Scope\n:4040"]
    end

    subgraph velero_ns["namespace: velero"]
        VEL["Velero"]
        MINIO[("Minio S3\n20Gi")]
    end

    %% Flux utilisateur → Ingress
    USER -->|"HTTPS :443"| INGRESS

    %% Ingress → apps
    INGRESS -->|"itadaki.acme.test /"| FRONT
    INGRESS -->|"itadaki.acme.test /api"| BACK
    INGRESS -->|"grafana.acme.test"| GRAFANA
    INGRESS -->|"prometheus.acme.test"| PROM
    INGRESS -->|"alertmanager.acme.test"| AM
    INGRESS -->|"argocd.acme.test"| ARGO
    INGRESS -->|"scope.acme.test"| SCOPE

    %% Flux services internes
    FRONT -->|":8080"| BACK
    BACK -->|"LDAP :389"| LDAP
    BACK --- H2
    LDAP --- LDAP_D
    CRON -->|"cp .mv.db"| BK_PVC

    %% Monitoring
    PROMTAIL -.->|"scrape logs"| FRONT
    PROMTAIL -.->|"scrape logs"| BACK
    PROMTAIL -->|"push"| LOKI
    PROM -.->|"scrape metrics :8080"| BACK
    PROM -.->|"scrape metrics"| INGRESS
    GRAFANA -->|"query"| PROM
    GRAFANA -->|"query"| LOKI
    PROM -->|"alertes"| AM

    %% Weave Scope observe tout
    SCOPE -.->|"observe pods/connexions"| services
    SCOPE -.->|"observe pods/connexions"| monitoring

    %% Velero backup
    VEL -->|"backup PVCs"| H2
    VEL -->|"backup PVCs"| LDAP_D
    VEL -->|"store"| MINIO

    %% Styles
    style client fill:#fee2e2,stroke:#ef4444
    style ingress_ns fill:#fef3c7,stroke:#f59e0b
    style services fill:#dbeafe,stroke:#3b82f6
    style monitoring fill:#d1fae5,stroke:#10b981
    style argocd_ns fill:#fce7f3,stroke:#ec4899
    style weave_ns fill:#e0f2fe,stroke:#0284c7
    style velero_ns fill:#ede9fe,stroke:#8b5cf6
```

## Flux réseau & NetworkPolicies

```mermaid
graph LR
    subgraph INET["🌐 Externe"]
        U([Client])
    end

    subgraph NI["ingress-nginx"]
        NG["nginx\n:443/:80"]
    end

    subgraph SVC["services ■ default-deny"]
        FE["frontend\n:3000"]
        BE["backend\n:8080"]
        OP["openldap\n:389"]
    end

    subgraph MON["monitoring ■ default-deny"]
        GR["grafana\n:3000"]
        PR["prometheus\n:9090"]
        AL["alertmanager\n:9093"]
        LK["loki"]
    end

    subgraph ACD["argocd"]
        AR["argocd-server\n:8080"]
    end

    subgraph WV["weave"]
        SC["scope\n:4040"]
    end

    %% Autorisé
    U -->|"✅ HTTPS"| NG
    NG -->|"✅ :3000"| FE
    NG -->|"✅ :8080"| BE
    NG -->|"✅ :3000"| GR
    NG -->|"✅ :9090"| PR
    NG -->|"✅ :9093"| AL
    NG -->|"✅ :8080"| AR
    NG -->|"✅ :4040"| SC
    FE -->|"✅ :8080"| BE
    BE -->|"✅ :389"| OP
    PR -.->|"✅ scrape"| BE
    PR -.->|"✅ scrape"| NG

    %% Bloqué
    U --->|"❌ direct"| FE
    U --->|"❌ direct"| BE
    U --->|"❌ direct"| OP
    FE --->|"❌ :389"| OP

    style INET fill:#fee2e2,stroke:#ef4444
    style NI fill:#fef3c7,stroke:#f59e0b
    style SVC fill:#dbeafe,stroke:#3b82f6
    style MON fill:#d1fae5,stroke:#10b981
    style ACD fill:#fce7f3,stroke:#ec4899
    style WV fill:#e0f2fe,stroke:#0284c7
```

## NetworkPolicies

| Règle | Source | Destination | Port | Namespace |
|-------|--------|-------------|------|-----------|
| allow-internet-to-ingress | 0.0.0.0/0 | ingress-nginx | 80, 443 | dmz |
| allow-dmz-to-itadaki-frontend | ingress-nginx | itadaki-frontend | 3000 | services |
| allow-dmz-to-itadaki-backend | ingress-nginx | itadaki-backend | 8080 | services |
| allow-frontend-to-backend | itadaki-frontend | itadaki-backend | 8080 | services |
| allow-backend-to-ldap | itadaki-backend | openldap | 389 | services |
| allow-monitoring-scrape | monitoring | services + dmz | 8080, 9100 | services |
| allow-ingress-to-grafana | ingress-nginx | grafana | 3000 | monitoring |
| allow-ingress-to-prometheus | ingress-nginx | prometheus | 9090 | monitoring |
| allow-ingress-to-alertmanager | ingress-nginx | alertmanager | 9093 | monitoring |
| allow-ingress-to-argocd | ingress-nginx | argocd-server | 8080 | argocd |
| allow-ingress-to-weave-scope | ingress-nginx | weave-scope-app | 4040 | weave |
| default-deny-all | — | dmz, services, monitoring | tout bloqué | * |

## Persistance des données

| Donnée | PVC | Taille | Survit à la suppression |
|--------|-----|--------|------------------------|
| H2 (Itadaki) | `itadaki-h2-pvc` | 2Gi | Oui |
| LDAP data | `ldap-data-openldap-0` | 1Gi | Oui |
| LDAP config | `ldap-config-openldap-0` | 100Mi | Oui |
| Backup H2 | `backup-pvc` | 10Gi | Oui |
| Loki logs | PVC Helm loki-stack | 5Gi | Oui |
| Minio (Velero) | `minio-pvc` | 20Gi | Oui |
