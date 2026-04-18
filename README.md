# Database & Automated Backup
https://roadmap.sh/projects/automated-backups
Base MongoDB sur EC2, backup quotidien vers S3 via GitHub Actions.

## Architecture

```
GitHub Actions (cron)
  └── SSH → EC2 Ubuntu
        ├── MongoDB (db: test) → mongodump → tar.gz
        └── aws s3 cp → bucket S3 (eu-west-3)
```

## EC2

```bash
chmod 400 cle.pem
ssh -i cle.pem ubuntu@<IP-PUBLIQUE>
```

Installation sur l'instance :

```bash
# MongoDB 8.0 (dépôt officiel)
curl -fsSL https://www.mongodb.org/static/pgp/server-8.0.asc | \
  sudo gpg -o /usr/share/keyrings/mongodb-server-8.0.gpg --dearmor
echo "deb [ arch=amd64,arm64 signed-by=/usr/share/keyrings/mongodb-server-8.0.gpg ] \
  https://repo.mongodb.org/apt/ubuntu noble/mongodb-org/8.0 multiverse" | \
  sudo tee /etc/apt/sources.list.d/mongodb-org-8.0.list
sudo apt-get update && sudo apt-get install -y mongodb-org
sudo systemctl enable --now mongod

# AWS CLI v2
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o awscliv2.zip
unzip awscliv2.zip && sudo ./aws/install
aws configure
```

## MongoDB

Base `test` avec deux collections :

- `finales_ucl` : finales de Ligue des Champions depuis 1956 (`annee`, `vainqueur`, `finaliste`, `score`)
- `buteurs_ldc` : top 20 buteurs historiques (`rang`, `joueur`, `nationalite`, `buts`, `matchs`, `ratio`)

## Backup

`mongodump` exporte la base en BSON, compressée en `.tar.gz` daté, puis uploadée sur S3.

Restauration :

```bash
mongorestore --db test /chemin/vers/backup
```

## GitHub Actions

`.github/workflows/mongodb-backup.yml` :

```yaml
name: MongoDB Backup

on:
  schedule:
    - cron: '0 2 * * *'
  workflow_dispatch:

jobs:
  backup:
    runs-on: ubuntu-latest
    steps:
      - name: Dump et upload S3
        uses: appleboy/ssh-action@master
        with:
          host: ${{ secrets.EC2_HOST }}
          username: ${{ secrets.EC2_USER }}
          key: ${{ secrets.EC2_SSH_KEY }}
          script: |
            mongodump --db test --out /tmp/backup
            tar -czf /tmp/backup-$(date +%Y-%m-%d).tar.gz -C /tmp backup
            aws s3 cp /tmp/backup-$(date +%Y-%m-%d).tar.gz s3://${{ secrets.S3_BUCKET_NAME }}/
```

## Secrets GitHub

| Secret | Valeur |
|--------|--------|
| `EC2_HOST` | IP publique EC2 |
| `EC2_USER` | `ubuntu` |
| `EC2_SSH_KEY` | contenu du `.pem` |
| `AWS_ACCESS_KEY_ID` | clé IAM |
| `AWS_SECRET_ACCESS_KEY` | clé secrète IAM |
| `AWS_REGION` | `eu-west-3` |
| `S3_BUCKET_NAME` | nom du bucket |

## S3 / IAM

Bucket en `eu-west-3`, utilisateur IAM dédié `mongodb-backup-user` avec `AmazonS3FullAccess`.

## Sécurité

Aucune clé en clair dans le repo : tout passe par les GitHub Secrets, injectés via `${{ secrets.* }}`.
