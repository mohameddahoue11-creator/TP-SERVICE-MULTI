## Multi-services avec Docker Compose

### Partie 1 — Décrire et lancer l'application

### Commandes exécutées et vérifications
Après avoir écrit le fichier `docker-compose.yml`, nous avons vérifié sa syntaxe puis lancé les services :

```bash
docker compose config -q
docker compose up -d --build
docker compose ps
[📸<img width="917" height="382" alt="Capture d&#39;écran 2026-11-14 160415" src="https://github.com/user-attachments/assets/78765682-3187-4eb4-a0c0-c2e9b92de5c4" />
 <img width="900" height="72" alt="Capture d&#39;écran 2026-11-14 160541" src="https://github.com/user-attachments/assets/2f643757-78b9-4320-8122-69de17b3fbca" />


Réponses aux questions
Q1. Quels sont les noms des trois conteneurs créés ? D'où vient le préfixe tp3 ?
Les conteneurs sont tp3-api-1, tp3-db-1 et tp3-minio-1. Le préfixe tp3 vient du nom du dossier dans lequel on a lancé la commande (~/tp3). Docker Compose l'utilise comme nom de projet par défaut.

Q2. Quelles adresses IP ont les trois services sur le réseau tp3_default ? À quelle adresse le nom db se résout-il depuis l'API ?
Chaque service possède une adresse IP interne au réseau Docker (ex: 172.18.0.x). Depuis l'API, le nom db se résout automatiquement vers l'adresse IP du conteneur de base de données grâce au DNS interne de Docker. C'est pourquoi l'URL DATABASE_URL utilise @db:5432.

Q3. Le port 5432 est-il joignable depuis la VM (nc) ? Pourquoi ? Est-ce un bon choix de sécurité ?
Non, il n'est pas joignable depuis la VM (hôte). C'est un excellent choix de sécurité : le port de la base de données n'est pas publié sur la machine hôte. Seuls les conteneurs du réseau interne peuvent s'y connecter, la base est donc invisible depuis l'extérieur.

Q4. À quoi servent le healthcheck de db et la condition service_healthy ?
Le healthcheck teste régulièrement si la base de données est prête à accepter des connexions (pg_isready). La condition service_healthy force Docker à attendre que ce test soit réussi avant de démarrer l'API. Sans ça, l'API démarrerait trop tôt et planterait en ne trouvant pas la base.

Partie 2 — Vérifier la persistance
Commandes exécutées
bash
docker compose exec db psql -U app -d app -c "CREATE TABLE test (msg TEXT); INSERT INTO test VALUES ('bonjour TP3');"
docker compose exec db psql -U app -d app -c "SELECT * FROM test;"
docker compose down
docker volume ls | grep tp3
docker compose up -d
docker compose exec db psql -U app -d app -c "SELECT * FROM test;"
docker compose down -v
Réponses aux questions
Q1. Remplissez un tableau : pour down puis down -v, qu'est-il arrivé aux conteneurs, au réseau, aux volumes et à la ligne bonjour TP3 ?

Commande	Conteneurs	Réseau	Volumes	Ligne "bonjour TP3"
docker compose down	Supprimés	Supprimé	Conservés	Conservée (on la retrouve au rallumage)
docker compose down -v	Supprimés	Supprimé	Supprimés	Perdue (la table est recréée vide)
Q2. Où se trouvent physiquement les données de PostgreSQL ? Quel est l'équivalent cloud d'un volume attaché à un serveur (cours, stockage bloc) ?
Les données sont physiquement stockées sur le disque de la VM, dans le dossier indiqué par docker volume inspect tp3_db-data (généralement /var/lib/docker/volumes/tp3_db-data/_data). Dans le cloud, l'équivalent d'un volume attaché à un serveur est un disque bloc (comme Amazon EBS ou Azure Managed Disk).

Q3. Pourquoi ne faut-il jamais lancer down -v sur un serveur de production sans sauvegarde ?
Parce que l'option -v supprime définitivement les volumes, et donc toutes les données de la base (et du stockage). Sur un serveur de production, cela équivaut à un effacement complet des données des utilisateurs sans possibilité de retour en arrière (sauf sauvegarde).

Partie 3 — Brancher l'API sur la base et le stockage
Commandes exécutées
L'API a été mise à jour avec la version 2.0. Les fichiers requirements.txt et app.py ont été créés. L'image a été reconstruite :

bash
docker compose up -d --build api
curl -s http://localhost:5000/sante
![Uploading Capture d'écran 2026-11-14 160541.png…]()

Tests de dépôt, liste et lecture de fichiers :

bash
echo "Relevé de notes fictif" > releve.txt
curl -s -F "fichier=@releve.txt" http://localhost:5000/fichiers
curl -s -F "fichier=@/etc/hostname" http://localhost:5000/fichiers
curl -s http://localhost:5000/fichiers
curl -s http://localhost:5000/fichiers/1<img width="969" height="246" alt="Capture d&#39;écran 2026-11-14 155653" src="https://github.com/user-attachments/assets/8d18194f-c93e-450f-a93e-f9ed701b5c09" />

<img width="955" height="116" alt="Capture d&#39;écran 2026-11-14 160810" src="https://github.com/user-attachments/assets/39f287cc-8139-4da4-ab1f-c70462bdabf6" />

<img width="965" height="176" alt="Capture d&#39;écran 2026-11-14 160847" src="https://github.com/user-attachments/assets/3757fd8a-aaf5-4f79-84d5-981169a84f71" />
<img width="955" height="58" alt="Capture d&#39;écran 2026-11-14 160604" src="https://github.com/user-attachments/assets/70396a8a-1572-4937-9d65-43b3d63fb26d" />

Les tests ont été concluants : les fichiers sont bien enregistrés en base, stockés dans MinIO, listés, et lus correctement.

Partie 4 — Sortir les secrets et nettoyer
Commandes exécutées
bash
cat > .env <<'EOF'
POSTGRES_PASSWORD=app_mdp
MINIO_ROOT_PASSWORD=tp3-secret-123
EOF
chmod 600 .env
echo ".env" >> .gitignore
# Modification de docker-compose.yml pour utiliser ${POSTGRES_PASSWORD} et ${MINIO_ROOT_PASSWORD}
docker compose config | grep -n 'PASSWORD'
docker compose up -d
docker compose down
<img width="926" height="162" alt="Capture d&#39;écran 2026-11-14 160929" src="https://github.com/user-attachments/assets/a6dc4e40-e503-4076-b3a3-ce92507ecbf3" />

<img width="969" height="246" alt="Capture d&#39;écran 2026-11-14 155653" src="https://github.com/user-attachments/assets/ce77debe-d0c9-4b7d-be16-95e8ef534f52" />

Réponses aux questions
Q1. Que montre docker compose config à la place de ${POSTGRES_PASSWORD} ? Le secret a-t-il disparu de la machine ou seulement du dépôt Git ?
Il affiche la vraie valeur du mot de passe (par exemple app_mdp), car il lit le fichier .env. Le secret n'a pas disparu de la machine, il a juste été retiré du fichier docker-compose.yml pour ne pas être poussé sur Git. Il est simplement caché dans un fichier .env local (lui-même ignoré par Git).

Q2. Citez deux solutions plus sûres qu'un fichier .env pour gérer des secrets dans le cloud.

Les services de gestion de secrets (ex: AWS Secrets Manager, Azure Key Vault, HashiCorp Vault) : ils stockent les mots de passe de manière chiffrée et les injectent à la volée.

Les identités managées (ex: AWS IAM Roles for Service Accounts) : le conteneur s'authentifie directement auprès du cloud sans avoir besoin de stocker un mot de passe (le cloud lui donne un jeton temporaire).
