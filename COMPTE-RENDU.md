# Compte rendu — TP Pipeline DevSecOps (Master TRC)
Étapes validées : 10/10

## Étape 1 — Préparer son poste
- [x] node -v affiche v22.x
- [x] docker run hello-world fonctionne sans sudo
- [x] Les 7 images sont téléchargées (docker images)
- [x] La base Trivy est téléchargée

## Étape 2 — Mise en place
- [x] npm test affiche 3 tests réussis
- [x] package-lock.json est commité
- [x] Le dépôt tp-notes est visible sur github.com

**Q.** Pourquoi faut-il commiter package-lock.json, et pourquoi utilisera-t-on npm ci plutôt que npm install dans le pipeline ?
il faut commiter package-lock.son Parce qu'il sert à figer les versions exactes de toutes les dépendances. Ça évite que le code fonctionne chez moi mais plante sur une autre machine à cause d'une mise à jour automatique d'un paquet. On utilise npm ci (clean Install) pultot que npm install dans pipeline car Parce que npm ci est fait exprès pour l'intégration continue (CI) : il nettoie tout avant d'installer, il lit uniquement le package-lock.json sans le modifier, et il est beaucoup plus rapide et fiable pour un serveur de build.

## Étape 3 — CI de base
- Durée du job build-test (secondes) : 11 secondes
- [x] Le pipeline passe au vert sur main
- [x] La pull request test-rouge est bloquée (pipeline rouge)
- [x] La pull request est fermée sans merge, branche supprimée

**Q.** Quelle est la différence entre ce que vous venez de faire (Continuous Integration) et le Continuous Delivery ? Que manque-t-il au pipeline pour en faire du Continuous Delivery ?
dans cette étape qu'on a fait CI(Continuous Integration) consiste à compiler le code et à lancer les tests automatiquement à chaque modification pour s'assurer que tout fonctionne et que les développeurs ne cassent rien.Et concernant CD(Continuous Delivery) c'est pour aller plus loin. En plus des tests, le code est automatiquement préparé et déployé sur un environnement de staging (pré-production) pour être validé. et pour Que manque au pipeline pour en faire  CD ,Il manque une étape (un job) de déploiement automatique vers un serveur ou un registre pour que l'application puisse être mise en production toute seule.

## Étape 4 — Conteneur
- Taille de l'image naïve (Mo) : 412 Mo
- Trivy : HIGH + CRITICAL (naïve) : 11
- Checkov : contrôles échoués (naïve) : 2
- Utilisateur affiché par whoami (naïve) : root

**Q.** Repérez trois défauts de la version naïve. Pour chacun, dites quel risque il crée.
le 1er défauts est Utiliser une image de base trop lourde (notre cas node:22) le risque est Elle embarque plein d'outils inutiles, ce qui augmente la taille de l'image et rajoute un max de failles de sécurité potentielles.
la 2émé défauts est Tourner en tant qu'administrateur (root) le risque Si l'application été piratée, l'attaquant récupère les pleins pouvoirs directement à l'intérieur du conteneur.
la 3émé défaut Installer tous les paquets avec npm install le risque Ça garde les outils de développement qui ne servent à rien en production, ce qui alourdit l'image pour rien et augmente les risques de vulnérabilités.
- Taille de l'image durcie (Mo) : 61.4 Mo
- Trivy : HIGH + CRITICAL (durcie) : 11
- Checkov : contrôles échoués (durcie) : 0
- Utilisateur affiché par whoami (durcie) : node
- [x] L'image durcie démarre et /health répond
- [x] L'état de santé Docker passe à healthy
- [x] Dockerfile et .dockerignore durcis sont commités

**Q.** Une image avec 0 vulnérabilité HIGH/CRITICAL aujourd'hui le sera-t-elle encore dans un mois ? Que faut-il prévoir ?
Non, une image  avec 0 vulnérabilité aujourd'hui ne le sera plus dans un mois car De nouvelles vulnérabilités  sont découvertes tous les jours dans les logiciels et les paquets déjà installés.
donc il faut prévoir  Mettre en place des scans automatiques réguliers et penser à mettre à jour régulièrement l'image de base et les dépendances pour reconstruire l'application.

## Étape 5 — Dépendances & SBOM
- Dépendances directes : express@4.22.3
- Paquets dans l'arbre complet : 128
- npm audit : vulnérabilités trouvées : 0 vulnerabilities
- Composants dans sbom-src : 69
- Composants dans sbom-image : 285
- [x] Les deux SBOM sont générés (et ignorés par Git)
- [x] dependabot.yml est commité et les alertes sont activées

**Q.** Vous n'avez déclaré qu'une dépendance. Pourquoi l'arbre en contient-il autant ? Et pourquoi le SBOM de l'image est-il plus gros que celui du code source ?
1er question l'arbre contient autant de dépendances alors qu'on en a déclaré qu'une car chaque bibliothèque que vous installez (comme express) a elle-même besoin d'autres bibliothèques pour fonctionner (ce qu'on appelle des dépendances transitives).
2éme question  le SBOM de l'image est plus gros que celui du code source  car il liste le code Node.js plus tous les paquets du système d'exploitation de base (comme Alpine avec musl ou busybox) nécessaires pour faire tourner le conteneur.

## Étape 6 — Secrets
- Gitleaks : fuites trouvées sur le dépôt : Pas de fuites trouvées (no leaks found)
- Gitleaks : fuites trouvées avec demo.env : 1 fuite trouvée (leaks found: 1)
- Nom de la règle déclenchée : generic-api-key
- [x] Le commit contenant demo.env a été refusé par le hook
- [x] demo.env est supprimé, rien n'a été poussé

**Q.** Le hook pre-commit est sur votre poste. Pourquoi faut-il quand même un scan de secrets dans le pipeline CI ? Et si un vrai secret avait été poussé, quelle serait la première action ?
1er question il faut quand même un scan de secrets dans le pipeline CI car le hook pre-commit s'exécute uniquement en local sur votre poste et peut être contourné, La CI sert de deuxième filet de sécurité indispensable sur GitHub.
2émé questions  la première action serait Il faut tout de suite supprimer le secret compromis sur la plateforme concernée pour couper l'accès aux pirates, avant même de s'occuper de nettoyer l'historique Git.

## Étape 7 — SAST
- Nombre de règles exécutées : 75
- Alertes (findings) : 0
- Alertes avec le Dockerfile naïf : 1 alerte

**Q.** Semgrep ne trouve rien (ou presque) sur votre code. Cela prouve-t-il que l'application est sûre ? Citez deux limites du SAST.
la 1er question : Non, cela ne prouve pas que l'application est sûre car un résultat vide signifie simplement qu'aucune faille correspondant aux règles intégrés dans l'outil n'a été détectée, mais cela ne garantit pas l'absence totale de bogues ou de vulnérabilités plus complexes.
la 2éme question:deux limites du SAST.
1er limite : les faux positifs et faux négatifs c-à-d L'analyse statique peut générer des faux positifs (fausses alertes signalées à tort) ou passer à côté de certaines vulnérabilités contextuelles (faux négatifs) qu'elle ne sait pas interpréter.
2éme limite: pas de vision à l'exécution c-à-d l'outil analyse uniquement le code source de manière statique sans l'exécuter , il ne peut donc pas détecter les failles liées à la configuration du serveur en production, aux environnements d'exécution, ou aux problèmes d'authentification dynamiques.

## Étape 8 — Pipeline & portes
- Durée totale du pipeline (minutes) : 58 secondes
- Job le plus long :  job sast avec 28 secondes
- Porte 1 : jobs en échec : le job image
- Porte 2 : jobs en échec : le job iac
- [x] Le pipeline complet est vert sur main
- [x] Les deux pull requests de test sont bloquées
- [x] Les branches de test sont fermées sans merge
- [x] Le SBOM est téléchargeable dans les artefacts

**Q.** Le pipeline utilise des images aquasec/trivy:latest, semgrep/semgrep, etc. sans version fixée. Quel risque cela crée-t-il pour la chaîne d'approvisionnement, et comment le réduire ?
le risque cela crée pour la chaîne d'approvisionnement estL'instabilité et les ruptures de compatibilité (Breaking Changes) c-à-d une mise à jour automatique et imprévue de l'outil peut modifier la sortie des scans, supprimer des options ou introduire des bugs, ce qui risque de casser brutalement vos builds et vos portes de sécurité sans modification de votre part.
En peut le réduire avec Mettre à jour de manière contrôlée c-à-d gérer les montées de version via des outils automatisés (comme Dependabot) pour les tester de façon sécurisée avant de les valider.

**Q.** Le job Trivy utilise --ignore-unfixed et ne bloque que HIGH et CRITICAL. Justifiez ces deux choix, et dites ce qu'ils laissent passer.
1er choix --ignore-unfixed: Cette option permet d'ignorer les vulnérabilités pour lesquelles aucun correctif (patch) ou mise à jour n'est encore disponible de la part des éditeurs de logiciels ou des mainteneurs de l'OS.
-->Cela laisse passer des vulnérabilités connues (CVE) mais non corrigées. L'objectif est d'éviter de bloquer inutilement les développeurs sur des alertes qu'ils ne peuvent pas corriger techniquement, en se concentrant uniquement sur les failles qui disposent d'un correctif applicable.
2éme choix ne bloque que HIGH et CRITICAL Face au volume très élevé d'alertes générées par les scanners de vulnérabilités, se focaliser sur les niveaux HIGH et CRITICAL permet d'établir un seuil de tolérance pragmatique (niveau de risque inacceptable) pour ne pas engorger le pipeline avec des failles de moindre importance (LOW ou MEDIUM).
-->Cela laisse passer toutes les vulnérabilités de niveaux LOW, MEDIUM, ainsi que les alertes de type UNKNOWN. Bien que moins critiques individuellement, un nombre accumulé de failles moyennes pourrait théoriquement être exploité dans le cadre d'une attaque combinée.

## Étape 9 — Tri d'alertes
- Alerte 1 : Vrai positif · Bloquer la livraison maintenant (juste)
- Alerte 2 : Faux positif · Accepter et documenter l'exception (juste)
- Alerte 3 : Vrai positif · Backlog / surveiller (juste)
- Alerte 4 : Faux positif · Accepter et documenter l'exception (juste)
- Alerte 5 : Vrai positif · Bloquer la livraison maintenant (juste)
- Alerte 6 : Vrai positif · Corriger sous quelques jours (juste)
- Alerte 7 : Vrai positif · Backlog / surveiller (juste)
Score : 7/7

**Q.** Pour l'alerte que vous avez trouvée la plus difficile à classer, expliquez votre raisonnement.
pour moi l'alerte que j' ai trouvée la plus difficile à classer est Alerte 7 ( Trivy · image
CVE haute (CVSS 7.5) dans busybox, aucun correctif publié. L'API n'appelle jamais de commande système.) car au début, je me suis dit : « C'est un vrai positif, donc on applique la règle et on bloque. » Mais j'ai compris qu'il ne faut pas faire du zèle bête : si on ne peut pas corriger la faille et que l'appli ne s'en sert même pas, bloquer le projet ne sert à rien à part pénaliser les devs. Il faut savoir faire la part des choses entre la théorie et la réalité du terrain.

## Bilan

**Q.** Placez chaque contrôle que vous avez mis en place sur la boucle Plan › Code › Build › Test › Release › Deploy › Operate › Monitor. Quelles étapes de la boucle restent sans contrôle dans votre pipeline ?
pour 1er question ona Code : Semgrep (SAST), Gitleaks (détection de secrets), Dependabot (gestion des dépendances),Build : Checkov (IaC / Dockerfile), Trivy (analyse de l'image de base),Test : Tests unitaires / npm audit / analyse de dépendances, et en fin Release : Portes de validation (image, iac) sur la Pull Request avant fusion sur main
les étapes de la boucle restent sans contrôle dans votre pipeline sont Plan, Deploy  et Operate / Monitor Il n'y a aucun contrôle automatisé au tout début (Plan), ni au moment du déploiement (Deploy), et surtout rien à la fin pour surveiller l'application quand elle tourne en production (Operate / Monitor).

**Q.** Si vous deviez ajouter un seul contrôle de plus à ce pipeline, lequel choisiriez-vous et pourquoi ?
Ajouter un contrôle de sécurité dynamique (DAST / Dynamic Application Security Testing) car Pour l'instant, tous nos tests vérifient le code et l'image "sur le papier" avant qu'ils ne sortent. Un outil dynamique permettrait de tester l'application en temps réel une fois qu'elle est lancée pour voir si des attaques passent à travers.

