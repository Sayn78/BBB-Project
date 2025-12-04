# Installation BigBlueButton avec Greenlight v3

Guide complet d'installation de BigBlueButton (backend) et Greenlight v3 (frontend) sur Ubuntu.

## 📋 Table des matières

- [Prérequis](#prérequis)
- [Installation du Backend (BigBlueButton)](#installation-du-backend-bigbluebutton)
- [Installation du Frontend (Greenlight v3)](#installation-du-frontend-greenlight-v3)
- [Configuration Nginx](#configuration-nginx)
- [Certificat SSL](#certificat-ssl)
- [Création du compte administrateur](#création-du-compte-administrateur)
- [Vérification](#vérification)
- [Documentation utilisé](#Documentation-utilisé)

## 🔧 Prérequis

- Serveur Ubuntu 22.04 (Jammy)
- Deux domaines configurés :
  - `bbb.senan.fr` (Backend BigBlueButton)
  - `greenlight.senan.fr` (Frontend Greenlight)
- Accès root au serveur

## 🎥 Installation du Backend (BigBlueButton)

### 1. Télécharger le script d'installation

```bash
wget https://raw.githubusercontent.com/bigbluebutton/bbb-install/v3.0.x-release/bbb-install.sh
chmod +x bbb-install.sh
```

### 2. Installer BigBlueButton

```bash
./bbb-install.sh -v jammy-300 -s bbb.senan.fr -e anthony.senan@laposte.net -w
```

**Options :**
- `-v jammy-300` : Version BigBlueButton 3.0 pour Ubuntu 22.04
- `-s bbb.senan.fr` : Nom de domaine
- `-e anthony.senan@laposte.net` : Email pour Let's Encrypt
- `-w` : Installation de Greenlight (optionnel, nous l'installons séparément)

### 3. Récupérer le secret BBB

```bash
bbb-conf --secret
```

**Note :** Conservez précieusement le secret affiché, il sera nécessaire pour la configuration de Greenlight.

## 🌐 Installation du Frontend (Greenlight v3)

### 1. Créer le répertoire de travail

```bash
mkdir -p ~/greenlight-v3
cd ~/greenlight-v3
```

### 2. Installer les dépendances

```bash
apt-get update && apt-get upgrade -y
apt-get install -y docker.io docker-compose
systemctl start docker
systemctl enable docker
apt-get install -y nginx certbot python3-certbot-nginx
```

### 3. Créer le fichier docker-compose.yml

```bash
cat > docker-compose.yml << 'EOF'
version: '3.8'

services:
  postgres:
    image: postgres:15-alpine
    container_name: greenlight-v3-postgres
    restart: unless-stopped
    environment:
      POSTGRES_DB: greenlight-v3-production
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: CHANGE_ME_POSTGRES_PASSWORD
    volumes:
      - postgres-data:/var/lib/postgresql/data
    networks:
      - greenlight-network

  redis:
    image: redis:7-alpine
    container_name: greenlight-v3-redis
    restart: unless-stopped
    command: redis-server --appendonly yes
    volumes:
      - redis-data:/data
    networks:
      - greenlight-network

  greenlight:
    image: bigbluebutton/greenlight:v3
    container_name: greenlight-v3
    restart: unless-stopped
    env_file: .env
    depends_on:
      - postgres
      - redis
    ports:
      - "5050:3000"
    volumes:
      - greenlight-storage:/usr/src/app/storage
      - greenlight-log:/usr/src/app/log
    networks:
      - greenlight-network

volumes:
  postgres-data:
  redis-data:
  greenlight-storage:
  greenlight-log:

networks:
  greenlight-network:
    driver: bridge
EOF
```

### 4. Générer les mots de passe et clés secrètes

```bash
# Générer un mot de passe PostgreSQL sécurisé
POSTGRES_PASSWORD=$(openssl rand -hex 24)

# Générer la clé secrète Rails
SECRET_KEY_BASE=$(openssl rand -hex 64)
```

### 5. Créer le fichier .env

```bash
cat > .env << EOF
# PostgreSQL Configuration
DATABASE_URL=postgresql://postgres:${POSTGRES_PASSWORD}@postgres:5432/greenlight-v3-production

# Redis Configuration
REDIS_URL=redis://redis:6379

# Rails Configuration
SECRET_KEY_BASE=${SECRET_KEY_BASE}
RAILS_ENV=production

# BigBlueButton Configuration
BIGBLUEBUTTON_ENDPOINT=https://bbb.senan.fr/bigbluebutton/
BIGBLUEBUTTON_SECRET=1X37gsM6pGgBmf3x84CQ6BiknLcqsbIvxu8TAwdivk

# Application Configuration
RELATIVE_URL_ROOT=/

# SMTP Configuration (optionnel)
SMTP_SERVER=
SMTP_PORT=587
SMTP_DOMAIN=
SMTP_USERNAME=
SMTP_PASSWORD=
SMTP_AUTH=plain
SMTP_STARTTLS_AUTO=true
SMTP_SENDER=noreply@greenlight.senan.fr

# Optional: Brand Configuration
BRANDING_IMAGE=
PRIMARY_COLOR=
EOF
```

**⚠️ Important :** Remplacez `BIGBLUEBUTTON_SECRET` par le secret obtenu avec `bbb-conf --secret`

### 6. Mettre à jour le mot de passe dans docker-compose.yml

```bash
sed -i "s/CHANGE_ME_POSTGRES_PASSWORD/${POSTGRES_PASSWORD}/g" docker-compose.yml
```

### 7. Démarrer Greenlight

```bash
docker-compose up -d
```

## 🔒 Configuration Nginx

### 1. Créer la configuration Nginx

```bash
cat > /etc/nginx/sites-available/greenlight << 'EOF'
server {
    listen 80;
    listen [::]:80;
    server_name greenlight.senan.fr;

    location / {
        proxy_pass http://127.0.0.1:5050;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        
        # Timeouts
        proxy_connect_timeout 60s;
        proxy_send_timeout 60s;
        proxy_read_timeout 60s;
    }
}
EOF
```

### 2. Activer le site

```bash
# Créer le lien symbolique
ln -s /etc/nginx/sites-available/greenlight /etc/nginx/sites-enabled/

# Tester la configuration Nginx
nginx -t

# Recharger Nginx
systemctl reload nginx
```

## 🔐 Certificat SSL

### Obtenir un certificat Let's Encrypt

```bash
certbot --nginx -d greenlight.senan.fr --email anthony.senan@laposte.net --agree-tos --non-interactive
```

**⏳ Attendre environ 30 secondes** que les services démarrent complètement.

## 👤 Création du compte administrateur

```bash
docker exec -it greenlight-v3 bundle exec rake admin:create["Admin","admin@senan.fr","VotreMotDePasse123"]
```

**Paramètres :**
- `Admin` : Nom de l'administrateur
- `admin@senan.fr` : Email de l'administrateur
- `VotreMotDePasse123` : Mot de passe (à personnaliser)

## ✅ Vérification

### Vérifier l'état des services

```bash
# Vérifier les conteneurs Docker
docker ps

# Vérifier les logs de Greenlight
docker logs greenlight-v3

# Vérifier Nginx
systemctl status nginx

# Vérifier BigBlueButton
bbb-conf --check
```

### Accéder à l'interface

- **Greenlight :** https://greenlight.senan.fr
- **BigBlueButton API :** https://bbb.senan.fr/bigbluebutton/api

## 🔧 Commandes utiles

```bash
# Redémarrer Greenlight
cd ~/greenlight-v3
docker-compose restart

# Voir les logs
docker-compose logs -f

# Arrêter Greenlight
docker-compose down

# Redémarrer BigBlueButton
bbb-conf --restart

# Vérifier la configuration BBB
bbb-conf --status
```

## 📝 Notes importantes

1. **Sécurité :** Changez tous les mots de passe par défaut
2. **Firewall :** Assurez-vous que les ports 80, 443, 5050 (interne) sont accessibles
3. **DNS :** Vérifiez que les enregistrements DNS pointent correctement vers votre serveur
4. **Backup :** Pensez à sauvegarder régulièrement les volumes Docker et la base de données
5. **SMTP :** Configurez les paramètres SMTP dans `.env` pour l'envoi d'emails

## 🐛 Dépannage

### Greenlight ne démarre pas

```bash
# Vérifier les logs
docker-compose logs greenlight

# Recréer les conteneurs
docker-compose down
docker-compose up -d
```

### Erreur de connexion à BigBlueButton

Vérifiez que :
- Le secret BBB dans `.env` est correct
- L'URL `BIGBLUEBUTTON_ENDPOINT` se termine par `/bigbluebutton/`
- Le certificat SSL de BBB est valide

### Problème Nginx

```bash
# Tester la configuration
nginx -t

# Voir les logs d'erreur
tail -f /var/log/nginx/error.log
```

---

### Documentation utilisé

- https://docs.bigbluebutton.org/administration/install/
- https://github.com/bigbluebutton/bbb-install
- https://docs.bigbluebutton.org/greenlight/v3/install/
- https://ressources.labomedia.org/bigbluebutton_installation_configuration
- https://github.com/bigbluebutton/greenlight/blob/master/sample.env (pour le .env)
- https://github.com/bigbluebutton/greenlight/blob/master/docker-compose.yml (le docker-compose)
- https://docs.nginx.com/nginx/admin-guide/web-server/reverse-proxy/ (reverse proxy nginx)
- https://hub.docker.com/r/bigbluebutton/greenlight
