1. Map the current architecture (Cartographier l'architecture actuelle)
Tu as déjà le diagramme généré précédemment. Voici les réponses écrites attendues.

Where does isolation happen? (Où se produit l'isolation ?)

Niveau Conteneur : Isolation des processus et du système de fichiers via les namespaces et cgroups Linux. Ils partagent le noyau de l'hôte mais ne voient pas les autres processus.

Niveau Pod : Isolation réseau (chaque pod a sa propre IP) et contexte d'exécution.

Niveau Namespace Kubernetes : Isolation logique des ressources au sein du cluster (ex: dev vs prod).

What restarts automatically? (Qu'est-ce qui redémarre automatiquement ?)

Les Pods : Si le processus de l'application plante ou si la livenessProbe échoue, le Kubelet redémarre le conteneur.

Les Répliques manquantes : Si un pod est supprimé manuellement, le Deployment (via le ReplicaSet) en crée un nouveau pour respecter le nombre de répliques désiré.

What does Kubernetes not manage? (Qu'est-ce que Kubernetes ne gère pas ?)

Le matériel physique : Serveurs, routeurs, disques physiques.

L'OS des nœuds : Mises à jour du noyau ou de l'OS de la machine hôte.

La logique applicative : Bugs de code, schémas de base de données.

La persistance des données par défaut : Sans PersistentVolume, les données d'un pod sont perdues à sa suppression.

2. Compare containers and virtual machines (Comparer conteneurs et machines virtuelles)
Tableau de comparaison :

Caractéristique	Conteneurs	Machines Virtuelles (VM)
Partage du Noyau (Kernel)	Partagé avec l'OS hôte	Noyau complet et indépendant par VM
Temps de démarrage	Secondes (ou millisecondes)	Minutes (boot complet de l'OS)
Surcharge (Overhead)	Faible (MBs), processus natifs	Élevée (GBs), OS invité complet + hyperviseur
Isolation de sécurité	Isolation logicielle (namespaces) - Plus faible	Isolation matérielle (virtuelle) - Plus forte
Complexité opérationnelle	Portable, immuable, orchestré par API	Gestion de patchs OS, configuration lourde
Réponses :

When would you prefer a VM? Pour des applications nécessitant une isolation de sécurité stricte (multi-tenant hostile), des noyaux OS spécifiques, ou des applications monolithiques héritées (legacy) difficiles à conteneuriser.

When would you combine both? Presque toujours dans le cloud. On utilise des VMs pour créer les nœuds du cluster Kubernetes (pour l'isolation infrastructure), et on y fait tourner des conteneurs (pour l'agilité applicative).

3. Introduce horizontal scaling (Introduction à la mise à l'échelle horizontale)
What changes when you scale? (Qu'est-ce qui change ?)

Le nombre de pods (répliques) augmente.

Le Service distribue désormais le trafic entre plusieurs IPs de pods (Load Balancing).

La disponibilité de l'application augmente.

What does not change? (Qu'est-ce qui ne change pas ?)

L'adresse IP du Service (ClusterIP) reste la même.

La configuration du Deployment (image, env vars).

La base de données (c'est toujours la même instance unique partagée).

4. Simulate failure (Simuler une panne)
Who recreated the pod? (Qui a recréé le pod ?)

Le ReplicaSet (contrôlé par le Deployment).

Why? (Pourquoi ?)

Pour maintenir l'état désiré (Desired State). Le Deployment spécifie "3 répliques". Si une disparaît, la boucle de réconciliation de Kubernetes détecte l'écart (3 vs 2) et crée un nouveau pod.

What would happen if the node itself failed? (Et si le nœud échouait ?)

Après un délai (timeout), le Scheduler Kubernetes détecterait que le nœud est "NotReady" et reprogrammerait les pods manquants sur d'autres nœuds sains du cluster.

5. Introduce resource limits (Limites de ressources)
What are requests vs limits? (Différence Requests vs Limits)

Requests (Demandes) : Le minimum garanti. Le Scheduler l'utilise pour décider sur quel nœud placer le pod (si un nœud n'a pas assez de RAM libre pour la Request, le pod ne s'y lance pas).

Limits (Limites) : Le plafond absolu.

CPU : Si dépassé, le conteneur est ralenti (throttled).

RAM : Si dépassé, le conteneur est tué (OOM Killed).

Why important in multi-tenant systems?

Pour éviter le problème du "voisin bruyant" (Noisy Neighbor). Sans limites, une seule application buggée pourrait consommer 100% du CPU/RAM du nœud et faire planter toutes les autres applications hébergées dessus.

6. Add readiness and liveness probes (Sondes de disponibilité et de vie)
Difference between readiness and liveness :

Liveness Probe (Suis-je vivant ?) : Vérifie si l'app fonctionne.

Action si échec : Redémarrer le conteneur.

Cas d'usage : Deadlocks, boucles infinies.

Readiness Probe (Suis-je prêt ?) : Vérifie si l'app peut recevoir du trafic.

Action si échec : Couper le trafic (retirer l'IP du Service), mais laisser le pod vivant.

Cas d'usage : Démarrage lent (chargement de cache), surcharge temporaire.

Why does this matter in production?

Cela garantit le Zero-Downtime. Lors d'une mise à jour, Kubernetes n'envoie du trafic vers la nouvelle version que lorsqu'elle est "Ready". Si l'app plante, elle est redémarrée automatiquement (Self-healing).

7. Connect Kubernetes to virtualization (Lien avec la virtualisation)
What runs underneath your k3s cluster?

Probablement des Machines Virtuelles (si c'est dans le cloud ou via un outil comme Lima/WSL) ou directement le métal (si Bare Metal).

Is Kubernetes replacing virtualization?

Non, il l'abstrait. Kubernetes gère les applications, la virtualisation gère l'infrastructure. Ils sont complémentaires.

Cloud Stack Examples:

Cloud Data Center : Serveur Physique -> Hyperviseur -> VM -> OS Linux -> Kubernetes -> Conteneur.

Embedded Automotive : Hardware -> Hyperviseur temps réel -> OS minimal -> K3s -> Conteneurs critiques.

Financial : Serveur dédié -> VM avec isolation stricte -> Kubernetes dédié (pas de multi-tenant) -> Conteneurs.

8. Design a production architecture (Conception Production)
Pour ton fichier architecture-notes.md, propose cette structure :

Architecture Proposée :

High Availability (HA) : Cluster Kubernetes avec au moins 3 Worker Nodes répartis sur plusieurs zones de disponibilité (AZ).

Base de données : Sortie du cluster Kubernetes. Utilisation d'un service managé (ex: AWS RDS ou Google Cloud SQL) pour gérer automatiquement les backups, la réplication et le failover.

Ingress Controller : Nginx ou Traefik pour gérer l'entrée HTTPS et le routage.

Monitoring : Prometheus pour les métriques + Grafana pour la visualisation.

Logging : EFK Stack (Elasticsearch, Fluentd, Kibana) ou Loki pour centraliser les logs des conteneurs.

Répartition :

In Kubernetes : L'application Quote (stateless), les Jobs, le cache (Redis), l'Ingress Controller.

In VMs : Les nœuds Kubernetes eux-mêmes (Control Plane et Workers).

Outside Cluster : PostgreSQL (Managé), Registre d'images (Docker Hub/ECR), DNS, Load Balancer Cloud.

9. Required Extension: Secret-based configuration
Why is this better than plain-text?

Sécurité : Les mots de passe ne sont pas visibles dans le code source (git) ni dans la sortie standard de kubectl get deploy -o yaml.

Gestion : On peut mettre à jour les secrets sans redéployer l'image (juste redémarrer les pods).

Contrôle d'accès : On peut restreindre qui a le droit de voir les Secrets via le RBAC Kubernetes.

Is a Secret encrypted by default?

Non. Par défaut, les Secrets sont seulement encodés en Base64 (facilement décodables).

Où ? Ils sont stockés dans la base de données etcd de Kubernetes. Pour qu'ils soient chiffrés, l'administrateur du cluster doit activer Encryption at Rest au niveau de l'API Server.

10. Optional: Architecture critique (Critique rapide)
Pour finir ton exercice, ajoute cette critique honnête de ton design actuel :

Single Point of Failure (SPOF) : La base de données PostgreSQL est une instance unique. Si elle tombe, l'app ne peut plus lire/écrire de citations.

Persistance : Si le pod DB est supprimé, les données sont perdues (sauf si un PersistentVolumeClaim est configuré, ce qui n'est pas explicite dans le lab de base).

Sécurité : Bien que nous utilisions des Secrets, nous utilisons le compte root ou admin par défaut de la DB, ce qui n'est pas idéal (principe du moindre privilège).