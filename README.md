# 🚀 Guide Simple : Déployer une App Web avec Docker sur VPS Ubuntu

## 📋 Ce dont vous avez besoin

- Un VPS Ubuntu (Hostinger ou autre)
- Un nom de domaine pointé vers votre VPS
- Un projet GitHub avec votre code

---

## 🏗️ Structure du Projet

```
mon-projet/
├── backend/           # API Express.js
│   ├── Dockerfile
│   └── package.json
├── frontend/          # App Next.js
│   ├── Dockerfile
│   └── package.json
├── docker-compose.yml
├── .env
└── deploy.sh         # Script de déploiement
```

---

## 🖥️ Étape 1: Préparer le VPS

Connectez-vous à votre VPS et exécutez :

```bash
# Se connecter au VPS
ssh root@votre-ip-vps

# Mettre à jour le système
apt update && apt upgrade -y

# Installer Docker
curl -fsSL https://get.docker.com -o get-docker.sh
sh get-docker.sh

# Installer Docker Compose
apt install docker-compose -y

# Installer Nginx et Certbot
apt install nginx certbot python3-certbot-nginx -y

# Démarrer les services
systemctl start nginx docker
systemctl enable nginx docker
```

---

## 📁 Étape 2: Créer les fichiers Docker

### `backend/Dockerfile`
```dockerfile
FROM node:18
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
EXPOSE 5000
CMD ["npm", "start"]
```

### `frontend/Dockerfile`
```dockerfile
FROM node:18
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
RUN npm run build
EXPOSE 3000
CMD ["npm", "start"]
```

### `docker-compose.yml`
```yaml
version: '3.8'
services:
  mysql:
    image: mysql:8
    environment:
      MYSQL_ROOT_PASSWORD: motdepasse123
      MYSQL_DATABASE: mabase
      MYSQL_USER: user
      MYSQL_PASSWORD: password
    volumes:
      - mysql_data:/var/lib/mysql
    ports:
      - "3306:3306"

  backend:
    build: ./backend
    environment:
      DB_HOST: mysql
      DB_USER: user
      DB_PASSWORD: password
      DB_NAME: mabase
    depends_on:
      - mysql
    ports:
      - "5000:5000"

  frontend:
    build: ./frontend
    depends_on:
      - backend
    ports:
      - "3000:3000"

volumes:
  mysql_data:
```

### `.env`
```bash
MYSQL_ROOT_PASSWORD=motdepasse123
MYSQL_DATABASE=mabase
MYSQL_USER=user
MYSQL_PASSWORD=password
```

---

## 🌐 Étape 3: Configurer Nginx

Créez le fichier de configuration :

```bash
nano /etc/nginx/sites-available/default
```

Remplacez le contenu par :

```nginx
server {
    listen 80;
    server_name mondomaine.com www.mondomaine.com;

    location / {
        proxy_pass http://localhost:3000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}

server {
    listen 80;
    server_name api.mondomaine.com;

    location / {
        proxy_pass http://localhost:5000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

Redémarrez Nginx :
```bash
nginx -t
systemctl reload nginx
```

---

## 🔐 Étape 4: Installer SSL (HTTPS)

```bash
# Obtenir les certificats SSL
certbot --nginx -d mondomaine.com -d www.mondomaine.com -d api.mondomaine.com

# Tester le renouvellement automatique
certbot renew --dry-run
```

---

## 📦 Étape 5: Script de Déploiement

Créez `deploy.sh` dans votre projet :

```bash
#!/bin/bash

echo "🚀 Début du déploiement..."

# Aller dans le dossier du projet
cd /root/mon-projet

# Télécharger les dernières modifications
git pull origin main

# Arrêter les conteneurs
docker-compose down

# Construire et démarrer
docker-compose up -d --build

# Attendre que tout soit prêt
sleep 30

# Vérifier que ça marche
if curl -f http://localhost:3000 > /dev/null 2>&1; then
    echo "✅ Frontend OK"
else
    echo "❌ Problème avec le frontend"
fi

if curl -f http://localhost:5000 > /dev/null 2>&1; then
    echo "✅ Backend OK"  
else
    echo "❌ Problème avec le backend"
fi

echo "🎉 Déploiement terminé !"
```

Rendre le script exécutable :
```bash
chmod +x deploy.sh
```

---

## 🔄 Étape 6: GitHub Actions (Automatisation)

Créez `.github/workflows/deploy.yml` :

```yaml
name: Déployer sur VPS

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    
    steps:
    - uses: actions/checkout@v3
    
    - name: Déployer sur le serveur
      uses: appleboy/ssh-action@v0.1.4
      with:
        host: ${{ secrets.VPS_HOST }}
        username: root
        key: ${{ secrets.VPS_SSH_KEY }}
        script: |
          cd /root/mon-projet
          git pull origin main
          ./deploy.sh
```

**Ajouter les secrets GitHub :**
- `VPS_HOST` : L'IP de votre VPS
- `VPS_SSH_KEY` : Votre clé SSH privée

---

## 🎯 Étape 7: Premier Déploiement

Sur votre VPS :

```bash
# Cloner votre projet
cd /root
git clone https://github.com/votre-username/mon-projet.git
cd mon-projet

# Premier déploiement
./deploy.sh
```

---

## ✅ Vérification

Testez vos sites :
- Frontend : `https://mondomaine.com`
- API : `https://api.mondomaine.com`

---

## 🔧 Commandes Utiles

```bash
# Voir les conteneurs qui tournent
docker-compose ps

# Voir les logs
docker-compose logs frontend
docker-compose logs backend

# Redémarrer un service
docker-compose restart backend

# Se connecter à la base de données
docker-compose exec mysql mysql -u user -p

# Nettoyer Docker
docker system prune -f
```

---

## 🚨 En cas de problème

1. **Site inaccessible** :
   ```bash
   systemctl status nginx
   nginx -t
   ```

2. **Conteneur qui ne démarre pas** :
   ```bash
   docker-compose logs nom-du-service
   ```

3. **Base de données** :
   ```bash
   docker-compose exec mysql mysql -u root -p
   ```

4. **Tout redémarrer** :
   ```bash
   docker-compose down
   docker-compose up -d --build
   ```

---

## 🎉 C'est tout !

Maintenant, à chaque fois que vous poussez du code sur GitHub, votre site se met à jour automatiquement !

**Pour déployer manuellement :**
```bash
ssh root@votre-vps
cd /root/mon-projet
./deploy.sh
```
