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

    aws s3 mb s3://backups-client-ecommerce-2026 --region eu-west-1


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
    <img width="1219" height="543" alt="image" src="https://github.com/user-attachments/assets/7ed7a64f-7fbf-474f-942a-dc37a445f736" />

### 4. Déploiement via Docker & Docker Compose

    sudo apt install docker.io docker-compose-v2 -y
    sudo systemctl enable --now docker

    sudo usermod -aG docker $USER && newgrp docker

    docker compose up -d --build

### 5. Finalisation et Initialisation du Projet
Une fois les conteneurs démarrés, nous procédons à la migration du schéma de base de données vers RDS et à la création de l'administrateur.

    docker exec -it rocket_django python manage.py migrate
    docker exec -it rocket_django python manage.py collectstatic --no-input
    docker exec -it rocket_django python manage.py createsuperuser




