# CR TP n°2 : Système de Base de Données Orienté Documents (Architecture Client-Serveur) : MongoDB

**Wael NAJAR**

**2 février 2025**

---

## 1. Introduction

MongoDB constitue une solution de base de données NoSQL orientée documents, offrant des capacités de stockage et de gestion optimisées pour les données non structurées.

---

## 2. Importation des Données

L'importation de la collection de films a été réalisée via la commande suivante :

```bash
./mongoimport --db lesfilms --collection films films.json --jsonArray
```

---

## 3. Requêtes et Résultats

### 3.1 Vérification de l'importation

```javascript
db.films.count()
```
Cette requête retourne le nombre total de documents présents dans la collection.

---

### 3.2 Affichage d'un document exemple

```javascript
db.films.findOne()
```
Permet de visualiser un document représentatif afin de comprendre l'organisation structurelle des données.

---

### 3.3 Liste des films d'action

```javascript
db.films.find({genre: "Action"})
```

---

### 3.4 Nombre de films d'action

```javascript
db.films.find({genre: "Action"}).count()
```

---

### 3.5 Films d'action produits en France

```javascript
db.films.find({genre: "Action", country: "FR"})
```

---

### 3.6 Films d'action produits en France en 1963

```javascript
db.films.find({genre: "Action", country: "FR", year: 1963})
```

---

### 3.7 Affichage sans les grades

```javascript
db.films.find({genre: "Action", country: "FR"}, {grades: 0})
```

---

### 3.8 Affichage sans identifiants

```javascript
db.films.find({genre: "Action", country: "FR"}, {_id: 0})
```

---

### 3.9 Titres et grades des films d'action français

```javascript
db.films.find({genre: "Action", country: "FR"}, {_id: 0, title: 1, grades: 1})
```

---

### 3.10 Films d'action français avec une note supérieure à 10

```javascript
db.films.find({"genre": "Action", "country": "FR", "grades.note": {$gt: 10}})
```

---

### 3.11 Films ayant uniquement des notes supérieures à 40

```javascript
db.films.find({"genre": "Action", "country": "FR", "grades.note": {$gt: 40}})
```

---

### 3.12 Liste des genres disponibles

```javascript
db.films.distinct("genre")
```

---

### 3.13 Liste des grades disponibles

```javascript
db.films.distinct("grades.grade")
```

---

### 3.14 Films joués par certains artistes

```javascript
db.films.find({"actors.id": {$in: ["artist:4", "artist:18", "artist:11"]}})
```

---

### 3.15 Films sans résumé

```javascript
db.films.find({summary: {$exists: false}})
```

---

### 3.16 Films avec Leonardo DiCaprio en 1997

```javascript
db.films.find({"actors.lastname": "DiCaprio", "year": 1997})
```

---

### 3.17 Films avec Leonardo DiCaprio ou en 1997

```javascript
db.films.find({$or: [{"actors.lastname": "DiCaprio"}, {"year": 1997}]})
```

---

## 4. Conclusion

L'ensemble de ces requêtes démontre la capacité d'exploration et de manipulation efficace des documents stockés dans MongoDB, illustrant ainsi la puissance et la flexibilité de cette base de données NoSQL orientée documents.
