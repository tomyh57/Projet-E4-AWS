# Projet-E4-AWS

## 📖 Présentation du Projet

Ce dépôt contient l'ensemble des scripts de déploiement et la documentation technique pour la mise en place d'une infrastructure AWS. Le projet répond à trois besoins majeurs :

    MVP Ecommerce : Déploiement d'une application de vente avec paiement Stripe.

    Migration WordPress : Mise en place d'une stack WordPress pour test.

    Sécurité & Scalabilité : Isolation réseau (VPC), bases de données gérées (RDS) et haute disponibilité.

## 🚀 Partie 1 : Déploiement de l'Infrastructure de Base

Cette étape consiste à mettre en place le réseau, la base de données hautement disponible et les serveurs d'application pour le MVP Ecommerce et la stack WordPress.

### 1. Architecture Réseau (VPC & Segmentation)

Pour garantir la sécurité, nous isolons les serveurs web dans un sous-réseau public et la base de données dans un sous-réseau privé.

Création du VPC

    aws ec2 create-vpc --cidr-block 10.0.0.0/16 --query 'Vpc.VpcId' --output text

Création des Sous-réseaux (Subnets)

    aws ec2 create-subnet --vpc-id vpc-ID --cidr-block 10.0.1.0/24 --availability-zone eu-west-1a
    aws ec2 create-subnet --vpc-id vpc-ID --cidr-block 10.0.2.0/24 --availability-zone eu-west-1b

Configuration de l'accès Internet

    aws ec2 create-internet-gateway --query 'InternetGateway.InternetGatewayId' --output text
    aws ec2 attach-internet-gateway --vpc-id vpc-ID --internet-gateway-id igw-ID
    aws ec2 create-route --route-table-id rtb-ID --destination-cidr-block 0.0.0.0/0 --gateway-id igw-ID

### 2. Base de Données Managée (Amazon RDS)

Le client exige une haute disponibilité et moins de gestion possible. Nous utilisons RDS avec l'option Multi-AZ.

    aws rds create-db-subnet-group \
        --db-subnet-group-name rds-private-group \
        --db-subnet-group-description "Subnets pour DB privée" \
        --subnet-ids subnet-ID-A subnet-ID-B
        
    aws rds create-db-instance \
        --db-instance-identifier ecommerce-db \
        --db-instance-class db.t3.micro \
        --engine mariadb \
        --allocated-storage 20 \
        --db-subnet-group-name rds-private-group \
        --multi-az \
        --master-username admin \
        --master-user-password VOTRE_MOT_DE_PASSE_SECURISE

### 3. Intance (EC2)

Déploiement des deux POC : l'application Ecommerce (Stripe) et WordPress.

    aws ec2 run-instances \
        --image-id ami-0c55b159cbfafe1f0 \
        --count 1 \
        --instance-type t3.micro \
        --key-name MaCleSsh \
        --subnet-id subnet-public-ID \
        --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=SVR-WEB-DEV}]'

### 4. Stratégie de Sauvegarde et Stockage (S3)

Pour répondre à la consigne de sauvegarde régulière des applications et bases de données.

    aws s3 mb s3://backups-client-ecommerce-2026 --region eu-west-1

