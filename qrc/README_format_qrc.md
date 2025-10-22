# 📄 Format `.qrc` – Spécification technique

Ce document décrit le format d’export propriétaire utilisé dans l’application **Qrcoder / Dandee** pour transférer de manière sécurisée des groupes d’éléments, de paramètres et de ressources visuelles (fonds d’écran).

---

## 🎯 Objectif

Le fichier `.qrc` vise à :

* encapsuler l’ensemble des données nécessaires à l’import d’un ou plusieurs groupes configurés,
* sécuriser les données métiers via un **cryptage ciblé**,
* **éviter les erreurs liées au stockage des images**, en les conservant en binaire non crypté,
* permettre un import rapide et fiable, même partiel.

---

## 🧱 Structure du fichier `.qrc`

Le fichier est composé de **trois sections distinctes** :

### 1. 📋 En-tête (non chiffré)

Le début du fichier contient un bloc **JSON lisible** avec les métadonnées d’export et des informations facilitant le parsing :

```json
{
  "version": "1.0",
  "exportedAt": "2025-07-23T15:12:04Z",
  "hasImages": true,
  "imageIds": ["img_123", "img_abc"],
  "encryption": {
    "algorithm": "AES-256-CBC",
    "iv": "3f2a1c4d5e6f7890a1b2c3d4e5f67890",
    "hmac": "HMAC-SHA256"
  },
  "blocks": {
    "encryptedJson": { "offset": 512, "length": 2048 },
    "images": [
      { "id": "img_123", "offset": 2560, "length": 102400 },
      { "id": "img_abc", "offset": 105000, "length": 204800 }
    ]
  }
}
```

* **iv** aléatoire : généré pour chaque export, inclus dans l’en-tête pour permettre le déchiffrement.
* **hmac** : champ décrivant l’algorithme et la taille du tag de vérification d’intégrité.
* **blocks** : listes des sections avec positions (`offset`) et longueurs (`length`), pour un parsing efficace et robuste.

Ce bloc est suivi d’un saut de ligne.

---

#### Métadonnées de version
Nous ajoutons :
- `appVersion` : version applicative (ex. `5.0.0+26`),
- `schemaVersion` : version du modèle de données interne (entier/SemVer),
- `fileVersion` : version du format `.qrc`.

Exemple :
{
  "fileVersion":"1.0.0",
  "appVersion":"5.0.0+26",
  "schemaVersion":1,
  "exportedAt":"2025-08-12T10:00:00Z",
  ...
}

#### Choix crypto
Pour V1, on **confirme** : `AES-256-CBC` + `HMAC-SHA256` (intégrité).
- IV aléatoire par fichier, encodé en hex dans l’en‑tête.
- HMAC calculé sur le bloc chiffré.

#### Sérialisation déterministe
Le JSON applicatif est **canonique** :
- clés triées, tableaux ordonnés (groupes & éléments par ordre utilisateur),
- dates au format ISO 8601 (UTC, `Z`),
- élimination des champs nuls.

#### Langue d’export
Le `.qrc` transporte **toutes les langue**. 

### 2. 🔒 Données chiffrées (JSON encrypté)

Le contenu JSON (groupes, éléments, préférences) est :

1. sérialisé en UTF‑8,
2. éventuellement compressé (optionnel),
3. chiffré avec **AES-256-CBC** en utilisant l’IV aléatoire de l’en-tête,
4. assorti d’un **HMAC-SHA256** calculé sur le bloc chiffré,
5. inséré entre deux balises :

```
==START_ENCRYPTED==
[octets chiffrés – brut]
==END_ENCRYPTED==
```

Le HMAC est stocké en fin de bloc chiffré pour vérifier l’intégrité avant déchiffrement.

---

### 3. 🖼️ Images (binaires non chiffrés)

Les images (fonds d’écran) sont ajoutées en clair, en binaire brut :

```
==IMAGE:img_123==
[octets PNG/JPEG]

==IMAGE:img_abc==
[octets PNG/JPEG]
```

Chaque bloc suit sa balise `==IMAGE:<id>==`. Les `offset` et `length` définis en en-tête permettent de les retrouver rapidement.

---

---

## 🔁 Exemple d’agencement complet

```text
{ "version":"1.0", … }
==START_ENCRYPTED==
[octets chiffrés + HMAC]
==END_ENCRYPTED==
==IMAGE:bg_montagne==
[octets PNG]
==IMAGE:bg_foret==
[octets PNG]
```

---

## ⚠️ Contraintes de parsing

* Aucun caractère précédant les balises `==...==`.
* Respect strict des `offset` et `length` définis.
* Lecture en mode binaire pour tous les blocs.

---

## 🔗 Gestion des références d’images

Nous utiliserons exclusivement **l’empreinte (hash)** pour lier les images :

* Chaque image est identifiée par son **SHA-256** calculé sur son flux binaire.
* Dans le JSON applicatif, on réfèrera l’image par ce `imageHash` : ex. `"imageHash":"da39a3ee5e6b4b0d3255bfef95601890afd80709"`.
* Dans la section binaire, chaque bloc portera la balise :

  ```text
  ==IMAGE:da39a3ee5e6b4b0d3255bfef95601890afd80709==
  [octets PNG/JPEG]
  ```
* À l’import, l’app :

  1. calcule le SHA-256 des images déjà stockées pour construire un cache `hash → image`.
  2. lorsqu’elle rencontre un `imageHash`, elle vérifie dans son cache :

     * si déjà présent : réutilise l’image existante, sans la dupliquer,
     * sinon : lit le bloc binaire, le stocke et ajoute son hash en cache.

Avantages :

* Évite totalement les doublons d’images dupliquées ;
* Pas besoin de mapping ou d’index supplémentaire ;
* Cohérence et idempotence du chargement.

### 📂 Nommage des fichiers images

Lors de l’écriture sur le système de fichiers local :

* Le fichier est enregistré sous le nom `<imageHash>.<ext>`, où :

  * `<imageHash>` est le SHA‑256 hexadécimal de l’image,
  * `<ext>` est l’extension d’origine (`png`, `jpg`, etc.) déduite du type MIME ou du contenu binaire.
* Exemple : une image PNG avec hash `da39a3ee5e6b4b0d3255bfef95601890afd80709` sera sauvegardée en tant que :
  `da39a3ee5e6b4b0d3255bfef95601890afd80709.png`.
* Cette convention permet :

  * un accès direct via hash,
  * une gestion simplifiée du cache,
  * la détection et la suppression aisée des images orphelines.

---

## 📌 Roadmap (évolutions restantes)

* utilisation de mot de passe (peut-être non pertinent dans le cas d'usage)
* 📦 Compression JSON avant chiffrement (optionnelle).

---
