TP n°2 – Réplication & Tolérance aux pannes : MongoDB et Cassandra

Réalisé par : Wael NAJAR


TP2_replication

Partie 1 — Notions fondamentales
1. Définition d’un Replica Set dans MongoDB

Un Replica Set est un ensemble de serveurs MongoDB contenant chacun une copie identique des données.
Il sert à :

assurer la disponibilité du service,

éviter la perte de données,

offrir de la redondance en cas de panne.

Il est composé de :

un Primary, qui gère toutes les opérations d’écriture ;

un ou plusieurs Secondary, qui reproduisent les données du Primary.

2. Rôle du Primary

Le Primary est le serveur principal : il reçoit toutes les écritures et diffuse ensuite les modifications aux Secondary pour les maintenir synchronisés.

3. Rôle des Secondary

Un Secondary sert à :

répliquer les données du Primary,

assurer la continuité de service si le Primary devient indisponible (élection automatique).

4. Pourquoi les Secondary ne peuvent pas recevoir d’écritures ?

Interdire les écritures sur les Secondary évite :

les conflits d’opérations,

des incohérences dans l’ordre des écritures,

des divergences de données.

MongoDB simplifie ainsi la réplication, qui fonctionne en lecture seule côté Secondary.

5. Cohérence forte

Dans MongoDB, la cohérence forte signifie qu’une lecture effectuée sur le Primary renvoie toujours la version la plus récente d’une donnée.

6. Différence entre readPreference "primary" et "secondary"

primary : garantit des données toujours à jour.

secondary : décharge le Primary, mais peut renvoyer une donnée légèrement en retard.

7. Pourquoi lire sur un Secondary ?

On peut choisir cette option lorsque :

les données n’ont pas besoin d’être à jour à la milliseconde près,

la charge de lecture sur le Primary est élevée,

on effectue du reporting ou de l’analyse.

Partie 2 — Commandes & configuration

Après avoir mis en place 3 serveurs MongoDB et un client, Wael NAJAR a réalisé la configuration suivante.

8. Initialiser un Replica Set
rs.initiate()

9. Ajouter un nœud après initialisation
rs.add("mongo2:27017")
rs.add("mongo3:27017")

10. Afficher l’état du Replica Set
rs.status()

11. Identifier le rôle du nœud
rs.isMaster()


ou
rs.status() → champ stateStr.

12. Forcer la bascule (stepdown)
rs.stepDown()


ou avec délai :

rs.stepDown(60)

13. Ajouter un arbitre et son utilité
rs.addArb("hostname:port")


Un arbitre ne stocke pas de données : il vote uniquement pour les élections et permet de conserver une majorité.

14. Configurer un délai de réplication (slaveDelay)
cfg = rs.config()
cfg.members[1].slaveDelay = 60
cfg.members[1].priority = 0
rs.reconfig(cfg)

Partie 3 — Résilience et haute disponibilité
15. Que se passe-t-il si le Primary tombe sans majorité ?

Aucun Primary n'est élu.

Les écritures sont bloquées.

Seules les lectures sur Secondary restent possibles.

16. Comment MongoDB choisit un nouveau Primary ?

MongoDB lance une élection en tenant compte de :

la majorité des votes,

la priorité configurée,

l’état de synchronisation des Secondary.

17. Définition d’une élection

Processus automatique permettant de désigner un nouveau Primary lorsqu’il n’y en a plus.

18. Auto-dégradation

Un Replica Set se dégrade lorsque la majorité n’est plus atteignable : aucun Primary ne peut exister.

19. Pourquoi un nombre impair de nœuds ?

Pour garantir l’obtention d’une majorité lors des votes et éviter les blocages 50/50.

20. Effets d’une partition réseau

Le groupe majoritaire continue de fonctionner normalement.

Le groupe minoritaire perd son Primary ou reste Secondary.

Partie 4 — Cas pratiques
21. Panne du Primary dans un cluster 27017 / 27018 / 27019-Arbiter

Le Secondary (27018) et l’Arbitre forment la majorité → 27018 devient Primary.

22. Utilité d’un Secondary retardé (slaveDelay = 120s)

Un nœud retardé permet de récupérer des données supprimées ou modifiées par erreur, car il applique les opérations du Primary avec un décalage de 2 minutes.

23. Paramètres pour garantir une lecture toujours à jour

writeConcern : "majority"

readConcern : "majority"

24. Écriture confirmée par au moins deux nœuds
{ writeConcern: { w: 2 } }

25. Pourquoi un Secondary peut renvoyer une donnée obsolète ?

À cause du retard de réplication.
Pour éviter cela :
→ readPreference: "primary"

26. Vérifier le Primary
rs.status()


ou

rs.isMaster()

27. Forcer une bascule sans interruption forte
rs.stepDown(60)

28. Ajouter un nouveau Secondary

Démarrer un nouveau mongod avec le même replicaSet.

Depuis le Primary :

rs.add("nouveauNœud:27017")

29. Retirer un nœud défectueux
rs.remove("mongo2:27017")

30. Rendre un Secondary invisible (hidden)
cfg = rs.config()
cfg.members[i].hidden = true
cfg.members[i].priority = 0
rs.reconfig(cfg)


Utilisé pour analyses internes ou sauvegardes.

31. Modifier la priorité pour favoriser un Primary
cfg = rs.config()
cfg.members[1].priority = 2
rs.reconfig(cfg)

32. Vérifier le retard de réplication
rs.printSlaveReplicationInfo()


ou comparer les optimeDate avec
rs.status().

33. Rôle de rs.freeze()

Empêche un nœud de devenir Primary pendant un temps donné.

rs.freeze(60)

34. Redémarrer un Replica Set sans perdre la configuration

Redémarrer les nœuds sans supprimer le dossier dbPath → la configuration se recharge automatiquement.

36. Surveiller la réplication en temps réel

rs.status()

rs.printReplicationInfo()

rs.printSlaveReplicationInfo()

db.currentOp()

logs mongod

Questions complémentaires
37. Qu’est-ce qu’un Arbitre ?

Membre ne stockant pas de données, utilisé uniquement pour voter lors des élections.

38. Vérifier la latence de réplication

rs.printSlaveReplicationInfo()

comparaison des horodatages dans rs.status()

40. Voir le retard des Secondary
rs.printSlaveReplicationInfo()

41. Réplication asynchrone vs synchrone

MongoDB utilise une réplication asynchrone : le Primary n’attend pas la confirmation des Secondary.

41 (bis). Modifier la configuration sans redémarrage

Oui → via rs.reconfig().

42. Conséquences d’un Secondary très en retard

données obsolètes,

incapable de devenir Primary,

risque de resynchronisation complète.

43. Gestion des conflits

MongoDB évite les conflits car seul le Primary écrit : les Secondary rejouent l’oplog.

44. Peut-il y avoir plusieurs Primary ?

Non, grâce au système d’élection basé sur la majorité.

45. Pourquoi ne pas écrire sur un Secondary ?

Parce que cela casserait :

la cohérence,

l’ordre des opérations,

la réplication.

46. Réseau instable

Cause :

des élections répétées,

l’impossibilité d’écrire,

du retard de réplication.
