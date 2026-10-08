# OnY Va 🎸 (nom provisoire)

> Réseau social de concerts et festivals : découvrir des événements, voir lesquels de ses amis y vont, partager photos et avis.

**Équipe** : Arthur, Martin, Baptiste
**Cours** : NoSQL / architecture orientée services

> ⚠️ Tous les JSON ci-dessous sont des **exemples de travail**. Ils seront ajustés au fil du projet.

---

## Étape 1 : Description du projet

### Le besoin

Organiser une sortie concert est aujourd'hui dispersé : la billetterie est à un endroit, les dates sur les réseaux sociaux, le line-up sur le site du festival. On ne sait pas lesquels de ses amis y vont, et les photos et avis sont noyés dans d'autres applications.

### Notre réponse

Une application qui centralise tout :

> *OnY Va permet aux amateurs de concerts de découvrir des événements, de voir lesquels de leurs amis y participent, et de partager leurs retours après coup, le tout au même endroit.*

### Fonctionnalités visées

**Côté utilisateur**
- Un **fil d'actualité** : concerts à venir, publications des amis et des artistes suivis, photos.
- Un **calendrier personnel** : événements passés et à venir.
- Sur la page d'un événement : **la liste des amis qui y participent**.
- Suivre des amis et des artistes, indiquer « j'y vais ».

**Côté artiste**
- Un **fil dédié** : retours et avis des spectateurs sur ses événements.
- **Création d'événements** (minimum : nom, date, lieu).

### Choix technologiques (prévisionnel)

| Service | Rôle | Base NoSQL | Pourquoi |
|---|---|---|---|
| **Evenements** | créer et gérer les événements | MongoDB (document) | structure variable (concert simple vs festival multi-jours) |
| **Social** | comptes, amis, suivis, participations | Neo4j (graphe) | requêtes « amis (d'amis) qui y vont » |
| **Feed** | publications, photos, avis | MongoDB (document) | posts autonomes, tri par date |
| **Calendrier** *(étape 6)* | calendrier personnel précalculé | MongoDB (document) | vue dénormalisée prête à lire |

### Architecture

```
Evenements ──(EvenementCree, EvenementReprogramme)──► Social
Evenements ──(EvenementCree, EvenementReprogramme)──► Feed
Social ──(UtilisateurRenomme)──► Feed
Social ──(ParticipationAjoutee)──► Calendrier
Evenements ──(EvenementCree, EvenementReprogramme)──► Calendrier
```

Chaque service a **sa propre base** et ne fait jamais d'appel direct aux autres : il s'appuie sur des **réplicas** alimentés par des événements.

---

## Étape 2 : Services et commandes

Une commande est une demande de traitement adressée à un service.

```json
{
  "services": [
    {
      "nom": "Evenements",
      "base": "MongoDB",
      "commandes": [
        {
          "nom": "CreerEvenement",
          "donnees": { "nom": "string", "date": "date", "lieu": { "nom": "string", "ville": "string" }, "createurId": "string" }
        },
        {
          "nom": "ReprogrammerEvenement",
          "donnees": { "evenementId": "string", "nouvelleDate": "date" }
        },
        {
          "nom": "AnnulerEvenement",
          "donnees": { "evenementId": "string" }
        }
      ]
    },
    {
      "nom": "Social",
      "base": "Neo4j",
      "commandes": [
        { "nom": "CreerCompte", "donnees": { "pseudo": "string", "role": "utilisateur | artiste" } },
        { "nom": "ModifierPseudo", "donnees": { "userId": "string", "nouveauPseudo": "string" } },
        { "nom": "AjouterAmi", "donnees": { "userId": "string", "amiId": "string" } },
        { "nom": "SuivreArtiste", "donnees": { "userId": "string", "artisteId": "string" } },
        { "nom": "ParticiperAEvenement", "donnees": { "userId": "string", "evenementId": "string" } },
        { "nom": "AnnulerParticipation", "donnees": { "userId": "string", "evenementId": "string" } }
      ]
    },
    {
      "nom": "Feed",
      "base": "MongoDB",
      "commandes": [
        {
          "nom": "PublierPost",
          "donnees": { "auteurId": "string", "evenementId": "string (optionnel)", "type": "avis | photo | annonce", "texte": "string", "photos": ["string"] }
        },
        { "nom": "SupprimerPost", "donnees": { "postId": "string" } },
        { "nom": "LikerPost", "donnees": { "postId": "string", "userId": "string" } }
      ]
    }
  ]
}
```

Le service **Calendrier** ne reçoit aucune commande : il est uniquement alimenté par des événements (étape 6).

---

## Étape 3 : Agrégats

L'agrégat est l'objet complet que le service lit et écrit en une seule fois.

### Evenements (MongoDB)

```json
{
  "_id": "evt42",
  "nom": "Rock en Seine",
  "type": "festival",
  "dates": ["2027-08-27", "2027-08-28"],
  "lieu": { "nom": "Domaine de Saint-Cloud", "ville": "Saint-Cloud" },
  "createurId": "u10",
  "lineup": [
    { "artisteId": "a12", "nom": "Phoenix", "jour": "2027-08-28", "heure": "21:30" }
  ],
  "statut": "programme"
}
```

Un concert simple n'aura ni `lineup` ni plusieurs `dates` : c'est le schéma souple.

### Social (Neo4j)

Nœuds et relations :

```
(:Utilisateur {id, pseudo, role})
(:Evenement   {id, nom, date, ville, statut})      -- réplica

(Léa)-[:AMI_DE]->(Tom)
(Léa)-[:SUIT]->(Phoenix)
(Léa)-[:PARTICIPE]->(Rock en Seine)
```

Représentation JSON de l'agrégat d'un utilisateur :

```json
{
  "id": "u1",
  "pseudo": "Léa",
  "role": "utilisateur",
  "amis": ["u2", "u3"],
  "artistesSuivis": ["a12"],
  "participations": ["evt42"]
}
```

### Feed (MongoDB)

```json
{
  "_id": "post981",
  "auteur": { "id": "u1", "pseudo": "Léa" },
  "evenement": { "id": "evt42", "nom": "Rock en Seine" },
  "type": "avis",
  "note": 5,
  "texte": "Phoenix était incroyable !",
  "photos": ["img1.jpg"],
  "date": "2027-08-28T23:10:00",
  "likes": 14
}
```

`auteur.pseudo` et `evenement.nom` sont des **copies** (voir étape 4).

---

## Étape 4 : Découpler les services (événements et réplicas)

### Réplicas

**Réplica d'un événement dans Social** (juste ce qu'il faut pour « j'y vais » et « amis participants ») :

```json
{ "id": "evt42", "nom": "Rock en Seine", "date": "2027-08-27", "ville": "Saint-Cloud", "statut": "programme" }
```

**Réplica d'un événement dans Feed** (pour afficher le nom dans un post) :

```json
{ "id": "evt42", "nom": "Rock en Seine" }
```

**Réplica d'un utilisateur dans Feed** :

```json
{ "id": "u1", "pseudo": "Léa" }
```

### Événements

| Événement | Émetteur | Récepteur | Donnée modifiée | Rôle |
|---|---|---|---|---|
| `EvenementCree` | Evenements | Social, Feed, Calendrier | tout le réplica | **crée** le réplica |
| `EvenementReprogramme` | Evenements | Social, Feed, Calendrier | `date` | **met à jour** |
| `EvenementAnnule` | Evenements | Social, Calendrier | `statut` | met à jour |
| `UtilisateurRenomme` | Social | Feed | `pseudo` | met à jour la copie |
| `ParticipationAjoutee` | Social | Calendrier | liste des participations | alimente la projection |

Exemple : **création du réplica**

```json
{
  "evenement": "EvenementCree",
  "emetteur": "Evenements",
  "recepteur": "Social",
  "donnee": "replica",
  "nouvelleValeur": { "id": "evt42", "nom": "Rock en Seine", "date": "2027-08-27", "ville": "Saint-Cloud", "statut": "programme" }
}
```

Exemple : **mise à jour du réplica**

```json
{
  "evenement": "EvenementReprogramme",
  "emetteur": "Evenements",
  "recepteur": "Social",
  "donnee": "date",
  "nouvelleValeur": "2027-09-05"
}
```

### Autonomie

Si le service Evenements tombe, Social continue de gérer amis et participations, et Feed continue d'afficher les posts : chacun a sa copie des données dont il a besoin.

### Requête métier : amis qui participent à un événement (Neo4j)

```cypher
MATCH (moi:Utilisateur {id: "u1"})-[:AMI_DE]-(ami)-[:PARTICIPE]->(e:Evenement {id: "evt42"})
RETURN ami.pseudo
```

---

## Étape 6 : Projection vers un nouveau service (Calendrier)

Le service **Calendrier** ne possède aucune donnée d'origine. Il écoute `EvenementCree`, `EvenementReprogramme`, `EvenementAnnule` et `ParticipationAjoutee`, puis **transforme** les réplicas en une vue prête à lire pour le calendrier d'un utilisateur.

### Agrégat (MongoDB)

```json
{
  "_id": "u1",
  "evenements": [
    {
      "evenementId": "evt42",
      "nom": "Rock en Seine",
      "date": "2027-08-27",
      "ville": "Saint-Cloud",
      "statut": "programme",
      "amisParticipants": ["Tom", "Sarah"]
    },
    {
      "evenementId": "evt07",
      "nom": "Concert Jazz au Sunset",
      "date": "2026-06-12",
      "ville": "Paris",
      "statut": "programme",
      "amisParticipants": []
    }
  ]
}
```

### Requêtes

**Calendrier complet de l'utilisateur** (une seule lecture) :

```js
db.calendriers.findOne({ _id: "u1" })
```

**Événements à venir uniquement**, triés par date :

```js
db.calendriers.aggregate([
  { $match: { _id: "u1" } },
  { $unwind: "$evenements" },
  { $match: { "evenements.date": { $gte: "2026-10-08" } } },
  { $sort: { "evenements.date": 1 } }
])
```

Pour les événements passés, on remplace `$gte` par `$lt`.

---

## Pistes pour la suite

- Fil dédié aux artistes (retours des spectateurs sur leurs événements) : filtre sur les posts du Feed par `evenement.id`, ou second service de projection.
- Fil d'actualité personnalisé (posts des amis et artistes suivis).
- Recommandations : « 3 amis vont à ce concert ».
- Choix final du nom, du lieu (entité à part ?) et des champs des événements.

## Dépôt Git

- Dépôt : `onyva-concerts` https://github.com/martmartin1103-cyber/Projet_NoSQL_ABM_EFREI
