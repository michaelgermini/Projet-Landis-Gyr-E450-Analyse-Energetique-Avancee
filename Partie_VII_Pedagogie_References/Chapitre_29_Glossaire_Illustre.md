# 📚 Chapitre 29 : Glossaire illustré

## 🔍 Termes techniques essentiels

### **DLMS/COSEM** (Device Language Message Specification / Companion Specification for Energy Metering)

```text
DLMS/COSEM est le protocole standard international pour la communication
avec les compteurs électriques intelligents. Il définit:

• La structure des messages (Application Layer)
• Les objets de données (COSEM objects)
• Les services d'accès (get, set, action)
• La sécurité et l'authentification
```

**Analogie** : Comme le langage diplomatique entre pays, DLMS/COSEM est le protocole officiel pour que les compteurs "parlent" aux systèmes informatiques.

### **OBIS** (Object Identification System)

```text
Système de codification hiérarchique pour identifier les données:
A.B.C.D.E.F

A = Médium (1=Électricité)
B = Canal (8=Total, 1-3=Phases)
C = Quantité (8=Énergie, 7=Puissance)
D = Type (1=Import, 2=Export)
E = Tarif (1=HP, 2=HC, 255=Total)
F = Stockage (255=Total, 0=Instantané)
```

**Exemple** : `1.8.0.255.2.255` = Énergie active totale exportée (tous tarifs confondus)

### **M-Bus** (Meter-Bus)

```text
Protocole de communication série pour réseaux de compteurs:
• Topologie: Bus RS485 half-duplex
• Distance: Jusqu'à 1000m
• Vitesse: 300-9600 bauds
• Adressage: 1-250 équipements
• Alimentation: Parasite via le bus (30-42V)
```

**Avantage** : Un câble pour données + alimentation = économie d'installation.

### **Port optique** (Optical port)

```text
Interface infrarouge normalisée (IEC 1107/62056-21):
• Technologie: LED IR + photodétecteur
• Portée: 0.1-1 mètre
• Alimentation: Passive
• Sécurité: Isolation galvanique
• Vitesse: Variable (300-9600 bauds)
```

**Usage** : Connexion directe sécurisée pour configuration et diagnostic.

## ⚡ Concepts électriques

### **Puissance active (W)**

```text
Énergie consommée réellement par les équipements:
P = U × I × cos(φ)

• Mesure l'énergie utile (chaleur, mouvement, lumière)
• Facturée par les fournisseurs
• Unité: Watt (W) ou kilowatt (kW)
```

**Exemple** : Une ampoule 100W consomme 100W de puissance active.

### **Puissance réactive (VAR)**

```text
Énergie stockée temporairement dans les champs magnétiques:
Q = U × I × sin(φ)

• Nécessaire au fonctionnement des moteurs et transformateurs
• Ne produit pas de travail utile
• Unité: Volt-ampère réactif (VAR)
```

**Impact** : Trop de puissance réactive surcharge le réseau.

### **Facteur de puissance (cos φ)**

```text
Rapport entre puissance active et puissance apparente:
cos φ = P / S

• Idéal: 1.0 (puissance purement active)
• Acceptable: > 0.9
• Critique: < 0.8 (pénalités possibles)
```

**Amélioration** : Batteries de compensation, correction des moteurs.

### **THD** (Total Harmonic Distortion)

```text
Distorsion harmonique totale - mesure de la qualité d'onde:
THD = √(∑(harmoniques²)) / fondamentale × 100%

• Normale: < 5%
• Acceptable: < 8%
• Critique: > 10%
```

**Causes** : Équipements électroniques non linéaires (ordinateurs, LED).

## 🏗️ Architecture logicielle

### **MVC** (Modèle-Vue-Contrôleur)

```text
Pattern architectural séparant les responsabilités:

Modèle (Model)      → Gestion des données (SQLAlchemy)
Vue (View)         → Interface utilisateur (Jinja2/HTML)
Contrôleur (Controller) → Logique métier (Flask routes)

Avantages:
• Maintenance facilitée
• Testabilité accrue
• Réutilisabilité du code
```

### **API REST**

```text
Interface de programmation basée sur HTTP:

GET    /api/data       → Récupération données
POST   /api/data       → Création données
PUT    /api/data/1     → Modification
DELETE /api/data/1     → Suppression

Principe REST:
• Stateless (sans état)
• Ressources identifiées par URL
• Actions via verbes HTTP
```

### **WebSocket**

```text
Communication bidirectionnelle temps réel:

Client ←→ Serveur (connexion persistante)
• Pas de polling répétitif
• Mise à jour instantanée
• Faible latence

Usage: Notifications temps réel, dashboards live
```

## 📊 Analyse de données

### **Séries temporelles** (Time Series)

```text
Données indexées par le temps:
• Timestamp + valeur(s)
• Fréquence: 1Hz à 1/jour
• Patterns: Saisonnier, tendance, bruit

Outils: InfluxDB, Grafana, Pandas
```

### **Analyse spectrale** (FFT)

```text
Décomposition fréquentielle du signal:
• Transformée de Fourier
• Identification des périodes dominantes
• Détection de patterns cachés

Application: Détection de consommations périodiques
```

### **Machine Learning**

```text
Algorithmes d'intelligence artificielle:

• Supervisé: Prédiction avec données d'entraînement
• Non supervisé: Détection d'anomalies
• Series temporelles: Prophet (Facebook/Meta)

Exemple: Prédiction de consommation, classification d'anomalies
```

## 🔐 Sécurité

### **Chiffrement AES**

```text
Standard de chiffrement symétrique:
• Clé 128/256 bits
• Sécurisé et rapide
• Utilisé pour les données sensibles
• Implémentation: cryptography (Python)
```

### **Authentification JWT**

```text
JSON Web Tokens:
• Stateless (pas de session serveur)
• Contient claims (droits, expiration)
• Signé numériquement
• Format: header.payload.signature
```

### **Rate Limiting**

```text
Protection contre les abus:
• Limite nombre de requêtes/minute
• Prévention des attaques DoS
• Algorithmes: Token bucket, Leaky bucket
• Outil: Flask-Limiter
```

## 🌐 IoT et connectivité

### **MQTT** (Message Queuing Telemetry Transport)

```text
Protocole IoT léger:
• Publish/Subscribe
• QoS (0,1,2) pour fiabilité
• Broker centralisé (Mosquitto)
• Faible bande passante
```

### **Home Assistant**

```text
Plateforme domotique:
• Auto-découverte MQTT
• Intégrations multiples
• Automatisations
• Interface utilisateur
```

### **InfluxDB**

```text
Base de données séries temporelles:
• Optimisée pour timestamps
• Requêtes Flux (similaire SQL)
• Haute performance
• Intégration Grafana native
```

---

**Navigation**
- [Chapitre précédent : Encadrés pédagogiques](Chapitre_28_Encadres_Pedagogiques.md)
- [Chapitre suivant : Annexes techniques](Chapitre_30_Annexes_Techniques.md)
- [Retour à la table des matières](../../README.md)
