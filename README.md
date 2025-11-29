# TP 1 - Redis : Découverte et Commandes Essentielles

**Par Wael NAJAR**

---

## Table des matières

1. **Première Partie : Découverte et Commandes Fondamentales**
   - Installation de Redis
   - Création d'une paire clé-valeur
   - Opérations CRUD dans Redis
   - Structures de données avancées
   - Pub/Sub : Communication en temps réel
   - Basculer entre les bases de données

2. **Deuxième Partie : Systèmes de Recommandation avec Redis**
   - Ensembles triés (ZSET)

3. **Troisième Partie : Manipulation de Données avec Hashes**
   - Utilisation des Hashes

---

## Introduction

Redis représente une solution NoSQL basée sur le modèle clé-valeur. Son principal atout réside dans la gestion des données en mémoire vive (RAM), garantissant ainsi des performances exceptionnelles en termes de temps d'accès. Néanmoins, cette architecture implique que les données ne sont pas automatiquement sauvegardées sur disque et risquent d'être perdues lors d'une panne système. Dans la pratique, Redis est fréquemment associé à une base de données relationnelle pour assurer la pérennité des informations.

---

## 1. Première Partie : Découverte et Commandes Fondamentales

### 1.1 Mise en place de Redis

**Prérequis : Docker doit être installé**

**Étape 1 : Récupération de l'image Redis**
```bash
docker pull redis
```
Cette instruction télécharge l'image officielle Redis depuis Docker Hub, préparant ainsi l'environnement pour la création de conteneurs.

**Étape 2 : Démarrage d'un conteneur Redis**
```bash
docker run --name mon-redis -d redis
```
Cette commande initialise et lance un conteneur Redis en mode détaché (arrière-plan), identifié par le nom `mon-redis`.

**Étape 3 : Connexion au serveur Redis**
```bash
docker exec -it mon-redis bash
redis-server --port 6380
```
L'accès au conteneur permet une interaction directe avec le serveur. La seconde commande lance Redis sur un port personnalisé (6380).

**Étape 4 : Ouverture du client Redis**
```bash
docker exec -it mon-redis bash
redis-cli
```
Cette commande active l'interface en ligne de commande (CLI) de Redis, permettant l'exécution de requêtes.

---

### 1.2 Création d'une paire clé-valeur

**Définir une première clé :**
```redis
SET demo "bonjour"
```
Cette instruction crée une clé nommée `demo` avec la valeur `bonjour`. Redis confirme l'opération en retournant `OK`.

---

### 1.3 Opérations CRUD dans Redis

Redis offre les fonctionnalités de création, lecture, modification et suppression de données. Voici des exemples concrets :

**Création d'une entrée :**
```redis
127.0.0.1:6379> SET user:1234 "wael"
OK
```
Cela ajoute une clé `user:1234` associée à la valeur `wael`.

**Lecture d'une valeur :**
```redis
127.0.0.1:6379> GET user:1234
"wael"
```
Cette commande extrait la valeur liée à la clé `user:1234`.

**Suppression d'une clé existante :**
```redis
127.0.0.1:6379> DEL user:1234
(integer) 1
```
La clé `user:1234` est effacée. Le retour de `1` confirme la suppression.

**Tentative de suppression d'une clé inexistante :**
```redis
127.0.0.1:6379> DEL user:123456
(integer) 0
```
Redis retourne `0` pour signaler qu'aucune clé n'a été trouvée et donc supprimée.

---

**Exemple pratique : Gestionnaire de compteur de visiteurs**

**Initialisation :**
```redis
SET cmpt 0
```
Création d'un compteur initialisé à zéro.

**Incrémentation :**
```redis
INCR cmpt
```
Augmente automatiquement la valeur du compteur d'une unité.

**Décrémentation :**
```redis
DECR cmpt
```
Diminue la valeur du compteur d'une unité.

---

**Gestion de l'expiration des clés**

**Définir une durée de vie :**
```redis
SET uneCle uneValeur
EXPIRE uneCle 30
```
La clé `uneCle` sera automatiquement supprimée après 30 secondes.

**Vérification du temps restant :**
```redis
TTL uneCle
```
Cette commande indique le nombre de secondes restantes avant l'expiration de la clé.

---

### 1.4 Structures de données avancées

**Les Listes**

**Ajout d'éléments :**
```redis
RPUSH mesCours "BDA"
RPUSH mesCours "WEB"
```
Ces commandes insèrent des éléments à la fin de la liste `mesCours`.

**Affichage complet :**
```redis
LRANGE mesCours 0 -1
```
Retourne l'intégralité des éléments contenus dans la liste.

**Suppression d'éléments :**
```redis
RPOP mesCours
LPOP mesCours
```
`RPOP` retire l'élément situé à droite (fin de liste), tandis que `LPOP` supprime celui à gauche (début de liste).

---

**Les Ensembles**

**Création et ajout :**
```redis
SADD myUsers wael
```
Insère l'élément `wael` dans l'ensemble `myUsers`.

**Consultation :**
```redis
SMEMBERS myUsers
```
Affiche tous les membres de l'ensemble `myUsers`.

**Retrait d'un élément :**
```redis
SREM myUsers wael
```
Supprime l'élément `wael` de l'ensemble.

---

### 1.5 Pub/Sub : Communication en temps réel

**Client 1 : Abonnement à un canal**
```redis
SUBSCRIBE mesCours
```
Ce client se met en écoute des messages diffusés sur le canal `mesCours`.

**Client 2 : Diffusion d'un message**
```redis
PUBLISH mesCours "Message de test :)"
```
Envoie un message sur le canal `mesCours`, qui sera reçu par tous les abonnés.

---

### 1.6 Basculer entre les bases de données

Redis propose par défaut 16 bases de données distinctes (numérotées de 0 à 15).

```redis
127.0.0.1:6379> SELECT 1
127.0.0.1:6379[1]> KEYS *
```
La commande `SELECT 1` active la base numéro 1. `KEYS *` liste ensuite toutes les clés présentes dans cette base.

---

## 2. Deuxième Partie : Systèmes de Recommandation avec Redis

### 2.1 Ensembles triés (ZSET)

**Ajout d'éléments avec scores :**
```redis
ZADD score1 10 "Karim" 20 "Ahmed"
```
Insère les membres `Karim` et `Ahmed` dans le sorted set `score1` avec leurs scores respectifs (10 et 20).

**Affichage ordonné :**
```redis
ZRANGE score1 0 -1
ZREVRANGE score1 0 -1
```
`ZRANGE` affiche les éléments par ordre croissant de score, `ZREVRANGE` dans l'ordre décroissant.

**Détermination du classement :**
```redis
ZRANK score1 "Karim"
```
Renvoie la position (rang) de `Karim` dans le classement.

---

## 3. Troisième Partie : Manipulation de Données avec Hashes

### 3.1 Utilisation des Hashes

**Création d'un hash :**
```redis
HSET user:11 username "aa" age 23 email "aa@aa.fr"
```
Définit un hash `user:11` contenant plusieurs attributs : nom d'utilisateur, âge et email.

**Récupération complète :**
```redis
HGETALL user:11
```
Affiche l'ensemble des champs et leurs valeurs associées dans le hash `user:11`.

**Modification par incrémentation :**
```redis
HINCRBY user:11 age 5
```
Augmente la valeur du champ `age` de 5 unités.

**Affichage des valeurs uniquement :**
```redis
HVALS user:11
```
Retourne exclusivement les valeurs (sans les noms de champs) du hash `user:11`.

---

**Fin du document**
