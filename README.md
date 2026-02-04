# Projet-E4-AWS

📖 Présentation du Projet

Ce dépôt contient l'ensemble des scripts de déploiement et la documentation technique pour la mise en place d'une infrastructure AWS. Le projet répond à trois besoins majeurs :

    MVP Ecommerce : Déploiement d'une application de vente avec paiement Stripe.

    Migration WordPress : Mise en place d'une stack WordPress pour test.

    Sécurité & Scalabilité : Isolation réseau (VPC), bases de données gérées (RDS) et haute disponibilité.

🚀 Partie 1 : Déploiement de l'Infrastructure de Base

Cette étape consiste à mettre en place le réseau, la base de données hautement disponible et les serveurs d'application pour le MVP Ecommerce et la stack WordPress.
1. Architecture Réseau (VPC & Segmentation)

Pour garantir la sécurité, nous isolons les serveurs web dans un sous-réseau public et la base de données dans un sous-réseau privé.

Création du VPC

    aws ec2 create-vpc --cidr-block 10.0.0.0/16 --query 'Vpc.VpcId' --output text
