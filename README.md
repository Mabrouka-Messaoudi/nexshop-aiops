# NexShop AIOps – Supervision intelligente d'une plateforme Kubernetes

**Projet de fin d'études · Data Engineering · Mabrouka Messaoudi · 2026**

NexShop est une plateforme e-commerce en microservices déployée sur Kubernetes. Ce projet remplace sa
supervision par alertes Grafana à seuils fixes par quatre modèles de machine learning, qui répondent aux
questions d'un ingénieur d'astreinte :

| | Question | Modèle |
|---|----------|--------|
| **M1** | Ce pod se comporte-t-il anormalement ? | Local Outlier Factor |
| **M2** | De quelle panne s'agit-il ? | Gradient Boosting, 5 classes |
| **M3** | Dans combien de temps va-t-il tomber ? | Gradient Boosting (régression du temps avant OOM) |
| **M4** | Cette valeur est-elle normale à ce moment ? | ARIMA par service et par métrique |

Ce dépôt est le point d'entrée du projet. Le code est réparti dans les dépôts ci-dessous, un par couche
du système.

## Architecture

```
 Infrastructure     Application        Observabilité       Données             Modèles            Plateforme
┌──────────────┐   ┌──────────────┐   ┌──────────────┐   ┌──────────────┐   ┌──────────────┐   ┌──────────────┐
│ cluster-k8s  │──►│ microservices│──►│ monitoring-  │──►│ nexshop-data-│──►│ nexshop-     │──►│ nexshop-     │
│ 4 nœuds      │   │ -java-k8s-   │   │ stack        │   │ collection   │   │ aiops-models │   │ aiops-       │
│ VirtualBox + │   │ monitoring   │   │ Prometheus,  │   │ k6, pannes   │   │ M1 M2 M3 M4  │   │ platform     │
│ KVM          │   │ Spring, Kafka│   │ Loki, Tempo  │   │ injectées    │   │ CRISP-DM     │   │ FastAPI +    │
└──────────────┘   └──────────────┘   └──────────────┘   └──────────────┘   └──────────────┘   │ cockpit React│
                   └───────── nexshop-gke : migration vers GKE ─────────┘                      └──────────────┘
```

## Dépôts, dans l'ordre de lecture

| # | Dépôt | Contenu |
|---|-------|---------|
| 1 | [cluster-k8s](https://github.com/Mabrouka-Messaoudi/cluster-k8s) | Cluster Kubernetes hybride sur deux machines (Vagrant + Ansible, kubeadm 1.29, Flannel) |
| 2 | [microservices-java-k8s-monitoring](https://github.com/Mabrouka-Messaoudi/microservices-java-k8s-monitoring) | Les 5 microservices Spring Boot et le frontend Angular, instrumentés (métriques, logs, traces), manifests et CI |
| 3 | [monitoring-stack](https://github.com/Mabrouka-Messaoudi/monitoring-stack) | Prometheus, Grafana, Loki, Promtail et Tempo en charts Helm, sur un nœud dédié |
| 4 | [nexshop-gke](https://github.com/Mabrouka-Messaoudi/nexshop-gke) | Migration de l'application et du monitoring vers Google Kubernetes Engine |
| 5 | [nexshop-data-collection](https://github.com/Mabrouka-Messaoudi/nexshop-data-collection) | Construction du dataset : profils de charge k6, injection de 4 pannes, collecteur labellisé |
| 6 | [nexshop-aiops-models](https://github.com/Mabrouka-Messaoudi/nexshop-aiops-models) | Les quatre modèles, un dossier de notebooks CRISP-DM chacun |
| 7 | [nexshop-aiops-platform](https://github.com/Mabrouka-Messaoudi/nexshop-aiops-platform) | API FastAPI qui enchaîne les modèles et cockpit React de démonstration |
| — | [microservices-angular](https://github.com/Mabrouka-Messaoudi/microservices-angular) | Annexe : première version du frontend, en local |

## Résultats

| Modèle | Résultat principal |
|--------|--------------------|
| M1 | Rappel de 100 % sur OOM, CrashLoop et CPU throttle, 92,7 % sur DB Down. Fausses alertes sur les pods voisins de la panne : de 85,9 % à 11,9 %. |
| M2 | F1 macro 0,987, accuracy 98,6 % sur 283 lignes de test. |
| M3 | Erreur moyenne de 61,6 s sur le temps avant crash (validation run par run), contre 382 s pour une extrapolation naïve. |
| M4 | Fausses alertes sous forte charge : de 58,8 % avec un seuil fixe à 4,8 % avec les seuils ARIMA. |

## Comment le système fonctionne

1. Les microservices exposent leurs métriques, leurs logs et leurs traces. Prometheus, Loki et Tempo les
   collectent.
2. Pour construire le dataset, des scénarios génèrent de la charge (k6) et injectent des pannes réelles
   dans le cluster : manque de mémoire, sonde de santé cassée, base de données coupée, CPU limité. Un
   collecteur interroge les trois sources toutes les 10 secondes et labellise chaque ligne au moment de
   sa collecte.
3. Les modèles sont entraînés selon la démarche CRISP-DM, puis servis par une API qui les enchaîne : M1
   détecte, M2 identifie la panne, M3 estime le temps restant en cas d'OOM, M4 évalue chaque métrique par
   rapport à son contexte récent.
4. Le cockpit rejoue des pannes réelles, y compris sans connaître la panne à l'avance (mode mystère), et
   affiche la décision de chaque modèle en direct.
