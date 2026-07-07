# SAA-C03 — Fiche J2 : EC2, Auto Scaling, ECS/EKS, Lambda

*Basée sur le Tutorials Dojo Study Guide (pages 56-119) — 06/07/2026*

---

## 1. Amazon EC2

### Composants d'une instance
Ordre de création : **AMI** (OS + config, non modifiable après lancement) → **type/taille** (modifiable après coup = "right sizing") → réseau/VPC/subnet → placement group → stockage EBS → tags → security groups → **key pair** (non ré-associable après lancement).

### Types d'instances
- **General Purpose** : équilibre compute/mémoire/réseau
- **Compute Optimized** : calcul intensif
- **Memory Optimized** : bases de données in-memory, gros datasets
- **Storage Optimized** : accès séquentiel haut débit sur gros volumes locaux, dizaines de milliers d'IOPS
- **Nitro-based** : bare metal, jusqu'à **64 000 IOPS** en EBS Provisioned IOPS (vs 32 000 sur les autres)

### Stockage le plus performant
- **EBS Provisioned IOPS (io1/io2)** : le plus haut niveau de perf avec persistance des données
- **Instance store** : latence encore plus basse (attaché physiquement à l'hôte), mais données perdues à l'arrêt — uniquement pour données temporaires/reconstructibles

### Options d'achat
| Option | Caractéristique clé |
|---|---|
| On-Demand | À l'heure/seconde, zéro engagement, instances arrêtées non facturées |
| Compute Savings Plans | Engagement en $/h sur 1-3 ans, **flexible** : toute famille, taille, AZ, région, OS, tenancy + **Fargate et Lambda** |
| EC2 Instance Savings Plans | Moins cher mais limité à une famille d'instance dans une région |
| Reserved Instances | Engagement sur type+région+tenancy+OS ; **revendables sur la Marketplace** (Standard) |
| Spot | Le moins cher ; interruption avec **préavis de 2 minutes** quand le prix spot dépasse ton max |
| Dedicated Hosts | Facturation par hôte physique, visibilité sockets/cores, **BYOL supporté** |
| Dedicated Instances | Facturation par instance, matériel single-tenant, pas de visibilité hardware, pas de BYOL |
| Capacity Reservations | Réserve de capacité dans une AZ, sans engagement de durée |

### Health checks — les 3 types
| | EC2 status check | ELB health check | Auto Scaling health check |
|---|---|---|---|
| Quoi | Hardware/software de l'instance | Ping/connexion/requête HTTP vers les cibles | Combine les sources |
| Désactivable | Non (natif) | Configurable (port, protocole, path) | Suspendable par process |
| Particularité | System status (AWS répare) vs Instance status (toi) | HTTP 200 attendu ; NLB = checks actifs + passifs | **Si un LB est attaché : EC2 status + ELB health check combinés** |

### Placement Groups
- **Cluster** : instances rapprochées dans **une seule AZ** → latence minimale, débit max (HPC). Incompatible multi-AZ.
- **Partition** : jusqu'à 7 partitions/AZ, pas de hardware partagé entre partitions → limite les pannes corrélées (big data distribué)
- **Spread** : chaque instance sur un rack distinct (réseau + alimentation séparés), max **7 instances/AZ/groupe** → isolation maximale (petites charges critiques)

### Security Groups vs NACL — LE tableau à connaître par cœur
| | Security Group | Network ACL |
|---|---|---|
| Niveau | Instance (précisément : l'**ENI**) | Subnet |
| État | **Stateful** — la réponse à un trafic autorisé passe automatiquement | **Stateless** — règles explicites nécessaires dans les deux sens |
| Règles | **Allow uniquement** | **Allow ET Deny** |
| Évaluation | Toutes les règles évaluées ensemble | Par numéro croissant, première règle qui matche s'applique |
| Défaut | SG par défaut du VPC si rien de spécifié | NACL par défaut du VPC = tout autorisé ; NACL custom = tout refusé au départ |
| Association | Plusieurs SG par ENI | **1 seule NACL par subnet** (mais 1 NACL peut couvrir plusieurs subnets) |

Points fins :
- Communication entre 2 instances du même VPC → règle SG avec **IP privée** (ou ID du SG), jamais l'IP publique/Elastic
- Un SG d'un VPC peeré peut être référencé dans une règle
- Ports éphémères : ne pas oublier la règle allow (NAT gateway utilise 1024-65535) dans les NACL des subnets publics

---

## 2. EC2 Auto Scaling

### Composants
- **ASG** = launch template (AMI, type, key pair, SG, IAM instance profile, user data) + service de scaling
- Tailles : **min / desired / max**
- Multi-AZ recommandé ; si LB attaché → **utiliser son health check**
- Un ASG est **régional** : multi-AZ oui, multi-région non. Les launch templates aussi.

### Scaling horizontal vs vertical
- **Horizontal** : ajouter des instances (nécessite ASG + ELB) → serveurs stateless
- **Vertical** : grossir l'instance → souvent nécessite un arrêt (EC2, RDS)

### Les 3 policies dynamiques
| Policy | Fonctionnement | Limite |
|---|---|---|
| **Simple Scaling** | Une action par alarme CloudWatch | Doit attendre cooldown + health checks avant de réagir à une nouvelle alarme |
| **Step Scaling** | Paliers proportionnels à l'ampleur du dépassement (+10% si 60-70% CPU, +30% au-delà...) | Continue à réagir aux alarmes **même pendant un scaling en cours** |
| **Target Tracking** | Tu fixes une cible (ex : CPU moyen 80%), AWS gère les seuils tout seul | Le plus simple à configurer |

Métriques Target Tracking notables : ASGAverageCPUUtilization, ASGAverageNetworkIn/Out, **ALBRequestCountPerTarget**.

À connaître aussi : **instance warmup** (délai avant que la métrique d'une nouvelle instance compte dans l'agrégat) et **scheduled/predictive scaling** pour les charges prévisibles.

### Lifecycle Hooks
- **Pending:Wait** (au lancement) / **Terminating:Wait** (à la terminaison)
- Heartbeat timeout : 30 s à 7200 s (défaut 3600 s), fin anticipée via `CompleteLifecycleAction`
- Default Result au timeout :
  - **CONTINUE** = l'ASG suppose le succès et poursuit le cycle normal (InService ou terminaison)
  - **ABANDON** = terminaison immédiate de l'instance
- Événements routés vers **EventBridge** (→ Lambda) ou notification **SNS** (nécessite un topic + rôle IAM)

### Process suspendables (troubleshooting)
Launch, Terminate, AddToLoadBalancer, AlarmNotification, AZRebalance, HealthCheck, ReplaceUnhealthy, ScheduledActions.

---

## 3. ECS / EKS / ECR

### ECS — Task Definition
Spec sheet du conteneur : image Docker, CPU/RAM, launch type (**EC2** ou **Fargate**), network mode, volumes, commande de démarrage.

### Les 3 rôles IAM d'ECS (souvent confondus à l'examen)
| Rôle | Sert à |
|---|---|
| **Container Instance Role** | L'instance EC2 hôte communique avec le service ECS (launch type EC2 uniquement) |
| **Task Execution Role** | Tirer l'image depuis ECR + publier les logs CloudWatch (infra) |
| **Task Role** | Les conteneurs appellent eux-mêmes des API AWS (applicatif, optionnel) |

### Les 4 network modes
| Mode | Caractéristique |
|---|---|
| **Bridge** (défaut Linux) | Réseau virtuel Docker, port mapping dynamique, perf moyenne |
| **Host** | Partage l'IP et la pile réseau de l'hôte, plus rapide, mais 1 seul conteneur par port |
| **awsvpc** | Une ENI + IP propre par tâche — **seul mode supporté par Fargate** |
| **None** | Pas de réseau (loopback uniquement) |

### Task Placement Strategies (launch type EC2)
- **Binpack** : remplit au max les instances existantes (moins d'instances = moins cher)
- **Spread** : répartit sur instances/AZ (haute dispo) — défaut pour les services
- **Random** : sans critère
- Combinables entre elles ; Fargate = spread multi-AZ par défaut

### EKS / ECR
- **EKS** : Kubernetes managé → choix **cloud-agnostic** (portable on-prem/Azure/GCP), pods sur Fargate ou EC2
- **ECR** : registre d'images managé, chiffrement au repos (S3 SSE + option KMS), cache de registres publics

---

## 4. AWS Lambda

### Concurrency
- Quota par défaut : **1000 exécutions concurrentes par région**, partagé entre toutes les fonctions du compte
- **Reserved concurrency** : capacité dédiée à une fonction — les autres ne peuvent pas la consommer, et elle ne peut pas dépasser sa réserve. Gratuit.
- **Provisioned concurrency** : pré-initialise des environnements → élimine les cold starts. Payant. Ne peut pas dépasser la reserved concurrency si les deux sont configurées.

### Lambda@Edge
- Code Lambda (Node.js/Python) exécuté dans les **edge locations CloudFront**
- 4 déclencheurs : viewer request, origin request, origin response, viewer response
- Différence clé vs Lambda+API Gateway : ces derniers sont **régionaux**, Lambda@Edge est global/edge

### Lambda dans un VPC
- Par défaut, **pas d'accès aux ressources privées d'un VPC**
- Connexion au VPC = Lambda crée une **ENI par subnet** configuré (partagées entre fonctions du même subnet)
- Rôle d'exécution requis : permissions type `AWSLambdaVPCAccessExecutionRole`
- **Effet secondaire majeur** : la fonction perd l'accès internet public, sauf NAT/IGW dans le VPC ou VPC endpoints pour les services AWS

---

## 5. Questions qui tombent souvent (patterns d'examen)

1. **"Latence réseau minimale entre instances"** → Cluster placement group (et rappel : une seule AZ)
2. **"Réduire les pannes corrélées, chaque instance isolée"** → Spread placement group (max 7/AZ)
3. **"Le moins cher pour un workload interruptible"** → Spot
4. **"Discount flexible toutes familles + Fargate/Lambda"** → Compute Savings Plans
5. **"BYOL / licences par socket"** → Dedicated Hosts (pas Dedicated Instances)
6. **"Bloquer des IP précises sur un subnet"** → NACL (règles deny), jamais un SG
7. **"Trafic de réponse bloqué alors que l'aller passe"** → penser NACL stateless (règle retour manquante sur les ports éphémères)
8. **"ASG remplace des instances que l'EC2 status check dit saines"** → normal si le health check ELB échoue (checks combinés)
9. **"Actions custom avant mise en service / avant terminaison"** → Lifecycle hooks (Pending:Wait / Terminating:Wait)
10. **"Scaling proportionnel à l'ampleur du pic"** → Step Scaling ; **"maintenir une métrique à une valeur"** → Target Tracking
11. **"Conteneurs Fargate"** → network mode awsvpc obligatoire
12. **"Minimiser le nombre d'instances ECS pour réduire les coûts"** → Binpack
13. **"Le conteneur doit appeler S3/DynamoDB"** → Task Role (pas Execution Role)
14. **"Lambda doit joindre une base RDS privée"** → attacher au VPC + anticiper la perte d'accès internet (NAT ou VPC endpoints)
15. **"Éliminer les cold starts"** → Provisioned concurrency (SnapStart si Java)

---

## 6. Mémo flash (à relire avant l'examen)

- SG = stateful, allow only, niveau ENI | NACL = stateless, allow+deny, niveau subnet, ordre numérique
- Entre 2 instances d'un même VPC : **IP privée** dans la règle SG
- Spot = préavis **2 min** | RI Standard = revendable Marketplace | Compute SP = couvre Fargate/Lambda
- Cluster PG = 1 AZ, perf | Spread PG = 7 instances/AZ, isolation | Partition PG = 7 partitions/AZ, big data
- ASG = régional (multi-AZ oui, multi-région non)
- ASG + ELB attaché = health checks **combinés** (EC2 status + ELB)
- Lifecycle hook : CONTINUE = poursuite normale | ABANDON = terminaison immédiate | timeout défaut 3600 s
- Simple < Step (paliers, réactif pendant scaling) < Target Tracking (le plus simple)
- ECS : Execution Role = pull image/logs | Task Role = API calls du conteneur
- Fargate = awsvpc only | Binpack = coût | Spread = haute dispo
- Lambda : 1000 concurrent/région | Reserved = quota dédié gratuit | Provisioned = anti-cold-start payant
- Lambda en VPC = perd internet public (NAT/IGW ou VPC endpoints pour compenser)
- Lambda@Edge = viewer/origin × request/response, global vs API Gateway régional

---

## 7. Corrections détaillées des QCM du jour

### QCM 1 — EC2 (7 questions)

**Q1 — HPC, latence minimale intra-AZ → C) Cluster** ✅
Cluster rapproche physiquement les instances dans une AZ. Spread/Partition font l'inverse (isolation). *Piège associé : cluster PG incompatible avec un ASG multi-AZ.*

**Q2 — Discount flexible toutes familles/régions/OS → B) Compute Savings Plans** ✅
Seule option dont la remise s'applique quelle que soit la famille, taille, AZ, région, OS, tenancy — et couvre aussi Fargate et Lambda. EC2 Instance SP = famille+région fixes. RI = config encore plus figée.

**Q3 — Autoriser HTTP sauf 3 IP malveillantes → B) Network ACL** ✅ *(compté juste, erreur de frappe)*
Il faut des règles **deny** explicites → NACL. Un SG ne sait faire que du allow. IAM ne gère pas le trafic réseau. Astuce : mettre les deny sur des numéros de règle bas (évalués en premier).

**Q4 — Communication entre 2 instances du même VPC → C) IP privée** ❌ (répondu A puis A de nouveau)
**Règle absolue** : dans une règle SG entre ressources d'un même VPC, on référence l'IP privée ou l'ID du security group. L'IP publique/Elastic ferait sortir le trafic du VPC (voire échouerait selon le routage). Raté 2 fois → prioritaire dans la fiche mémo.

**Q5 — ASG + ELB, instance KO au check ELB mais OK au status EC2 → B) remplacement programmé** ✅ *(raté au 1er passage, corrigé au 2e)*
Avec un LB attaché, l'ASG combine les deux checks : un échec ELB suffit à marquer l'instance unhealthy → remplacement. L'option "seul le LB arrête de router" décrit le comportement **sans** intégration ASG.

**Q6 — Prix spot dépasse le max → B) préavis de 2 minutes** ✅
L'instance est stoppée ou terminée après un avertissement de 2 min. Elle ne bascule jamais automatiquement en On-Demand.

**Q7 — Chaque instance sur un rack distinct → B) Spread** ✅
Spread = un rack (réseau + alimentation propres) par instance, max 7 par AZ par groupe. Partition isole des **groupes** d'instances, pas chaque instance individuellement.

### QCM 2 — Auto Scaling (5 questions)

**Q1 — Scaling proportionnel + réactif pendant une action en cours → B) Step Scaling** ✅
Les "step adjustments" varient la réponse selon l'ampleur du dépassement, et la policy continue de traiter les alarmes pendant un scaling (contrairement à Simple qui attend le cooldown).

**Q2 — Maintenir 80% CPU sans définir de seuils → C) Target Tracking** ✅
On fixe la cible, AWS crée et gère les alarmes CloudWatch tout seul.

**Q3 — Retarder l'InService le temps d'un script d'init → B) Lifecycle Hook** ✅
Pending:Wait retient l'instance jusqu'à `CompleteLifecycleAction` ou timeout. Le grace period ne fait que différer les *health checks* ; le warmup ne concerne que l'agrégation des *métriques* ; le cooldown espace les *actions de scaling*.

**Q4 — Default Result ABANDON au lancement, timeout expiré → B) terminaison immédiate** ❌ (répondu C)
Le heartbeat timeout ne se relance jamais seul. ABANDON = échec présumé → instance terminée immédiatement. CONTINUE = succès présumé → poursuite du cycle. (Une relance du timeout est possible mais uniquement via l'API `RecordLifecycleActionHeartbeat`, pas automatiquement.)

**Q5 — ASG + LB : quel health check → B) celui du load balancer** ✅
Recommandation officielle : activer le health check ELB sur l'ASG pour que les instances défaillantes applicativement (pas juste hardware) soient remplacées.

### QCM 3 — ECS/Lambda (5 questions)

**Q1 — Maximiser l'utilisation des instances existantes → C) Binpack** ❌ (répondu B)
Binpack concentre les tâches sur le moins d'instances possible (critère : CPU ou RAM la plus faible restante) → économie. Spread fait l'inverse : il disperse pour la haute dispo. Mnémo : **Binpack = portefeuille, Spread = disponibilité**.

**Q2 — Network mode obligatoire sur Fargate → C) awsvpc** ✅
Pas d'hôte géré par toi sur Fargate → chaque tâche reçoit sa propre ENI/IP, seul awsvpc le permet.

**Q3 — Lambda attachée à un VPC, effet secondaire → B) perte d'internet public** ✅
Sauf NAT/IGW dans le VPC, ou VPC endpoints pour joindre les services AWS sans sortir sur internet.

**Q4 — Capacité dédiée sans viser les cold starts → B) Reserved concurrency** ✅
Réserve un quota exclusif (gratuit). Provisioned concurrency pré-chauffe les environnements (payant) — c'est elle qui cible les cold starts.

**Q5 — ASG multi-AZ + cluster placement group → B) impossible** ✅
Un cluster placement group ne peut pas s'étendre sur plusieurs AZ, par définition.

---

**Score global du jour : 13/17 (76%)** — au-dessus du seuil de passage dès le premier jour de contenu, avec 2 notions à retravailler : IP privée dans les règles SG (×2) et Binpack vs Spread.
