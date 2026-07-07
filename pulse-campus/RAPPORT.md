# RAPPORT TECHNIQUE — TP 3 : LIVRAISON PROGRESSIVE ET OBSERVABILITÉ (PULSE-CAMPUS)
**Cours d'architecture logicielle & déploiement — ESGI 5 SRC**  
**Binôme :** alexisprk  
**Dépôt de configuration :** [github.com/alexisprk/tp-argocd.git](https://github.com/alexisprk/tp-argocd.git)  

---

## Étape 0 — Outillage complémentaire

### Livrable : Sortie des versions des CLI requises

```text
$ kubectl argo rollouts version
kubectl-argo-rollouts: v1.7.1+6a99ea9
  BuildDate: 2024-06-24T22:53:28Z
  GitCommit: 6a99ea9908e8f1e816ccd71e4c35adbbbbdd5f6c
  Platform: windows/amd64

$ promtool --version
promtool, version 2.53.1 (branch: HEAD, revision: 14cfec3f6048b735e08c1e9c64c8d4211d32bab4)
  build date:       20240710-10:18:30
  platform:         windows/amd64

$ jq --version
jq-1.7.1
```

---

## Étape 1 — SLI, SLO, error budget : Le vocabulaire avant le clavier

### 1. Tableaux de définition des SLI / SLO par service

#### Service : `annuaire` (Node.js)
| SLI | SLO | Justification du seuil | Error budget mensuel |
| :--- | :--- | :--- | :--- |
| **Disponibilité** : Taux de requêtes HTTP réussies (non 5xx) sur 30 jours | **99.5%** | Ce service gère les requêtes métier d'affichage de l'annuaire. Un taux de 99.5% offre un bon compromis pour le confort utilisateur tout en autorisant des mises à jour fréquentes. | **216 minutes** (3h 36m) |
| **Latence** : Latence au 95e percentile inférieure à 300ms sur 30 jours | **p95 < 300ms** | L'utilisateur s'attend à un affichage instantané des fiches de l'annuaire. Au-delà de 300ms, le ralentissement devient perceptible. | **216 minutes** sous le seuil |
| **Taux d'erreur SQL** : Proportion de requêtes retournant une erreur de base de données | **< 0.1%** | Les erreurs de base de données indiquent des requêtes mal écrites ou des indisponibilités de connexion avec PostgreSQL. | **43.2 minutes** |

#### Service : `planning` (Python FastAPI)
| SLI | SLO | Justification du seuil | Error budget mensuel |
| :--- | :--- | :--- | :--- |
| **Disponibilité** : Taux de requêtes HTTP réussies (non 5xx) sur 30 jours | **99.0%** | Service de calcul de créneaux plus complexe. Un SLO légèrement inférieur de 99% permet des périodes de calcul lourd sans pénaliser drastiquement l'error budget. | **432 minutes** (7h 12m) |
| **Latence** : Latence au 95e percentile inférieure à 500ms sur 30 jours | **p95 < 500ms** | Le calcul de chevauchement de créneaux est plus lourd qu'un simple CRUD de l'annuaire. 500ms est un seuil d'attente tolérable pour l'utilisateur. | **432 minutes** sous le seuil |
| **Taux de timeout externe** : Proportion d'appels vers l'annuaire se terminant en timeout | **< 1%** | Le planning a besoin de valider l'existence des intervenants auprès de l'annuaire. Les échecs réseau doivent rester minimes. | **432 minutes** |

#### Service : `notif` (Go)
| SLI | SLO | Justification du seuil | Error budget mensuel |
| :--- | :--- | :--- | :--- |
| **Disponibilité** : Taux de réussite du traitement des événements de notification | **99.9%** | Le service de notification tourne en arrière-plan et doit être extrêmement fiable pour ne perdre aucun événement important (Slack/Email). | **43.2 minutes** (43m 12s) |
| **Latence de traitement** : Délai entre la réception de l'événement et son traitement < 2s | **p95 < 2s** | Les notifications n'ont pas besoin d'être instantanées à la milliseconde près, mais un délai de traitement de plus de 2s montre une surcharge de la file. | **43.2 minutes** sous le seuil |
| **Taux de perte** : Taux d'événements abandonnés ou non délivrés | **0% (strict)** | Perdre une notification de sécurité ou de changement de salle est critique. Les erreurs d'envois doivent être rejouées automatiquement. | **0 minutes** |

---

### 2. Pseudo-code PromQL des SLI

#### Disponibilité (`annuaire`)
```promql
sum(rate(http_requests_total{service="annuaire", status_class!~"5.."}[5m]))
/
sum(rate(http_requests_total{service="annuaire"}[5m]))
```

#### Latence p95 (`annuaire`)
```promql
histogram_quantile(
  0.95,
  sum(rate(http_request_duration_seconds_bucket{service="annuaire"}[5m])) by (le)
)
```

#### Disponibilité (`planning`)
```promql
sum(rate(http_requests_total{service="planning", status_class!~"5.."}[5m]))
/
sum(rate(http_requests_total{service="planning"}[5m]))
```

#### Latence p95 (`planning`)
```promql
histogram_quantile(
  0.95,
  sum(rate(http_request_duration_seconds_bucket{service="planning"}[5m])) by (le)
)
```

#### Disponibilité (`notif`)
```promql
sum(rate(http_requests_total{service="notif", status_class!~"5.."}[5m]))
/
sum(rate(http_requests_total{service="notif"}[5m]))
```

#### Latence p95 (`notif`)
```promql
histogram_quantile(
  0.95,
  sum(rate(http_request_duration_seconds_bucket{service="notif"}[5m])) by (le)
)
```

---

## Étape 2 — Lire, comprendre et configurer l’instrumentation Prometheus

### 1. Choix des buckets configurés dans `values.yaml` (metrics.buckets)
Les buckets ont été calibrés de façon à encadrer précisément les SLOs de latence définis à l'Étape 1 afin de garantir une interpolation linéaire exacte pour le calcul des quantiles :
* **Service `annuaire`** (SLO p95 < 300ms) : Buckets `0.05, 0.1, 0.2, 0.3, 0.5, 1, 2`. Nous avons inséré un bucket exact à `0.3` (300ms) pour surveiller le respect direct du SLO.
* **Service `planning`** (SLO p95 < 500ms) : Buckets `0.05, 0.1, 0.2, 0.3, 0.5, 1, 2, 5`. Le bucket à `0.5` (500ms) cible l'exactitude à la limite de notre SLO.
* **Service `notif`** (SLO p95 < 2s) : Buckets `0.05, 0.1, 0.5, 1, 2, 5, 10`. Le bucket à `2` (2 secondes) cible la limite de traitement de la file.

### 2. Capture de la sortie `curl /metrics`
Voici un extrait de la sortie du endpoint `/metrics` validée avec `promtool` montrant les buckets effectifs sur le service `annuaire` :

```text
# HELP http_request_duration_seconds Durée des requêtes HTTP en secondes.
# TYPE http_request_duration_seconds histogram
http_request_duration_seconds_bucket{le="0.05",method="GET",route="/students",status_class="2xx"} 1
http_request_duration_seconds_bucket{le="0.1",method="GET",route="/students",status_class="2xx"} 1
http_request_duration_seconds_bucket{le="0.2",method="GET",route="/students",status_class="2xx"} 1
http_request_duration_seconds_bucket{le="0.3",method="GET",route="/students",status_class="2xx"} 1
http_request_duration_seconds_bucket{le="0.5",method="GET",route="/students",status_class="2xx"} 1
http_request_duration_seconds_bucket{le="1",method="GET",route="/students",status_class="2xx"} 1
http_request_duration_seconds_bucket{le="2",method="GET",route="/students",status_class="2xx"} 1
http_request_duration_seconds_bucket{le="+Inf",method="GET",route="/students",status_class="2xx"} 1
http_request_duration_seconds_sum{method="GET",route="/students",status_class="2xx"} 0.004065923
http_request_duration_seconds_count{method="GET",route="/students",status_class="2xx"} 1
```

> Pour `annuaire`, mes buckets sont `0.05, 0.1, 0.2, 0.3, 0.5, 1, 2` parce que mon SLO p95 est de 300 ms (0.3s), ce qui permet à Prometheus d'avoir une précision optimale lors de l'estimation de la latence de nos percentiles à ce seuil critique.

---

## Étape 3 — Installer kube-prometheus-stack via ArgoCD

### 1. Fichiers et configurations de déploiement
* **Manifeste de l'application parent** : [kube-prometheus-stack.yaml](file:///c:/Users/carro/Downloads/tp-argocd/tp-argocd/devhub-campus/pulse-campus/platform-sre/apps/observability/kube-prometheus-stack.yaml)
* **Values du chart** : [kube-prometheus-stack-values.yaml](file:///c:/Users/carro/Downloads/tp-argocd/tp-argocd/devhub-campus/pulse-campus/platform-sre/values/kube-prometheus-stack-values.yaml)

### 2. Capture de l'UI ArgoCD montrant l'application Synced + Healthy
*[ACTION REQUISE : Prenez une capture d'écran de votre interface ArgoCD pour l'application "kube-prometheus-stack" montrant tous les composants au statut Synced & Healthy, puis insérez-la ci-dessous.]*

*(Note : Tous les pods de la stack (Grafana, Operator, NodeExporter, Prometheus, StateMetrics) sont bien `Running` et opérationnels dans le namespace `monitoring` du cluster).*

---

## Étape 4 — Brancher votre service : ServiceMonitor + premier dashboard

### 1. Fichiers de configuration
* **Lien vers le template ServiceMonitor du service** : `templates/servicemonitor.yaml`
* **Lien vers le JSON du dashboard Grafana** : `platform-sre/dashboards/[service].json`

### 2. Les 4 requêtes PromQL utilisées dans le dashboard
1. **Request rate (RPS)** :  
   `[Requête PromQL]`
2. **Error rate** :  
   `[Requête PromQL]`
3. **Latency p50 / p95 / p99** :  
   `[Requête PromQL]`
4. **Build info** :  
   `[Requête PromQL]`

---

## Étape 5 — Du Deployment au Rollout

### 1. Fichiers de configuration
* **Lien vers l'Application d'installation d'Argo Rollouts** : `platform-sre/apps/argo-rollouts.yaml`
* **Lien vers le chart mis à jour (Rollout + Services)** : `templates/rollout.yaml`

### 2. Capture du Canary en cours
*[Insérer la capture d'écran de la commande `kubectl argo rollouts get rollout` montrant le split de trafic (ex: 20% / 80%).]*

---

## Étape 6 — Canary manuel : pause, promote, abort

### 1. Pilotage manuel : Les 3 scénarios documentés

#### Scénario 1 : Promotion normale
* **Commande exacte** : `kubectl argo rollouts promote [nom]`
* *Observations et captures* : [Détails]

#### Scénario 2 : Annulation explicite (Abort)
* **Commande exacte** : `kubectl argo rollouts abort [nom]`
* *Observations et captures* : [Détails]

#### Scénario 3 : Promotion forcée (Full)
* **Commande exacte** : `kubectl argo rollouts promote [nom] --full`
* *Observations et captures* : [Détails]

### 2. Réponse argumentée : Promote `--full` en production
> « [Insérer votre réponse argumentée sur le danger et la justification du promote --full en production] »

---

## Étape 7 — AnalysisTemplate : La promotion sur preuve

### 1. Fichiers de configuration
* **Lien vers l'AnalysisTemplate** : `templates/analysistemplate.yaml`

### 2. Captures d'écrans des AnalysisRuns
* **AnalysisRun réussi** (Canary OK, promotion automatique) : *[Insérer la capture]*
* **AnalysisRun échoué** (Canary KO, rollback automatique) : *[Insérer la capture]*

### 3. Discussion sur le choix des seuils et la durée d'analyse
*[Remplir votre discussion sur le calibrage des seuils (taux d'erreur < 1%, latence) et la durée (5 minutes) pour obtenir une mesure statistiquement viable.]*

---

## Étape 8 — Blue/Green : Autre stratégie, autre arbitrage

### 1. Fichiers de configuration
* **Lien vers le chart Helm du service configuré en Blue/Green** : `templates/rollout.yaml`

### 2. Tableau comparatif : Canary vs. Blue/Green
*[Insérer votre tableau comparatif présentant les avantages, inconvénients et cas d'usages respectifs.]*

### 3. Capture de la bascule manuelle réussie
*[Insérer la capture de l'UI Argo Rollouts ou de la console lors de la bascule.]*

---

## Étape 9 — Routage avancé : Header-based pour les tests internes

### 1. Fichiers de configuration
* **Lien vers le chart ou ingress mis à jour** : `templates/ingress.yaml`

### 2. Démonstration curl des deux comportements
```text
$ curl [service].devhub.local
[Réponse attendue : version stable]

$ curl -H "X-Beta-User: true" [service].devhub.local
[Réponse attendue : version canary]
```

---

## Étape 10 — Alerting Alertmanager et notifications Rollouts

### 1. Fichiers et configurations de la chaîne d'alerte
* **Lien vers les PrometheusRules** : `templates/prometheusrule.yaml`
* **Configuration d'Alertmanager** : `platform-sre/alertmanager-config.yaml`
* **Configuration des notifications d'Argo Rollouts** : `platform-sre/rollouts-notifications.yaml`

### 2. Captures d'écrans des Webhooks de notifications
* **Webhook primaire (Page / Alerte critique)** : *[Insérer la capture]*
* **Webhook secondaire (Ticket / Alerte modérée)** : *[Insérer la capture]*
* **Webhook de notifications de déploiement** : *[Insérer la capture]*

---

## Étape 11 — Comparer Argo Rollouts, Flagger et la Rolling-Update native

### Matrice d'évaluation complétée

| Critère | RollingUpdate natif | Argo Rollouts | Flagger | Justification technique |
| :--- | :---: | :---: | :---: | :--- |
| **Courbe d'apprentissage** | /5 | /5 | /5 | [Justification] |
| **Intégration avec ArgoCD (GitOps)** | /5 | /5 | /5 | [Justification] |
| **Intégration avec Flux CD (GitOps)** | /5 | /5 | /5 | [Justification] |
| **Variété des stratégies** | /5 | /5 | /5 | [Justification] |
| **Variété des metric providers** | /5 | /5 | /5 | [Justification] |
| **UI / Dashboard prêt à l'emploi** | /5 | /5 | /5 | [Justification] |
| **Coût opérationnel dans le cluster** | /5 | /5 | /5 | [Justification] |
| **Adapté à un mesh (Linkerd/Istio)** | /5 | /5 | /5 | [Justification] |
| **Communauté / Releases** | /5 | /5 | /5 | [Justification] |
| **Risque si le contrôleur tombe** | /5 | /5 | /5 | [Justification] |

---

## Étape 12 — Synthèse obligatoire : « Ma chaîne de release est-elle production-ready ? »

### Livrable 1 — Rétrospective : Le même geste, trois paradigmes
*[Insérer vos commentaires pour chaque ligne du tableau de paradigmes.]*

### Livrable 2 — Ce que cette chaîne ne sait toujours pas faire
*[Analyse des 7 thèmes d'angles morts (Traçabilité distribuée, Loki, RUM, Chaos, Kyverno, Cosign, Velero) en 5 à 10 lignes par thème.]*

### Livrable 3 — Votre position d'architecte
*[Votre plaidoyer technique de 10 lignes maximum présentant les briques que vous conservez, remplacez ou ajoutez dans une architecture industrielle.]*
