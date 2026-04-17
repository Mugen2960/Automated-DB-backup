# 🏆 UCL Database & Automated Backup

Documentation complète du projet : base de données MongoDB sur EC2 avec backup automatisé vers S3 via GitHub Actions.

---

## 📋 Table des matières

- [Architecture du projet](#architecture)
- [Instance EC2](#ec2)
- [MongoDB](#mongodb)
- [Collections](#collections)
- [Backup automatisé](#backup)
- [GitHub Actions](#github-actions)
- [AWS S3](#s3)

---

## 🏗️ Architecture du projet <a name="architecture"></a>

```
GitHub Actions (scheduler cron)
    │
    ├── SSH → EC2 Ubuntu
    │             │
    │             ├── MongoDB
    │             │     ├── Collection : finales_ucl
    │             │     └── Collection : buteurs_ldc
    │             │
    │             └── mongodump → tarball (.tar.gz)
    │
    └── aws s3 cp → Bucket S3 (eu-west-3)
```

---

## 🖥️ Instance EC2 <a name="ec2"></a>

### Caractéristiques
| Paramètre | Valeur |
|-----------|--------|
| OS | Ubuntu 24.04 LTS |
| Type | t2.micro (free tier) |
| Région | eu-west-3 (Paris) |
| Accès | SSH via clé `.pem` |

### Connexion SSH

```bash
# Sécuriser la clé (obligatoire)
chmod 400 /chemin/vers/votre-cle.pem

# Se connecter
ssh -i /chemin/vers/votre-cle.pem ubuntu@<IP-PUBLIQUE>
```

> **Note :** Le `-i` signifie *identity file* — il indique à SSH quelle clé utiliser.

### Prérequis installés sur l'EC2

```bash
# MongoDB
sudo apt-get install -y gnupg curl
curl -fsSL https://www.mongodb.org/static/pgp/server-8.0.asc | \
  sudo gpg -o /usr/share/keyrings/mongodb-server-8.0.gpg --dearmor
echo "deb [ arch=amd64,arm64 signed-by=/usr/share/keyrings/mongodb-server-8.0.gpg ] \
  https://repo.mongodb.org/apt/ubuntu noble/mongodb-org/8.0 multiverse" | \
  sudo tee /etc/apt/sources.list.d/mongodb-org-8.0.list
sudo apt-get update && sudo apt-get install -y mongodb-org
sudo systemctl start mongod && sudo systemctl enable mongod

# AWS CLI v2
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
sudo apt-get install -y unzip
unzip awscliv2.zip && sudo ./aws/install

# Configurer les credentials AWS
aws configure
```

### Security Group
Le port **22 (SSH)** doit être ouvert en inbound pour permettre la connexion depuis GitHub Actions.

---

## 🍃 MongoDB <a name="mongodb"></a>

### Installation
MongoDB n'est pas dans les dépôts officiels Ubuntu (changement de licence). Il faut passer par le dépôt officiel MongoDB — d'où l'ajout du `.list` file qui dit à `apt` où trouver les paquets.

### Commandes de base

```js
show dbs                        // lister les bases de données
use test                        // sélectionner une base
show collections                // lister les collections
db.maCollection.find().pretty() // afficher les documents
```

### Hiérarchie

```
MongoDB
└── test (base de données)
    ├── finales_ucl    (collection)
    └── buteurs_ldc    (collection)
```

> **Important :** Une base de données n'apparaît dans `show dbs` que si elle contient au moins un document.

---

## 📦 Collections <a name="collections"></a>

### finales_ucl

Contient les résultats de toutes les finales de la Ligue des Champions depuis 1956.

**Structure d'un document :**
```json
{
  "annee": 2025,
  "vainqueur": "PSG",
  "finaliste": "Inter Milan",
  "score": "5-0"
}
```

**Requêtes utiles :**
```js
// Clubs les plus titrés
db.finales_ucl.aggregate([
  { $group: { _id: "$vainqueur", titres: { $sum: 1 } } },
  { $sort: { titres: -1 } }
])

// Finales décidées aux tirs au but
db.finales_ucl.find({ note: /tirs au but/ })

// Toutes les finales du Real Madrid
db.finales_ucl.find({
  $or: [{ vainqueur: "Real Madrid" }, { finaliste: "Real Madrid" }]
})
```

---

### buteurs_ldc

Contient le top 20 des meilleurs buteurs de l'histoire de la Ligue des Champions.

**Structure d'un document :**
```json
{
  "rang": 1,
  "joueur": "Cristiano Ronaldo",
  "nationalite": "Portugal",
  "buts": 140,
  "matchs": 183,
  "ratio": 0.77
}
```

**Requêtes utiles :**
```js
// Meilleur ratio buts/match
db.buteurs_ldc.find().sort({ ratio: -1 })

// Joueurs avec plus de 50 buts
db.buteurs_ldc.find({ buts: { $gt: 50 } })
```

---

## 🔄 Backup automatisé <a name="backup"></a>

### Fonctionnement

1. GitHub Actions se déclenche selon le cron configuré
2. Connexion SSH à l'EC2
3. `mongodump` exporte la base `test` en BSON
4. Compression en tarball `.tar.gz` avec la date du jour
5. Upload du tarball sur le bucket S3

### Format des backups

Les fichiers exportés par `mongodump` :
- `.bson` — données en format binaire (natif MongoDB)
- `.json` — métadonnées de la collection

> Le BSON est illisible dans un éditeur texte, c'est normal. Pour le lire : `bsondump fichier.bson`

### Restaurer un backup

```bash
mongorestore --db test /chemin/vers/le/dossier/backup
```

---

## ⚙️ GitHub Actions <a name="github-actions"></a>

### Workflow `.github/workflows/mongodb-backup.yml`

```yaml
name: MongoDB Backup

on:
  schedule:
    - cron: '0 2 * * *'  # tous les jours à 2h du matin UTC
  workflow_dispatch:       # déclenchement manuel possible

jobs:
  backup:
    runs-on: ubuntu-latest

    steps:
      - name: Connexion SSH et backup
        uses: appleboy/ssh-action@master
        with:
          host: ${{ secrets.EC2_HOST }}
          username: ${{ secrets.EC2_USER }}
          key: ${{ secrets.EC2_SSH_KEY }}
          script: |
            mongodump --db test --out /tmp/backup
            tar -czf /tmp/backup-$(date +%Y-%m-%d).tar.gz -C /tmp backup

      - name: Upload sur S3
        uses: aws-actions/configure-aws-credentials@v2
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ${{ secrets.AWS_REGION }}

      - name: Copier vers S3
        uses: appleboy/ssh-action@master
        with:
          host: ${{ secrets.EC2_HOST }}
          username: ${{ secrets.EC2_USER }}
          key: ${{ secrets.EC2_SSH_KEY }}
          script: |
            aws s3 cp /tmp/backup-$(date +%Y-%m-%d).tar.gz s3://${{ secrets.S3_BUCKET_NAME }}/
```

### Secrets GitHub requis

| Secret | Description |
|--------|-------------|
| `AWS_ACCESS_KEY_ID` | Clé d'accès IAM |
| `AWS_SECRET_ACCESS_KEY` | Clé secrète IAM |
| `AWS_REGION` | Région AWS (`eu-west-3`) |
| `S3_BUCKET_NAME` | Nom du bucket S3 |
| `EC2_HOST` | IP publique de l'EC2 |
| `EC2_SSH_KEY` | Contenu du fichier `.pem` |
| `EC2_USER` | Utilisateur SSH (`ubuntu`) |

### Syntaxe cron

```
┌───── minute (0-59)
│ ┌───── heure (0-23)
│ │ ┌───── jour du mois (1-31)
│ │ │ ┌───── mois (1-12)
│ │ │ │ ┌───── jour de la semaine (0-6)
│ │ │ │ │
0 2 * * *   → tous les jours à 2h00 UTC
0 */2 * * * → toutes les 2 heures
```

---

## 🪣 AWS S3 <a name="s3"></a>

### Configuration

| Paramètre | Valeur |
|-----------|--------|
| Région | eu-west-3 (Paris) |
| Type | Standard |

### Utilisateur IAM

Un utilisateur IAM dédié `mongodb-backup-user` a été créé avec uniquement les permissions **AmazonS3FullAccess** — principe du moindre privilège.

> **Bonne pratique :** Ne jamais donner plus de permissions que nécessaire à un utilisateur IAM.

### Free Tier AWS S3
- **5 GB** de stockage gratuit
- **20 000 requêtes GET** / **2 000 requêtes PUT** par mois
- Valable **12 mois** après création du compte

---

## 🔐 Sécurité

- Les clés AWS et SSH ne sont **jamais** commitées dans le code
- Elles sont stockées dans les **GitHub Secrets** (chiffrées)
- Dans le workflow elles sont injectées via `${{ secrets.NOM_DU_SECRET }}`
- L'utilisateur IAM a les **permissions minimales** nécessaires

---

*Documentation générée le 17/04/2026*
