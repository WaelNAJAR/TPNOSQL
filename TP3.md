# 📚 TP3 - Exploration Approfondie de CouchDB

> **Base de Données NoSQL orientée documents avec MapReduce**

---

## 📑 Table des matières

1. [Introduction](#1-introduction)
2. [Aux origines du MapReduce : matrice de liens et calculs distribués](#2-aux-origines-du-mapreduce--matrice-de-liens-et-calculs-distribués)
   - [Contexte et énoncé](#21-contexte-et-énoncé)
   - [Modèle de documents pour représenter M](#22-modèle-de-documents-pour-représenter-la-matrice-m)
   - [MapReduce pour calculer la norme des lignes](#23-mapreduce-pour-calculer-la-norme-des-lignes)
   - [MapReduce pour le produit matrice-vecteur](#24-mapreduce-pour-le-produit-matrice-vecteur)
3. [Installation et déploiement de CouchDB](#3-installation-et-déploiement-de-couchdb)
   - [Préparation de l'environnement Docker](#31-préparation-de-lenvironnement-docker)
   - [Lancement du conteneur CouchDB](#32-lancement-du-conteneur-couchdb)
   - [Accès à l'interface Web Fauxton](#33-accès-à-linterface-web-fauxton)
   - [Vérification de l'installation](#34-vérification-de-linstallation)
4. [Création et gestion des bases de données](#4-création-et-gestion-des-bases-de-données)
   - [Création d'une base de données](#41-création-dune-base-de-données)
   - [Insertion de documents](#42-insertion-de-documents)
   - [Lecture et consultation](#43-lecture-et-consultation)
   - [Mise à jour de documents](#44-mise-à-jour-de-documents)
5. [Définition et utilisation de vues MapReduce](#5-définition-et-utilisation-de-vues-mapreduce)
   - [Vue de comptage des films par année](#51-vue-de-comptage-des-films-par-année)
   - [Index par acteurs](#52-index-par-acteurs)
   - [Requêtes avancées sur les vues](#53-requêtes-avancées-sur-les-vues)

---

## 1. Introduction

### Qu'est-ce que CouchDB ?

**Apache CouchDB** est une base de données NoSQL orientée documents qui repose sur trois principes fondamentaux essentiels à son architecture :

| Principe | Description | Avantages |
|----------|-------------|-----------|
| 🗂️ **Stockage JSON** | Documents stockés nativement au format JSON | Format lisible, flexible et universel |
| 🌐 **API REST** | Accès complet via HTTP/HTTPS | Compatible avec tous les langages et plateformes |
| 🔄 **MapReduce** | Mécanisme d'analyse et d'agrégation | Requêtes complexes sur de grands volumes |

### Caractéristiques distinctives

CouchDB se distingue par plusieurs fonctionnalités avancées qui en font une solution particulièrement adaptée aux systèmes distribués :

- **🔁 Réplication incrémentale** : Synchronisation efficace entre instances distantes
- **🛡️ Tolérance aux pannes** : Architecture résiliente avec récupération automatique
- **📱 Offline-first** : Support natif des applications déconnectées (via PouchDB)
- **🌍 Multi-nœuds** : Clustering natif pour la haute disponibilité
- **⚡ MVCC** : Contrôle de concurrence multi-version sans blocage

### Objectifs de ce TP

Ce travail pratique couvre l'ensemble du cycle de vie d'utilisation de CouchDB :

1. ✅ Installation et configuration via Docker
2. ✅ Import et manipulation de données JSON
3. ✅ Création de vues MapReduce pour l'analyse
4. ✅ Application pratique : algorithme PageRank distribué
5. ✅ Requêtes avancées et optimisation

---

## 2. Aux origines du MapReduce : matrice de liens et calculs distribués

### 2.1 Contexte et énoncé

#### Problématique : Algorithme PageRank à l'échelle du Web

Considérons une matrice **M** de dimension **N × N** représentant les liens entre un très grand nombre de pages web (N pages, où N peut atteindre des milliards). Chaque lien est étiqueté par un poids représentant son importance.

**Caractéristiques du problème :**
- 🌐 **N très grand** : Des milliards de pages web
- 📊 **Matrice creuse** : Seulement quelques liens par page (< 0.01% de remplissage)
- 💾 **Impossible en RAM** : La matrice complète ne tient pas en mémoire
- 🔄 **Calculs itératifs** : Algorithme PageRank nécessite des produits matrice-vecteur répétés

#### Questions à résoudre

**Question 1 :** Proposer un modèle sous forme de documents structurés pour représenter une telle matrice. S'inspirer du cas PageRank vu en cours. Soit **C** la collection ainsi obtenue.

**Question 2 :** La ligne i peut être vue comme un vecteur à N dimensions décrivant la page Pi. Spécifier le traitement MapReduce qui calcule la norme de ces vecteurs à partir des documents de la collection C.

La norme d'un vecteur V(v₁, v₂, ..., vₙ) est :
```
‖V‖ = √(v₁² + v₂² + ... + vₙ²)
```

**Question 3 :** Calculer le produit de la matrice M avec un vecteur de dimension N, W(w₁, w₂, ..., wₙ). Le résultat est un vecteur φ défini par :
```
φᵢ = Σ(j=1 to N) Mᵢⱼ × wⱼ
```

On suppose que le vecteur W tient en mémoire RAM et est accessible comme variable statique par toutes les fonctions Map ou Reduce. Spécifier le traitement MapReduce qui implémente ce calcul.

### 2.2 Modèle de documents pour représenter la matrice M

#### Stratégie : Représentation sparse (creuse)

La matrice est énorme et creuse : **nous stockons uniquement les liens existants** (coefficients non nuls). Cette approche réduit drastiquement l'espace de stockage.

**Modèle choisi : "Ligne par document"**

Un document représente une page source Pᵢ (la ligne i) et contient la liste de ses liens sortants non nuls (colonnes j) avec leur poids Mᵢⱼ.

#### Structure détaillée d'un document

```json
{
  "_id": "page:P123",
  "type": "page",
  "i": 123,
  "url": "https://example.org/article-machine-learning",
  "title": "Introduction au Machine Learning",
  "outlinks": [
    {
      "j": 17,
      "to": "page:P17",
      "url_target": "https://example.org/neural-networks",
      "w": 0.42,
      "anchor_text": "réseaux de neurones"
    },
    {
      "j": 93,
      "to": "page:P93",
      "url_target": "https://example.org/deep-learning",
      "w": 0.35,
      "anchor_text": "apprentissage profond"
    },
    {
      "j": 156,
      "to": "page:P156",
      "url_target": "https://example.org/datasets",
      "w": 0.23,
      "anchor_text": "jeux de données"
    }
  ],
  "meta": {
    "crawl_timestamp": "2025-12-16T10:30:00Z",
    "page_rank": 0.0015,
    "in_degree": 42,
    "out_degree": 3
  }
}
```

#### Correspondance avec la matrice M

| Élément | Signification |
|---------|---------------|
| **Ligne i** | Document `page:Pᵢ` |
| **Mᵢⱼ ≠ 0** | Entrée dans `outlinks` avec `j` et poids `w` |
| **Mᵢⱼ = 0** | Absence dans `outlinks` (coefficient implicitement nul) |
| **Poids w** | Importance du lien (calculée selon divers critères) |

#### Avantages de ce modèle

✅ **Économie d'espace** : Seuls les liens existants sont stockés
✅ **Scalabilité** : Chaque document est indépendant et peut être distribué
✅ **Flexibilité** : Métadonnées additionnelles faciles à ajouter
✅ **Performance** : Pas de scan de coefficients nuls
✅ **Réplication** : Documents atomiques réplicables individuellement

#### Calcul de la taille de stockage

Pour une matrice de **1 milliard × 1 milliard** avec en moyenne **50 liens par page** :

```
Stockage matrice dense : 1×10⁹ × 1×10⁹ × 8 bytes = 8 × 10¹⁸ bytes = 8 exaoctets ❌

Stockage sparse :
- Nombre de liens : 1×10⁹ × 50 = 5×10¹⁰
- Taille par lien : ~100 bytes (avec métadonnées)
- Total : 5×10¹² bytes = 5 téraoctets ✅

Réduction : 1,600,000× plus compact !
```

### 2.3 MapReduce pour calculer la norme des lignes ‖Mᵢ‖

#### Formule mathématique

La norme euclidienne d'une ligne i est :

```
‖Mᵢ‖ = √(Σ(j=1 to N) Mᵢⱼ²)
```

Avec le modèle sparse, la somme se fait **uniquement sur les liens sortants présents** dans le document.

#### Principe de l'algorithme MapReduce

```mermaid
graph LR
    A[Document page:Pᵢ] --> B[Phase Map]
    B --> C[Émet i, w²]
    C --> D[Phase Shuffle]
    D --> E[Groupe par i]
    E --> F[Phase Reduce]
    F --> G[Sᵢ = Σw²]
    G --> H[‖Mᵢ‖ = √Sᵢ]
```

**Étapes détaillées :**

1. **Map** : Pour chaque lien (i, j) de poids w = Mᵢⱼ, émettre w² sous la clé i
2. **Shuffle** : Regroupement automatique des carrés par clé i
3. **Reduce** : Sommer les carrés pour obtenir Sᵢ = Σⱼ Mᵢⱼ²
4. **Post-traitement** : Calculer ‖Mᵢ‖ = √Sᵢ après lecture du résultat

#### Implémentation : Fonction Map (CouchDB view)

```javascript
function (doc) {
  // Filtrer uniquement les documents de type "page"
  if (doc.type !== "page" || !doc.outlinks) {
    return;
  }
  
  // Pour chaque lien sortant, émettre le carré du poids
  for (var k = 0; k < doc.outlinks.length; k++) {
    var w = doc.outlinks[k].w;
    
    // Clé = indice de la ligne i
    // Valeur = contribution au carré (w²)
    emit(doc.i, w * w);
  }
  
  // Cas particulier : page sans liens sortants
  // On émet 0 pour que la page apparaisse avec norme 0
  if (doc.outlinks.length === 0) {
    emit(doc.i, 0);
  }
}
```

#### Implémentation : Fonction Reduce

```javascript
// Réduction associative : somme des carrés
// En CouchDB, utiliser la fonction built-in optimisée
Reduce = "_sum"

// Équivalent manuel (moins performant) :
function (keys, values, rereduce) {
  return sum(values);
}
```

#### Résultat de la vue

La vue MapReduce retourne, pour chaque page i :

```json
{
  "rows": [
    {"key": 17, "value": 0.5789},
    {"key": 93, "value": 0.3421},
    {"key": 123, "value": 0.4074},
    {"key": 156, "value": 0.1892}
  ]
}
```

Où `value = Sᵢ = Σⱼ Mᵢⱼ²`

**Calcul final de la norme :**

```javascript
// Post-traitement côté client
results.rows.forEach(function(row) {
  var i = row.key;
  var Si = row.value;
  var norm = Math.sqrt(Si);
  
  console.log("Page " + i + " : norme = " + norm);
});
```

#### Exemple de calcul complet

Document page:P123 :
```json
{
  "i": 123,
  "outlinks": [
    {"j": 17, "w": 0.42},
    {"j": 93, "w": 0.35},
    {"j": 156, "w": 0.23}
  ]
}
```

**Phase Map :**
```
emit(123, 0.42²) = emit(123, 0.1764)
emit(123, 0.35²) = emit(123, 0.1225)
emit(123, 0.23²) = emit(123, 0.0529)
```

**Phase Reduce :**
```
S₁₂₃ = 0.1764 + 0.1225 + 0.0529 = 0.3518
```

**Norme finale :**
```
‖M₁₂₃‖ = √0.3518 = 0.593 ✅
```

### 2.4 MapReduce pour le produit matrice-vecteur φ = M · W

#### Formule mathématique

Le produit matrice-vecteur calcule, pour chaque ligne i :

```
φᵢ = Σ(j=1 to N) Mᵢⱼ × wⱼ
```

**Interprétation dans PageRank :**
- M = matrice de transition des liens
- W = vecteur PageRank actuel
- φ = nouveau vecteur PageRank (après une itération)

#### Hypothèse importante

Le vecteur W tient en mémoire RAM et est accessible comme **variable statique globale** par toutes les fonctions Map/Reduce.

**En pratique :**
- W peut être chargé au démarrage de la vue
- Ou injecté dans l'environnement d'exécution
- Ou stocké dans un document spécial accessible rapidement

#### Principe de l'algorithme MapReduce

**Étapes :**

1. **Map** : Pour le document (ligne) i, calculer le partiel Σⱼ∈out(i) Mᵢⱼ × wⱼ en lisant wⱼ dans W[j]
2. **Emit** : Émettre le résultat partiel sous la clé i
3. **Reduce** : Somme des partiels (utile si la ligne est fragmentée ; sinon triviale)

#### Implémentation : Fonction Map (produit matrice-vecteur)

```javascript
// Hypothèse : W est accessible globalement
// Exemple : var W = {0: 1.2, 1: -0.3, 17: 0.8, 93: 0.5, 156: 0.3, ...};

function (doc) {
  // Filtrer les documents de type page
  if (doc.type !== "page" || !doc.outlinks) {
    return;
  }
  
  // Accumulateur pour le calcul du produit partiel
  var accumulator = 0;
  
  // Pour chaque lien sortant de la page i
  for (var k = 0; k < doc.outlinks.length; k++) {
    var j = doc.outlinks[k].j;      // Indice de la colonne
    var mij = doc.outlinks[k].w;    // Coefficient Mᵢⱼ
    
    // Lecture de wⱼ depuis le vecteur W (en RAM)
    var wj = W[j];
    
    // Si wⱼ existe, ajouter la contribution Mᵢⱼ × wⱼ
    if (wj !== undefined && wj !== null) {
      accumulator += mij * wj;
    }
    // Si W[j] est absent : contribution 0 (on ignore)
  }
  
  // Émettre le résultat partiel pour la ligne i
  // Clé = i, Valeur = Σⱼ Mᵢⱼ × wⱼ
  emit(doc.i, accumulator);
}
```

#### Implémentation : Fonction Reduce

```javascript
// Somme associative des contributions partielles
// Utiliser la fonction built-in CouchDB pour l'efficacité
Reduce = "_sum"

// Équivalent manuel :
function (keys, values, rereduce) {
  return sum(values);
}
```

#### Résultat de la vue

La vue retourne, pour chaque ligne i, la valeur du produit :

```json
{
  "rows": [
    {"key": 17, "value": 0.856},
    {"key": 93, "value": 1.234},
    {"key": 123, "value": 0.567},
    {"key": 156, "value": 0.892}
  ]
}
```

Où `value = φᵢ = Σⱼ Mᵢⱼ × wⱼ`

#### Exemple de calcul complet

**Données d'entrée :**

Document page:P123 :
```json
{
  "i": 123,
  "outlinks": [
    {"j": 17, "w": 0.42},
    {"j": 93, "w": 0.35},
    {"j": 156, "w": 0.23}
  ]
}
```

Vecteur W :
```javascript
W = {
  17: 0.8,
  93: 0.5,
  156: 0.3
}
```

**Phase Map :**
```
Ligne 123:
  acc = 0
  acc += 0.42 × 0.8 = 0.336
  acc += 0.35 × 0.5 = 0.175
  acc += 0.23 × 0.3 = 0.069
  acc = 0.580

emit(123, 0.580)
```

**Phase Reduce :**
```
φ₁₂₃ = 0.580 ✅
```

#### Application : Algorithme PageRank itératif

Le produit matrice-vecteur est au cœur de PageRank :

```javascript
// Initialisation
var W_old = {0: 1/N, 1: 1/N, ..., N-1: 1/N};  // Vecteur uniforme
var alpha = 0.85;  // Facteur d'amortissement

// Itération PageRank
for (var iteration = 0; iteration < 50; iteration++) {
  // Calculer φ = M · W_old via la vue MapReduce
  var phi = computeMatrixVectorProduct(W_old);
  
  // Mise à jour : W_new = α × φ + (1-α) × (1/N)
  var W_new = {};
  for (var i in phi) {
    W_new[i] = alpha * phi[i] + (1 - alpha) / N;
  }
  
  // Convergence ?
  if (distance(W_new, W_old) < epsilon) {
    break;
  }
  
  W_old = W_new;
}
```

#### Optimisations possibles

**1. Pré-calcul des poids normalisés**
```javascript
// Au lieu de stocker Mᵢⱼ brut, stocker Mᵢⱼ / Σⱼ Mᵢⱼ
"outlinks": [
  {"j": 17, "w_normalized": 0.42}
]
```

**2. Combiners pour réduire le shuffle**
```javascript
// Agréger localement avant le shuffle
function combiner(key, values) {
  return sum(values);
}
```

**3. Partitionnement intelligent**
```javascript
// Distribuer les pages par communautés pour réduire le réseau
// Pages du même domaine → même nœud
```

---

## 3. Installation et déploiement de CouchDB

### 3.1 Préparation de l'environnement Docker

#### Création d'un volume persistant

Pour garantir la persistance des données CouchDB même après l'arrêt ou la suppression du conteneur, nous créons un volume Docker dédié :

```bash
docker volume create couchdb_data
```

**Vérification du volume créé :**
```bash
docker volume ls
docker volume inspect couchdb_data
```

**Résultat attendu :**
```json
[
    {
        "CreatedAt": "2025-12-16T10:30:00Z",
        "Driver": "local",
        "Labels": {},
        "Mountpoint": "/var/lib/docker/volumes/couchdb_data/_data",
        "Name": "couchdb_data",
        "Options": {},
        "Scope": "local"
    }
]
```

#### Avantages du volume Docker

| Avantage | Description |
|----------|-------------|
| 📦 **Persistance** | Les données survivent à l'arrêt/redémarrage du conteneur |
| 🔄 **Portabilité** | Facilite la migration entre environnements |
| 🔐 **Isolation** | Sépare les données du conteneur |
| 🚀 **Performance** | Accès direct au système de fichiers de l'hôte |
| 💾 **Backup** | Sauvegarde simplifiée du volume entier |

### 3.2 Lancement du conteneur CouchDB

#### Commande Docker complète

```bash
docker run \
  --name couchdb \
  -e COUCHDB_USER=NAJAR \
  -e COUCHDB_PASSWORD=wael \
  -p 5984:5984 \
  -v couchdb_data:/opt/couchdb/data \
  -d couchdb:latest
```

#### Explication détaillée des paramètres

| Paramètre | Description | Valeur |
|-----------|-------------|--------|
| `--name couchdb` | Nom du conteneur pour faciliter la gestion | `couchdb` |
| `-e COUCHDB_USER` | Variable d'environnement : utilisateur admin | `NAJAR` |
| `-e COUCHDB_PASSWORD` | Variable d'environnement : mot de passe admin | `wael` |
| `-p 5984:5984` | Mappage du port : hôte:conteneur | Port HTTP standard de CouchDB |
| `-v couchdb_data:/opt/couchdb/data` | Montage du volume pour la persistance | Répertoire de données interne |
| `-d` | Mode détaché (arrière-plan) | Conteneur s'exécute en background |
| `couchdb:latest` | Image Docker officielle | Dernière version stable |

#### Vérification du déploiement

**Lister les conteneurs en cours d'exécution :**
```bash
docker ps
```

**Résultat attendu :**
```
CONTAINER ID   IMAGE     COMMAND                  CREATED          STATUS          PORTS                    NAMES
a1b2c3d4e5f6   couchdb   "tini -- /docker-ent…"   10 seconds ago   Up 9 seconds    0.0.0.0:5984->5984/tcp   couchdb
```

**Consulter les logs du conteneur :**
```bash
docker logs couchdb
```

**Logs de démarrage typiques :**
```
[notice] 2025-12-16T10:30:00.123456Z couchdb@localhost <0.268.0> -------- Application couch started on node 'couchdb@localhost'
[notice] 2025-12-16T10:30:00.234567Z couchdb@localhost <0.350.0> -------- Apache CouchDB has started on http://0.0.0.0:5984/
```

#### Commandes de gestion du conteneur

```bash
# Arrêter le conteneur
docker stop couchdb

# Démarrer le conteneur
docker start couchdb

# Redémarrer le conteneur
docker restart couchdb

# Afficher les statistiques en temps réel
docker stats couchdb

# Supprimer le conteneur (les données dans le volume persistent)
docker rm couchdb

# Accéder au shell du conteneur
docker exec -it couchdb bash
```

### 3.3 Accès à l'interface Web Fauxton

#### URL d'accès

CouchDB inclut **Fauxton**, une interface web moderne et intuitive pour l'administration :

```
http://localhost:5984/_utils/
```

#### Connexion

**Identifiants :**
- 👤 **Utilisateur** : `NAJAR`
- 🔑 **Mot de passe** : `wael`

#### Fonctionnalités de Fauxton

L'interface Fauxton offre une expérience complète de gestion :

| Section | Fonctionnalités |
|---------|-----------------|
| 🗄️ **Databases** | Créer, supprimer, explorer les bases |
| 📄 **Documents** | CRUD sur les documents avec éditeur JSON |
| 🔍 **Views** | Créer et tester des vues MapReduce |
| 📊 **Queries** | Exécuter des requêtes Mango (SQL-like) |
| 🔄 **Replication** | Configurer la réplication entre instances |
| 📈 **Monitoring** | Statistiques et performances en temps réel |
| ⚙️ **Configuration** | Paramètres du serveur CouchDB |
| 👥 **Users** | Gestion des utilisateurs et permissions |

#### Aperçu de l'interface

**Dashboard principal :**
- Liste des bases de données avec tailles et nombre de documents
- Statistiques globales (requêtes/sec, documents, stockage)
- Accès rapide aux fonctions courantes

**Éditeur de documents :**
- Coloration syntaxique JSON
- Validation en temps réel
- Historique des révisions (_rev)
- Attachements (fichiers binaires)

**Éditeur de vues MapReduce :**
- Tests interactifs avec aperçu des résultats
- Débogage avec logs
- Sauvegarde dans des Design Documents

### 3.4 Vérification de l'installation

#### Test de connexion HTTP

**Commande curl :**
```bash
curl -X GET http://NAJAR:wael@localhost:5984
```

**Réponse attendue (JSON) :**
```json
{
  "couchdb": "Welcome",
  "version": "3.5.1",
  "git_sha": "09e000e08",
  "uuid": "a1b2c3d4e5f6g7h8i9j0k1l2m3n4o5p6",
  "features": [
    "access-ready",
    "partitioned",
    "pluggable-storage-engines",
    "reshard",
    "scheduler"
  ],
  "vendor": {
    "name": "The Apache Software Foundation"
  }
}
```

#### Test sans authentification

```bash
# Cette commande doit échouer (401 Unauthorized)
curl -X GET http://localhost:5984
```

**Réponse :**
```json
{
  "error": "unauthorized",
  "reason": "Authentication required."
}
```

✅ **Ceci confirme que l'authentification est correctement configurée.**

#### Vérification de la configuration

**Lister toutes les bases de données :**
```bash
curl -X GET http://NAJAR:wael@localhost:5984/_all_dbs
```

**Réponse initiale (système) :**
```json
["_replicator", "_users"]
```

> **💡 Note :** Les bases `_replicator` et `_users` sont des bases système créées automatiquement par CouchDB.

**Obtenir les statistiques du serveur :**
```bash
curl -X GET http://NAJAR:wael@localhost:5984/_stats
```

**Informations sur la configuration :**
```bash
curl -X GET http://NAJAR:wael@localhost:5984/_node/_local/_config
```

---

## 4. Création et gestion des bases de données

### 4.1 Création d'une base de données

#### Commande de création

```bash
curl -X PUT http://NAJAR:wael@localhost:5984/films
```

**Réponse de succès :**
```json
{"ok": true}
```

#### Règles de nommage des bases

Les noms de bases de données dans CouchDB doivent respecter les contraintes suivantes :

| Règle | Description | Exemples |
|-------|-------------|----------|
| ✅ **Minuscules uniquement** | Pas de majuscules | `films`, `movies_2024` |
| ✅ **Caractères autorisés** | Lettres, chiffres, et `_ $ ( ) + - /` | `my_db`, `test-2024` |
| ❌ **Premier caractère** | Ne doit pas être un chiffre | ❌ `2024films`, ✅ `films2024` |
| ❌ **Caractères interdits** | Espaces, @, #, etc. | ❌ `my database`, ❌ `films@2024` |
| 📏 **Longueur** | Maximum 256 caractères | - |

#### Créer plusieurs bases

```bash
# Base pour les films
curl -X PUT http://NAJAR:wael@localhost:5984/films

# Base pour les acteurs
curl -X PUT http://NAJAR:wael@localhost:5984/acteurs

# Base pour les réalisateurs
curl -X PUT http://NAJAR:wael@localhost:5984/realisateurs
```

#### Vérifier l'existence d'une base

```bash
curl -X GET http://NAJAR:wael@localhost:5984/films
```

**Réponse :**
```json
{
  "db_name": "films",
  "purge_seq": "0-g1AAAABteJzLYWBg4MhgTmHgS04sKU7NS8",
  "update_seq": "0-g1AAAABteJzLYWBg4MhgTmHgS04sKU7NS8",
  "sizes": {
    "file": 8360,
    "external": 0,
    "active": 0
  },
  "other": {
    "data_size": 0
  },
  "doc_count": 0,
  "doc_del_count": 0,
  "disk_size": 8360,
  "disk_format_version": 8,
  "data_size": 0,
  "compact_running": false,
  "cluster": {
    "q": 2,
    "n": 1,
    "w": 1,
    "r": 1
  },
  "instance_start_time": "0"
}
```

#### Supprimer une base

```bash
curl -X DELETE http://NAJAR:wael@localhost:5984/films
```

⚠️ **Attention :** Cette opération est **irréversible** et supprime toutes les données !

### 4.2 Insertion de documents

#### Insertion unitaire (POST)

La méthode **POST** laisse CouchDB générer automatiquement un `_id` unique :

```bash
curl -X POST http://NAJAR:wael@localhost:5984/films \
  -H "Content-Type: application/json" \
  -d '{
    "title": "Inception",
    "year": 2010,
    "genre": "Science-Fiction",
    "director": {
      "first_name": "Christopher",
      "last_name": "Nolan"
    },
    "cast": ["Leonardo DiCaprio", "Marion Cotillard", "Tom Hardy"],
    "rating": 8.8,
    "duration": 148
  }'
```

**Réponse :**
```json
{
  "ok": true,
  "id": "8a7e9c4b5f6d3a2e1b0c9d8e7f6a5b4c",
  "rev": "1-967a00dff5e02add41819138abb3284d"
}
```

#### Insertion avec ID personnalisé (PUT)

La méthode **PUT** permet de spécifier un `_id` personnalisé :

```bash
curl -X PUT http://NAJAR:wael@localhost:5984/films/inception-2010 \
  -H "Content-Type: application/json" \
  -d '{
    "title": "Inception",
    "year": 2010,
    "genre": "Science-Fiction"
  }'
```

> **💡 Conseil :** Utilisez des IDs sémantiques pour faciliter les références croisées.

#### Insertion en masse (_bulk_docs)

Pour insérer plusieurs documents simultanément, utilisez l'endpoint `_bulk_docs` :

**Préparer le fichier JSON :**

Créez un fichier `films_couchdb.json` :
```json
{
  "docs": [
    {
      "title": "The Matrix",
      "year": 1999,
      "genre": "Science-Fiction",
      "director": {
        "first_name": "Lana",
        "last_name": "Wachowski"
      },
      "cast": ["Keanu Reeves", "Laurence Fishburne", "Carrie-Anne Moss"],
      "rating": 8.7
    },
    {
      "title": "Pulp Fiction",
      "year": 1994,
      "genre": "Crime",
      "director": {
        "first_name": "Quentin",
        "last_name": "Tarantino"
      },
      "cast": ["John Travolta", "Uma Thurman", "Samuel L. Jackson"],
      "rating": 8.9
    },
    {
      "title": "The Shawshank Redemption",
      "year": 1994,
      "genre": "Drama",
      "director": {
        "first_name": "Frank",
        "last_name": "Darabont"
      },
      "cast": ["Tim Robbins", "Morgan Freeman"],
      "rating": 9.3
    },
    {
      "title": "Interstellar",
      "year": 2014,
      "genre": "Science-Fiction",
      "director": {
        "first_name": "Christopher",
        "last_name": "Nolan"
      },
      "cast": ["Matthew McConaughey", "Anne Hathaway", "Jessica Chastain"],
      "rating": 8.6
    },
    {
      "title": "The Dark Knight",
      "year": 2008,
      "genre": "Action",
      "director": {
        "first_name": "Christopher",
        "last_name": "Nolan"
      },
      "cast": ["Christian Bale", "Heath Ledger", "Aaron Eckhart"],
      "rating": 9.0
    }
  ]
}
```

**Commande d'insertion :**
```bash
curl -X POST http://NAJAR:wael@localhost:5984/films/_bulk_docs \
  -H "Content-Type: application/json" \
  -d @films_couchdb.json
```

**Réponse :**
```json
[
  {
    "ok": true,
    "id": "a1b2c3d4e5f6g7h8i9j0k1l2m3n4o5p6",
    "rev": "1-abc123"
  },
  {
    "ok": true,
    "id": "b2c3d4e5f6g7h8i9j0k1l2m3n4o5p6q7",
    "rev": "1-def456"
  },
  ...
]
```

#### Avantages de _bulk_docs

✅ **Performance** : Une seule requête HTTP pour plusieurs documents
✅ **Atomicité** : Tous les documents sont insérés dans la même transaction
✅ **Efficacité réseau** : Réduit considérablement la latence
✅ **Batch optimal** : Recommandé pour l'import de données volumineuses

### 4.3 Lecture et consultation

#### Lire un document par son ID

```bash
curl -X GET http://NAJAR:wael@localhost:5984/films/<id_document>
```

**Exemple :**
```bash
curl -X GET http://NAJAR:wael@localhost:5984/films/8a7e9c4b5f6d3a2e1b0c9d8e7f6a5b4c
```

**Réponse :**
```json
{
  "_id": "8a7e9c4b5f6d3a2e1b0c9d8e7f6a5b4c",
  "_rev": "1-967a00dff5e02add41819138abb3284d",
  "title": "Inception",
  "year": 2010,
  "genre": "Science-Fiction",
  "director": {
    "first_name": "Christopher",
    "last_name": "Nolan"
  },
  "cast": ["Leonardo DiCaprio", "Marion Cotillard", "Tom Hardy"],
  "rating": 8.8,
  "duration": 148
}
```

#### Lister tous les documents

```bash
curl -X GET http://NAJAR:wael@localhost:5984/films/_all_docs
```

**Réponse (sans contenu) :**
```json
{
  "total_rows": 5,
  "offset": 0,
  "rows": [
    {"id": "8a7e...", "key": "8a7e...", "value": {"rev": "1-967a..."}},
    {"id": "a1b2...", "key": "a1b2...", "value": {"rev": "1-abc1..."}},
    ...
  ]
}
```

#### Lister avec le contenu des documents

```bash
curl -X GET http://NAJAR:wael@localhost:5984/films/_all_docs?include_docs=true
```

**Réponse (avec contenu) :**
```json
{
  "total_rows": 5,
  "offset": 0,
  "rows": [
    {
      "id": "8a7e9c4b5f6d3a2e1b0c9d8e7f6a5b4c",
      "key": "8a7e9c4b5f6d3a2e1b0c9d8e7f6a5b4c",
      "value": {"rev": "1-967a00dff5e02add41819138abb3284d"},
      "doc": {
        "_id": "8a7e9c4b5f6d3a2e1b0c9d8e7f6a5b4c",
        "_rev": "1-967a00dff5e02add41819138abb3284d",
        "title": "Inception",
        "year": 2010,
        "genre": "Science-Fiction"
      }
    },
    ...
  ]
}
```

#### Lire plusieurs documents spécifiques

```bash
curl -X POST http://NAJAR:wael@localhost:5984/films/_all_docs?include_docs=true \
  -H "Content-Type: application/json" \
  -d '{
    "keys": [
      "8a7e9c4b5f6d3a2e1b0c9d8e7f6a5b4c",
      "inception-2010"
    ]
  }'
```

#### Paramètres de requête utiles

| Paramètre | Description | Exemple |
|-----------|-------------|---------|
| `include_docs=true` | Inclut le contenu complet des documents | `?include_docs=true` |
| `limit=N` | Limite le nombre de résultats | `?limit=10` |
| `skip=N` | Saute les N premiers résultats | `?skip=5` |
| `descending=true` | Ordre décroissant | `?descending=true` |
| `startkey="..."` | Commence à partir d'une clé | `?startkey="a"` |
| `endkey="..."` | Termine à une clé | `?endkey="z"` |

### 4.4 Mise à jour de documents

#### Principe du MVCC (Multi-Version Concurrency Control)

CouchDB utilise un système de **versioning** pour gérer la concurrence. Chaque modification crée une nouvelle révision avec un nouveau `_rev`.

**Règle importante :** Pour modifier un document, vous **devez** fournir son `_rev` actuel.

#### Mise à jour via PUT

**Étape 1 : Récupérer le document avec son _rev actuel**
```bash
curl -X GET http://NAJAR:wael@localhost:5984/films/inception-2010
```

**Réponse :**
```json
{
  "_id": "inception-2010",
  "_rev": "1-967a00dff5e02add41819138abb3284d",
  "title": "Inception",
  "year": 2010,
  "genre": "Science-Fiction"
}
```

**Étape 2 : Modifier et renvoyer avec le _rev**
```bash
curl -X PUT http://NAJAR:wael@localhost:5984/films/inception-2010 \
  -H "Content-Type: application/json" \
  -d '{
    "_rev": "1-967a00dff5e02add41819138abb3284d",
    "title": "Inception",
    "year": 2010,
    "genre": "Science-Fiction",
    "rating": 8.8,
    "awards": ["Oscar du meilleur son", "Oscar des meilleurs effets visuels"]
  }'
```

**Réponse de succès :**
```json
{
  "ok": true,
  "id": "inception-2010",
  "rev": "2-abc123def456789"
}
```

> **💡 Note :** Le `_rev` a changé de `1-...` à `2-...` (nouvelle révision).

#### Gestion des conflits

Si deux clients tentent de modifier simultanément le même document :

**Client 1 :**
```bash
curl -X PUT http://NAJAR:wael@localhost:5984/films/inception-2010 \
  -d '{"_rev": "1-967a...", "title": "Inception (Updated)"}'
# Succès : {"ok": true, "rev": "2-abc123..."}
```

**Client 2 (avec le même _rev 1-...) :**
```bash
curl -X PUT http://NAJAR:wael@localhost:5984/films/inception-2010 \
  -d '{"_rev": "1-967a...", "title": "Inception (Modified)"}'
# Échec : Erreur 409 Conflict
```

**Réponse d'erreur :**
```json
{
  "error": "conflict",
  "reason": "Document update conflict."
}
```

**Solution :** Le client 2 doit :
1. Récupérer la dernière version (avec `_rev` actuel)
2. Fusionner ses modifications
3. Renvoyer avec le nouveau `_rev`

#### Suppression de documents

```bash
curl -X DELETE http://NAJAR:wael@localhost:5984/films/inception-2010?rev=2-abc123def456789
```

> ⚠️ **Important :** La suppression nécessite également le `_rev` actuel.

**Réponse :**
```json
{
  "ok": true,
  "id": "inception-2010",
  "rev": "3-deleted..."
}
```

> **💡 Note :** Le document n'est pas réellement supprimé ; il est marqué comme `_deleted: true` avec une nouvelle révision.

---

## 5. Définition et utilisation de vues MapReduce

### 5.1 Vue de comptage des films par année

#### Objectif

Créer une vue MapReduce qui compte le nombre de films pour chaque année de sortie.

#### Fonction Map

```javascript
function (doc) {
  // Vérifier que le document contient une année
  if (doc.year) {
    // Émettre : clé = année, valeur = titre du film
    emit(doc.year, doc.title);
  }
}
```

**Explication du Map :**
- **Clé émise** : `doc.year` (l'année de sortie)
- **Valeur émise** : `doc.title` (le titre du film)
- **Résultat** : Chaque film émet une paire (année, titre)

**Exemple de données émises :**
```
emit(2010, "Inception")
emit(1999, "The Matrix")
emit(1994, "Pulp Fiction")
emit(1994, "The Shawshank Redemption")
emit(2014, "Interstellar")
emit(2008, "The Dark Knight")
```

#### Fonction Reduce

```javascript
function (keys, values, rereduce) {
  // Compter le nombre de valeurs (= nombre de films)
  return values.length;
}
```

**Ou utiliser la fonction built-in :**
```javascript
Reduce = "_count"
```

**Explication du Reduce :**
- **Entrée** : Liste de toutes les valeurs (titres) pour une même année
- **Sortie** : Nombre de films pour cette année

**Exemple de traitement Reduce :**
```
Année 1994 : ["Pulp Fiction", "The Shawshank Redemption"] → count = 2
Année 1999 : ["The Matrix"] → count = 1
Année 2008 : ["The Dark Knight"] → count = 1
Année 2010 : ["Inception"] → count = 1
Année 2014 : ["Interstellar"] → count = 1
```

#### Création de la vue dans CouchDB

**Méthode 1 : Via Fauxton (Interface Web)**

1. Ouvrir Fauxton : `http://localhost:5984/_utils/`
2. Aller dans la base `films`
3. Cliquer sur "Design Documents" → "New View"
4. Créer un Design Document : `_design/analytics`
5. Créer une vue : `by_year`
6. Copier les fonctions Map et Reduce
7. Sauvegarder

**Méthode 2 : Via curl (Ligne de commande)**

```bash
curl -X PUT http://NAJAR:wael@localhost:5984/films/_design/analytics \
  -H "Content-Type: application/json" \
  -d '{
    "_id": "_design/analytics",
    "views": {
      "by_year": {
        "map": "function(doc) { if (doc.year) { emit(doc.year, doc.title); } }",
        "reduce": "_count"
      }
    }
  }'
```

#### Interrogation de la vue

**Obtenir le nombre de films par année :**
```bash
curl -X GET "http://NAJAR:wael@localhost:5984/films/_design/analytics/_view/by_year?group=true"
```

**Résultat :**
```json
{
  "rows": [
    {"key": 1994, "value": 2},
    {"key": 1999, "value": 1},
    {"key": 2008, "value": 1},
    {"key": 2010, "value": 1},
    {"key": 2014, "value": 1}
  ]
}
```

**Obtenir uniquement les films d'une année spécifique :**
```bash
curl -X GET "http://NAJAR:wael@localhost:5984/films/_design/analytics/_view/by_year?key=1994&reduce=false"
```

**Résultat :**
```json
{
  "rows": [
    {"key": 1994, "id": "doc1", "value": "Pulp Fiction"},
    {"key": 1994, "id": "doc2", "value": "The Shawshank Redemption"}
  ]
}
```

**Obtenir les films entre deux années :**
```bash
curl -X GET "http://NAJAR:wael@localhost:5984/films/_design/analytics/_view/by_year?startkey=2008&endkey=2014&reduce=false"
```

### 5.2 Index par acteurs

#### Objectif

Créer une vue MapReduce qui indexe tous les films par acteur, permettant de trouver rapidement tous les films dans lesquels un acteur a joué.

#### Fonction Map

```javascript
function (doc) {
  // Vérifier que le document contient une liste d'acteurs
  if (doc.cast && Array.isArray(doc.cast)) {
    // Pour chaque acteur, émettre une paire (acteur, titre)
    doc.cast.forEach(function(actor) {
      emit(actor, doc.title);
    });
  }
}
```

**Explication du Map :**
- **Itération** : Parcourt tous les acteurs du tableau `doc.cast`
- **Émission multiple** : Un film avec N acteurs émet N paires
- **Clé** : Nom de l'acteur
- **Valeur** : Titre du film

**Exemple de données émises :**
```
// Pour "Inception" : ["Leonardo DiCaprio", "Marion Cotillard", "Tom Hardy"]
emit("Leonardo DiCaprio", "Inception")
emit("Marion Cotillard", "Inception")
emit("Tom Hardy", "Inception")

// Pour "The Matrix" : ["Keanu Reeves", "Laurence Fishburne", "Carrie-Anne Moss"]
emit("Keanu Reeves", "The Matrix")
emit("Laurence Fishburne", "The Matrix")
emit("Carrie-Anne Moss", "The Matrix")
```

#### Fonction Reduce

```javascript
function (keys, values, rereduce) {
  // Compter le nombre de films pour chaque acteur
  return values.length;
}
```

**Ou utiliser la fonction built-in :**
```javascript
Reduce = "_count"
```

**Explication du Reduce :**
- **Entrée** : Liste de tous les titres de films pour un acteur donné
- **Sortie** : Nombre de films dans lesquels l'acteur a joué

#### Création de la vue

```bash
curl -X PUT http://NAJAR:wael@localhost:5984/films/_design/analytics \
  -H "Content-Type: application/json" \
  -d '{
    "_id": "_design/analytics",
    "_rev": "1-...",
    "views": {
      "by_year": {
        "map": "function(doc) { if (doc.year) { emit(doc.year, doc.title); } }",
        "reduce": "_count"
      },
      "by_actor": {
        "map": "function(doc) { if (doc.cast && Array.isArray(doc.cast)) { doc.cast.forEach(function(actor) { emit(actor, doc.title); }); } }",
        "reduce": "_count"
      }
    }
  }'
```

> **💡 Note :** N'oubliez pas d'inclure le `_rev` actuel du Design Document si vous le mettez à jour.

#### Interrogation de la vue

**Compter le nombre de films par acteur :**
```bash
curl -X GET "http://NAJAR:wael@localhost:5984/films/_design/analytics/_view/by_actor?group=true"
```

**Résultat :**
```json
{
  "rows": [
    {"key": "Anne Hathaway", "value": 1},
    {"key": "Carrie-Anne Moss", "value": 1},
    {"key": "Christian Bale", "value": 1},
    {"key": "Heath Ledger", "value": 1},
    {"key": "John Travolta", "value": 1},
    {"key": "Keanu Reeves", "value": 1},
    {"key": "Laurence Fishburne", "value": 1},
    {"key": "Leonardo DiCaprio", "value": 1},
    {"key": "Marion Cotillard", "value": 1},
    {"key": "Matthew McConaughey", "value": 1},
    {"key": "Morgan Freeman", "value": 1},
    {"key": "Samuel L. Jackson", "value": 1},
    {"key": "Tim Robbins", "value": 1},
    {"key": "Tom Hardy", "value": 2},
    {"key": "Uma Thurman", "value": 1}
  ]
}
```

**Lister tous les films d'un acteur spécifique :**
```bash
curl -X GET "http://NAJAR:wael@localhost:5984/films/_design/analytics/_view/by_actor?key=\"Leonardo DiCaprio\"&reduce=false"
```

**Résultat :**
```json
{
  "rows": [
    {"key": "Leonardo DiCaprio", "id": "doc1", "value": "Inception"}
  ]
}
```

> **💡 Astuce :** L'encodage de l'URL pour les espaces : utilisez `%20` ou mettez entre guillemets `\"Leonardo DiCaprio\"`

**Trouver les acteurs dont le nom commence par "C" :**
```bash
curl -X GET "http://NAJAR:wael@localhost:5984/films/_design/analytics/_view/by_actor?startkey=\"C\"&endkey=\"D\"&group=true"
```

### 5.3 Requêtes avancées sur les vues

#### Vue composée : Films par genre et année

**Fonction Map avec clé composite :**
```javascript
function (doc) {
  if (doc.genre && doc.year) {
    // Clé composite : [genre, année]
    emit([doc.genre, doc.year], {
      title: doc.title,
      rating: doc.rating
    });
  }
}
```

**Création de la vue :**
```bash
curl -X PUT http://NAJAR:wael@localhost:5984/films/_design/analytics \
  -H "Content-Type: application/json" \
  -d '{
    "views": {
      "by_genre_year": {
        "map": "function(doc) { if (doc.genre && doc.year) { emit([doc.genre, doc.year], {title: doc.title, rating: doc.rating}); } }",
        "reduce": "_count"
      }
    }
  }'
```

**Requêtes possibles :**

```bash
# Tous les films de Science-Fiction
curl ".../_view/by_genre_year?startkey=[\"Science-Fiction\"]&endkey=[\"Science-Fiction\",{}]&reduce=false"

# Nombre de films par genre et année
curl ".../_view/by_genre_year?group=true"

# Films de Science-Fiction entre 2000 et 2020
curl ".../_view/by_genre_year?startkey=[\"Science-Fiction\",2000]&endkey=[\"Science-Fiction\",2020]&reduce=false"
```

#### Vue avec agrégation : Rating moyen par genre

**Fonction Map :**
```javascript
function (doc) {
  if (doc.genre && doc.rating) {
    emit(doc.genre, doc.rating);
  }
}
```

**Fonction Reduce personnalisée :**
```javascript
function (keys, values, rereduce) {
  if (rereduce) {
    // Combiner des résultats partiels
    var total = 0;
    var count = 0;
    values.forEach(function(v) {
      total += v.sum;
      count += v.count;
    });
    return {sum: total, count: count, avg: total / count};
  } else {
    // Premier niveau de réduction
    var sum = values.reduce(function(a, b) { return a + b; }, 0);
    return {sum: sum, count: values.length, avg: sum / values.length};
  }
}
```

**Ou utiliser `_stats` (built-in) :**
```javascript
Reduce = "_stats"
```

**Résultat avec _stats :**
```json
{
  "rows": [
    {
      "key": "Action",
      "value": {
        "sum": 9.0,
        "count": 1,
        "min": 9.0,
        "max": 9.0,
        "sumsqr": 81.0
      }
    },
    {
      "key": "Science-Fiction",
      "value": {
        "sum": 26.1,
        "count": 3,
        "min": 8.6,
        "max": 8.8,
        "sumsqr": 227.41
      }
    }
  ]
}
```

#### Paramètres de pagination

**Pagination avec limit et skip :**
```bash
# Page 1 (10 premiers résultats)
curl ".../_view/by_actor?limit=10&skip=0&reduce=false"

# Page 2 (10 résultats suivants)
curl ".../_view/by_actor?limit=10&skip=10&reduce=false"
```

**Pagination efficace avec startkey_docid :**
```bash
# Page 1
curl ".../_view/by_actor?limit=10&reduce=false"

# Page 2 (en utilisant le dernier ID de la page 1)
curl ".../_view/by_actor?limit=10&startkey=\"Actor Name\"&startkey_docid=last_doc_id&skip=1&reduce=false"
```

#### Optimisation des vues

**Bonnes pratiques :**

1. ✅ **Utiliser les fonctions built-in** : `_count`, `_sum`, `_stats` sont optimisées
2. ✅ **Éviter les émissions excessives** : Limiter le nombre de `emit()` par document
3. ✅ **Indexer sélectivement** : Ne créer des vues que pour les requêtes fréquentes
4. ✅ **Tester localement** : Valider les vues sur un échantillon avant déploiement
5. ✅ **Utiliser stale=ok** : Pour des lectures rapides avec index possiblement obsolète

**Exemple avec stale=ok :**
```bash
# Vue possiblement obsolète, mais réponse instantanée
curl ".../_view/by_year?group=true&stale=ok"
```

---

## 🎯 Conclusion

Ce TP a permis d'explorer en profondeur **Apache CouchDB** et le paradigme **MapReduce**, de l'installation à l'utilisation avancée.

### Points clés maîtrisés

✅ **Architecture distribuée** : Compréhension des principes de stockage sparse et de l'algorithme PageRank

✅ **Installation Docker** : Déploiement rapide et configuration avec persistance

✅ **API RESTful** : Manipulation complète des bases et documents via HTTP/curl

✅ **Vues MapReduce** : Création de vues analytiques pour l'agrégation et l'indexation

✅ **MVCC** : Gestion de la concurrence et des conflits avec le versioning

### Compétences acquises

- 🔍 **Modélisation NoSQL** : Représentation efficace de données complexes en JSON
- ⚡ **Performance** : Optimisation des requêtes avec indexes MapReduce
- 🔄 **Scalabilité** : Compréhension des principes de distribution et réplication
- 🛠️ **Pratique** : Maîtrise de l'API CouchDB et des outils d'administration

### Applications pratiques

Ce TP couvre des cas d'usage réels :
- 📊 **Analyse de graphes** : Algorithme PageRank pour le ranking de pages web
- 🎬 **Système de recommandation** : Index par acteurs et genres pour suggérer des films
- 📈 **Business Intelligence** : Agrégations et statistiques sur de gros volumes


**Auteur :** Wael NAJAR  
**Année universitaire :** 2025-2026  
**Groupe :** G5SI2  
**Date :** Décembre 2025  
**Version :** 1.0 - TP3 Exploration CouchDB
