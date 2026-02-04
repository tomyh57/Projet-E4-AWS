# Projet-E4-AWS

## Présentation du Projet

Ce dépôt contient l'ensemble des scripts de déploiement et la documentation technique pour la mise en place d'une infrastructure AWS. Le projet répond à trois besoins majeurs :

    MVP Ecommerce : Déploiement d'une application de vente avec paiement Stripe.

    Migration WordPress : Mise en place d'une stack WordPress pour test.

    Sécurité & Scalabilité : Isolation réseau (VPC), bases de données gérées (RDS) et haute disponibilité.

## Partie 1 : Déploiement de l'Infrastructure de Base

Cette étape consiste à mettre en place le réseau, la base de données hautement disponible et les serveurs d'application pour le MVP Ecommerce et la stack WordPress.

### 1. Architecture Réseau (VPC & Segmentation)

Pour garantir la sécurité, nous isolons les serveurs web dans un sous-réseau public et la base de données dans un sous-réseau privé.

Création du VPC

    aws ec2 create-vpc --cidr-block 10.1.0.0/16 --query 'Vpc.VpcId' --output text

Création des Sous-réseaux (Subnets)

    aws ec2 create-subnet --vpc-id vpc-02b34098f121dd272 --cidr-block 10.1.1.0/24 --availability-zone us-east-2a
    aws ec2 create-subnet --vpc-id vpc-02b34098f121dd272 --cidr-block 10.1.2.0/24 --availability-zone us-east-2b

Configuration de l'accès Internet

    aws ec2 create-internet-gateway --query 'InternetGateway.InternetGatewayId' --output text
    aws ec2 attach-internet-gateway --vpc-id vpc-02b34098f121dd272 --internet-gateway-id igw-0a1efe59caa7ac25b
    
    aws ec2 create-route aws ec2 create-route-table --vpc-id vpc-02b34098f121dd272
    aws ec2 create-route --route-table-id rtb-09d0bf2e568cca802 --destination-cidr-block 0.0.0.0/0 --gateway-id igw-0a1efe59caa7ac25b
    aws ec2 associate-route-table --route-table-id rtb-09d0bf2e568cca802 --subnet-id subnet-05e8caac9f82d6e34

    aws ec2 create-route-table --vpc-id vpc-02b34098f121dd272
    aws ec2 associate-route-table --route-table-id rtb-006fc7197f7b28952 --subnet-id subnet-0e6601b928cbf6996

### 2. Base de Données Managée (Amazon RDS)

Le client exige une haute disponibilité et moins de gestion possible. Nous utilisons RDS avec l'option Multi-AZ.

    aws rds create-db-subnet-group \
        --db-subnet-group-name rds-private-group \
        --db-subnet-group-description "Subnets pour DB privée" \
        --subnet-ids subnet-ID-A subnet-ID-B
        
    aws rds create-db-instance \
        --db-instance-identifier database-1-bfhk \
        --db-instance-class db.m7g.large \
        --engine mariadb \
        --allocated-storage 20 \
        --db-subnet-group-name rds-private-group \
        --multi-az \
        --master-username admin \
        --master-user-password VOTRE_MOT_DE_PASSE_SECURISE

### 3. Intance (EC2)

Déploiement des deux POC : l'application Ecommerce (Stripe) et WordPress.

    aws ec2 run-instances \
        --image-id ami-004cab0cd49679909\
        --count 1 \
        --instance-type t3.small \
        --key-name KeyVM-BFHK \
        --subnet-id subnet-05e8caac9f82d6e34 \

    aws ec2 run-instances \
        --image-id ami-0ecbf87a86cfa9a8f \
        --count 1 \
        --instance-type t3.small \
        --key-name KeyVM-BFHK \
        --subnet-id subnet-05e8caac9f82d6e34 \

### 4. Stratégie de Sauvegarde et Stockage (S3)

Pour répondre à la consigne de sauvegarde régulière des applications et bases de données.

    aws s3 mb s3://backups-client-ecommerce-2026 --region us-east-2


# Installation Applicative : POC Ecommerce (Rocket-Ecommerce)

## 1. Accès sécurisé à l'instance

Connexion SSH à l'instance via son adresse IP publique à l'aide de la clé privée PEM.

    ssh -i "KeyVM-BFHK.pem" ubuntu@3.149.236.21

### 2. Préparation du système et dépendances
Mise à jour des dépôts et installation des utilitaires nécessaires (unzip pour l'archive et client MySQL pour tester la connexion RDS).

    sudo apt update
    sudo apt install unzip default-mysql-client -y

### 3. Configuration de l'Application
Extraction de l'archive et configuration des variables d'environnement pour lier l'application à la base de données Amazon RDS créée en Partie 1.

    mysql -u admin -h database-1-bfhk.ctykc0gcmtio.us-east-2.rds.amazonaws.com -p

Configuration des variables d'environnement

    nano .env
    
Comme illustré dans la configuration du fichier .env, l'application communique avec Stripe grâce à deux clés uniques :

    STRIPE_PUBLISHABLE_KEY : Utilisée côté front-end pour générer le formulaire de paiement sécurisé.

    STRIPE_SECRET_KEY : Utilisée côté back-end (Django) pour valider les transactions avec les serveurs de Stripe.

<img width="1219" height="543" alt="env" src="https://github.com/user-attachments/assets/38682fe9-7e6a-4477-b859-503cd0e20d3d" />


### 4. Déploiement via Docker & Docker Compose

    sudo apt install docker.io docker-compose-v2 -y
    sudo systemctl enable --now docker

    docker compose up -d --build

### Docker compose
<img width="571" height="591" alt="dockercompose" src="https://github.com/user-attachments/assets/a66f0db5-94c2-4be1-ae4b-fae442a21f45" />

### Appseed.conf 

<img width="703" height="346" alt="appseed" src="https://github.com/user-attachments/assets/31474289-371d-4198-9566-74cc3413a02e" />

### Page web Ecommerce

<img width="1894" height="1031" alt="ecommecer" src="https://github.com/user-attachments/assets/57ee8c83-9c69-4a5e-b84d-d54146b3f983" />

### 5. Finalisation et Initialisation du Projet
Une fois les conteneurs démarrés, nous procédons à la migration du schéma de base de données vers RDS et à la création de l'administrateur.

    docker exec -it rocket_django python manage.py migrate
    docker exec -it rocket_django python manage.py collectstatic --no-input
    docker exec -it rocket_django python manage.py createsuperuser


# Installation de WordPress sur l'Instance 
### 1. Préparation de l'Instance

installe les outils Docker.

    sudo apt update && sudo apt install docker.io docker-compose-v2 -y
    sudo systemctl enable --now docker

### 2. Création du fichier docker-compose.yml

Puisque WordPress est seul sur cette instance, nous pouvons utiliser le port 80 standard.

    mkdir ~/wordpress && cd ~/wordpress
    nano docker-compose.yml
<img width="1091" height="305" alt="dcokercomposewordpress" src="https://github.com/user-attachments/assets/bf38b4e3-1eb7-456c-8859-188ac8c5ce1f" />

### 3. Lancement et vérification

    docker compose up -d

Page web de wordpress 
<img width="1917" height="969" alt="Capture d&#39;écran 2026-02-04 123023" src="https://github.com/user-attachments/assets/994b6c80-b9dc-4615-8807-eef0851d00b9" />

# Procédure de Sauvegarde de la Base de Données (RDS vers S3)

Conformément aux consignes du projet, nous avons mis en place un processus de sauvegarde exportant les données de l'instance Amazon RDS vers un bucket Amazon S3.

### 1. Extraction des données (Dump SQL)

    mysqldump -h database-1-bfhk.ctykc0gcmtio.us-east-2.rds.amazonaws.com \
      -u admin -p \
      --single-transaction \
      --set-gtid-purged=OFF \
      Database1BFHK > backup.sql

### 2. Configuration de l'accès AWS CLI

    aws configure
 
### 3. Transfert vers le stockage dédié (S3)

    aws s3 cp backup.sql s3://bucket1-bfhk/DB1/

<img width="1907" height="685" alt="S3" src="https://github.com/user-attachments/assets/34bfb4e6-5bed-4e85-a9a9-9d4cca962127" />

# Partie 2 : Isolation et Connectivité Inter-Équipes

L'objectif est de scinder l'infrastructure pour accueillir deux nouvelles équipes tout en maintenant une isolation stricte.

### 1. Création des VPC dédiés

    aws ec2 create-vpc --cidr-block 10.2.0.0/16 --tag-specifications 'ResourceType=vpc,Tags=[{Key=Name,Value=VPC-IA-BFHK}]'
    aws ec2 create-vpc --cidr-block 10.3.0.0/16 --tag-specifications 'ResourceType=vpc,Tags=[{Key=Name,Value=VPC-Cyber-BFHK}]'

### 2. Connectivité Inter-VPC (VPC Peering)

Pour permettre à l'équipe Cybersécurité d'accéder aux environnements IA et Web (MVP), nous avons établi des connexions de peering bidirectionnelles.

### 1. Établissement des connexions

    PEERING_MVP_ID=$(aws ec2 create-vpc-peering-connection \
        --vpc-id $CYBER_ID \
        --peer-vpc-id $MVP_ID \
        --tag-specifications 'ResourceType=vpc-peering-connection,Tags=[{Key=Name,Value=Peering-Cyber-MVP-BFHK}]' \
        --query 'VpcPeeringConnection.VpcPeeringConnectionId' --output text)

    aws ec2 accept-vpc-peering-connection --vpc-peering-connection-id $PEERING_MVP_ID

    PEERING_IA_ID=$(aws ec2 create-vpc-peering-connection \
        --vpc-id $CYBER_ID \
        --peer-vpc-id $IA_ID \
        --tag-specifications 'ResourceType=vpc-peering-connection,Tags=[{Key=Name,Value=Peering-Cyber-IA-BFHK}]' \
        --query 'VpcPeeringConnection.VpcPeeringConnectionId' --output text)

    aws ec2 accept-vpc-peering-connection --vpc-peering-connection-id $PEERING_IA_ID

### 2. Mise à jour des Tables de Routage (Étape cruciale)

Pour que la communication fonctionne, nous devons ajouter des routes vers les CIDR adverses.

    aws ec2 create-route --route-table-id $RT_CYBER_ID --destination-cidr-block 10.1.0.0/16 --vpc-peering-connection-id $PEERING_MVP_ID
    aws ec2 create-route --route-table-id $RT_MVP_ID --destination-cidr-block 10.3.0.0/16 --vpc-peering-connection-id $PEERING_MVP_ID

# 3. Création des Internet Gateways (IGW)

 Chaque VPC a besoin de sa propre porte de sortie vers Internet.
 
    IGW_IA_ID=$(aws ec2 create-internet-gateway --query 'InternetGateway.InternetGatewayId' --output text)
    aws ec2 attach-internet-gateway --vpc-id $IA_ID --internet-gateway-id $IGW_IA_ID

    IGW_CYBER_ID=$(aws ec2 create-internet-gateway --query 'InternetGateway.InternetGatewayId' --output text)
    aws ec2 attach-internet-gateway --vpc-id $CYBER_ID --internet-gateway-id $IGW_CYBER_ID

# 4. Mise en place de la NAT Gateway

Une NAT Gateway (Network Address Translation) permet aux instances d'un subnet privé d'initier du trafic vers internet (pour télécharger des packages, par exemple) tout en empêchant internet d'initier une connexion directe vers elles.

### 1. Allouer une IP Élastique (EIP)

    aws ec2 allocate-address --domain vpc --query 'AllocationId' --output text

### 2. Créer la NAT Gateway

    aws ec2 create-nat-gateway \
        --subnet-id subnet-0ffa006ce9b02033d \
        --allocation-id eipalloc-0b6342740726e0001 \
        --query 'NatGateway.NatGatewayId' --output text

    aws ec2 create-nat-gateway \
        --subnet-id subnet-0ab919f86f672063b \
        --allocation-id eipalloc-06b4437d4faaec0fd \
        --query 'NatGateway.NatGatewayId' --output text

### 3. Configurer la Table de Routage Privée

Table Publique IA (RouteIAPub-BFHK)
    
    aws ec2 create-route-table --vpc-id vpc-08c58a7ebc10f4aeb --tag-specifications 'ResourceType=route-table,Tags=[{Key=Name,Value=RouteIAPub-BFHK}]'
    aws ec2 create-route --route-table-id rtb-0c109bc902995016f --destination-cidr-block 0.0.0.0/0 --gateway-id igw-00dd9a81f2a3288de
    aws ec2 associate-route-table --route-table-id rtb-0c109bc902995016f --subnet-id subnet-0ab919f86f672063b
    
Table Privée IA (RouteIAPriv-BFHK)

    aws ec2 create-route-table --vpc-id vpc-08c58a7ebc10f4aeb --tag-specifications 'ResourceType=route-table,Tags=[{Key=Name,Value=RouteIAPriv-BFHK}]'
    aws ec2 create-route --route-table-id rtb-0c109bc902995016f --destination-cidr-block 0.0.0.0/0 --gateway-id nat-02662faf7b9c89505
    aws ec2 associate-route-table --route-table-id rtb-00d5cac12a3f620b6 --subnet-id subnet-00208f3848dcef8e8


Table Publique Cyber (RouteCyberPub-BFHK)

    aws ec2 create-route-table --vpc-id vpc-08babe0c09d5cad59 --tag-specifications 'ResourceType=route-table,Tags=[{Key=Name,Value=RouteCyberPub-BFHK}]'
    aws ec2 create-route --route-table-id rtb-091ef9535e0787457 --destination-cidr-block 0.0.0.0/0 --gateway-id igw-0c482a7fad8b142d5
    aws ec2 associate-route-table --route-table-id rtb-091ef9535e0787457 --subnet-id subnet-0ffa006ce9b02033d

Table Privée Cyber (RouteCyberPriv-BFHK)

    aws ec2 create-route-table --vpc-id vpc-08babe0c09d5cad59 --tag-specifications 'ResourceType=route-table,Tags=[{Key=Name,Value=RouteCyberPriv-BFHK}]'
    aws ec2 create-route --route-table-id rtb-08ca6dd77278a4253 --destination-cidr-block 0.0.0.0/0 --gateway-id nat-03f168f72156ff941
    aws ec2 associate-route-table --route-table-id rtb-08ca6dd77278a4253 --subnet-id subnet-079a89320c4bd6691

### 4. Instance EC2 (Gitlab & Uptime)

### Déploiement d'une instance GitLab dans le VPC Cyber pour centraliser le code et automatiser les tests de sécurité.

    aws ec2 run-instances \
        --image-id ami-06e3c045d79fd65d9 \
        --instance-type t3.large \
        --subnet-id subnet-0ffa006ce9b02033d \
        --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=VMCyber-Gitlab-BFHK}]'

### 1. Procédure d'installation

    sudo apt-get update

    sudo EXTERNAL_URL="http://18.217.232.200/" apt-get install gitlab-ce -y

### 2. Configuration et Initialisation
Une fois les paquets installés, GitLab doit configurer ses nombreux services internes.

    sudo gitlab-ctl reconfigure

### 3. Premier accès et Sécurité
Lors de la première connexion, un mot de passe administrateur (root) est généré de manière temporaire.

    sudo cat /etc/gitlab/initial_root_password

### 4. Accès à l'interface 

<img width="2554" height="1349" alt="Gtilab" src="https://github.com/user-attachments/assets/fbec89d7-71b0-4a5d-ab8e-5a7b74830457" />

### Déploiement d'une instance pour le monitoring

    aws ec2 run-instances \
        --image-id ami-06e3c045d79fd65d9 \
        --instance-type t3.micro \
        --subnet-id subnet-079a89320c4bd6691 \
        --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=VMCyber-Monitoring-BFHK}]'

Pour garantir la disponibilité de nos services (Ecommerce, WordPress, GitLab), nous avons déployé Uptime Kuma. Cet outil permet de surveiller en temps réel l'état des services et d'alerter en cas de coupure.

### 1. Procédure d'installation (Docker)

    sudo docker volume create uptime-kuma

    sudo docker run -d \
      --restart always \
      -p 3001:3001 \
      -v uptime-kuma:/app/data \
      --name uptime-kuma \
      louislam/uptime-kuma:1

 ### 2. Accès à l'interface 

 <img width="1907" height="906" alt="uptime" src="https://github.com/user-attachments/assets/2fd49160-6e15-4da4-91f2-9cda167f903c" />

