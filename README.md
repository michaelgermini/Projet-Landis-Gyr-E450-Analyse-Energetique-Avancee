# 📊 Projet Landis+Gyr E450 - Analyse Énergétique Avancée

[![Python](https://img.shields.io/badge/Python-3.8+-blue.svg)](https://www.python.org/)
[![Flask](https://img.shields.io/badge/Flask-2.0+-lightgrey.svg)](https://flask.palletsprojects.com/)
[![License](https://img.shields.io/badge/License-MIT-green.svg)](LICENSE)

> **Vision** : Transformer votre compteur électrique Landis+Gyr E450 en un centre de contrôle énergétique intelligent avec analyse prédictive et visualisation temps réel.

## 🧭 Vue d'ensemble du projet

Ce projet complet vous guide à travers la compréhension, l'exploitation et l'extension des capacités du compteur Landis+Gyr E450. De la lecture simple des données à l'analyse prédictive avancée, vous apprendrez à créer une plateforme complète de monitoring énergétique.

### 🎯 Objectifs pédagogiques

- **Comprendre** l'architecture DLMS/COSEM et les protocoles de communication
- **Maîtriser** la lecture et l'acquisition de données via port optique et M-Bus
- **Développer** une application web moderne avec Flask
- **Intégrer** des technologies IoT (MQTT, Home Assistant, InfluxDB)
- **Analyser** les données énergétiques avec des algorithmes avancés
- **Prévoir** la consommation future avec l'IA

### 🏗️ Architecture technique

```
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   E450 Meter    │────│   Flask App     │────│   Home Assistant│
│   (DLMS/COSEM)  │    │   (Python)      │    │   (MQTT)        │
└─────────────────┘    └─────────────────┘    └─────────────────┘
         │                       │                       │
         └───────────────────────┼───────────────────────┘
                                 │
                    ┌─────────────────┐
                    │   InfluxDB      │
                    │   + Grafana     │
                    │   (Time Series) │
                    └─────────────────┘
```

## 📚 Structure du livre

### 🧭 Partie I — Introduction & Fondations
- **Chapitre 1** : Présentation du projet et objectifs
- **Chapitre 2** : Vision et aperçu de l'application Flask
- **Chapitre 3** : Introduction au compteur Landis+Gyr E450
- **Chapitre 4** : Caractéristiques électriques et mécaniques
- **Chapitre 5** : Architecture et protocoles de communication

### ⚙️ Partie II — Lecture et acquisition des données
- **Chapitre 6** : Lecture locale via port optique USB
- **Chapitre 7** : Lecture par bus M-Bus
- **Chapitre 8** : Structure des données OBIS
- **Chapitre 9** : Stockage et persistance des données

### 💻 Partie III — Développement de l'application Flask
- **Chapitre 10** : Installation et configuration
- **Chapitre 11** : Structure du projet Flask
- **Chapitre 12** : Conception de l'interface utilisateur
- **Chapitre 13** : Authentification et sécurité

### 🔌 Partie IV — Connectivité & intégration intelligente
- **Chapitre 14** : Lecture réelle du compteur
- **Chapitre 15** : Envoi vers Home Assistant (MQTT)
- **Chapitre 16** : Webhook vers InfluxDB + Grafana
- **Chapitre 17** : Calcul du coût énergétique
- **Chapitre 18** : Détection automatique d'anomalies

### 📊 Partie V — Analyse & visualisation avancée
- **Chapitre 19** : Tendances et statistiques
- **Chapitre 20** : Prévision énergétique avec Prophet
- **Chapitre 21** : Comparaison multi-sources
- **Chapitre 22** : Analyse réseau vs autoproduction

### 🏠 Partie VI — Cas d'usage et exemples pratiques
- **Chapitre 23** : Appartement individuel
- **Chapitre 24** : Atelier industriel
- **Chapitre 25** : Maison solaire
- **Chapitre 26** : Immeuble collectif

### 🧩 Partie VII — Pédagogie & références
- **Chapitre 27** : Encadrés pédagogiques
- **Chapitre 28** : Glossaire illustré
- **Chapitre 29** : Annexes techniques
- **Chapitre 30** : Bibliographie & ressources

### 🧠 Partie VIII — Annexes développeur
- **Chapitre 31** : API et documentation technique
- **Chapitre 32** : Packaging et déploiement
- **Chapitre 33** : Service systemd pour Raspberry Pi
- **Chapitre 34** : Extensions futures

## 🚀 Démarrage rapide

### Prérequis

```bash
# Python 3.8+
python --version

# Git pour cloner le repository
git --version

# Adaptateur USB série (FTDI) pour port optique
# Compteur Landis+Gyr E450 avec accès port optique
```

### Installation

```bash
# Cloner le repository
git clone https://github.com/votre-repo/compteur-e450.git
cd compteur-e450

# Créer un environnement virtuel
python -m venv venv
source venv/bin/activate  # Windows: venv\Scripts\activate

# Installer les dépendances
pip install -r requirements.txt

# Configuration initiale
cp config.example.py config.py
# Éditer config.py avec vos paramètres

# Lancer l'application
python app.py
```

### Configuration matérielle

1. **Port optique USB** :
   - Connecter l'adaptateur FTDI au port optique du compteur
   - Vérifier le port COM avec `python -c "import serial.tools.list_ports; print([p.device for p in serial.tools.list_ports.comports()])"`

2. **Bus M-Bus** (optionnel) :
   - Câblage RS485 pour installations collectives
   - Configuration de l'adresse secondaire du compteur

## 📁 Structure des fichiers

```
compteur-e450/
├── README.md                    # Ce fichier
├── requirements.txt             # Dépendances Python
├── config.example.py           # Configuration exemple
├── app.py                      # Application Flask principale
├── models/                     # Modèles de données
├── routes/                     # Routes Flask
├── templates/                  # Templates HTML
├── static/                     # CSS, JS, images
├── scripts/                    # Scripts de lecture compteur
├── tests/                      # Tests unitaires
├── docs/                       # Documentation
├── code_source/                # Code source organisé
│   ├── lecteur_compteur/       # Module de lecture
│   ├── analyse_donnees/        # Module d'analyse
│   └── visualisation/          # Module de graphs
└── Partie_[I-VIII]_*/          # Chapitres du livre
```

## 🛠️ Technologies utilisées

### Backend
- **Python 3.8+** : Langage principal
- **Flask** : Framework web
- **SQLAlchemy** : ORM pour base de données
- **gurux-dlms** : Bibliothèque DLMS/COSEM
- **pymodbus** : Communication Modbus/M-Bus

### Frontend
- **HTML5/CSS3** : Structure et style
- **JavaScript ES6+** : Interactivité
- **Chart.js** : Graphiques dynamiques
- **Bootstrap/Material Design** : Framework UI

### Base de données & Temps réel
- **SQLite/PostgreSQL** : Stockage local
- **InfluxDB** : Time series database
- **MQTT** : Communication IoT
- **WebSocket** : Mise à jour temps réel

### IA & Analyse
- **Pandas/NumPy** : Traitement des données
- **Prophet** : Prévision énergétique
- **Scikit-learn** : Algorithmes de machine learning
- **TensorFlow Lite** : IA embarquée (futur)

## 📈 Fonctionnalités clés

### ✅ Implémentées
- [x] Lecture port optique USB
- [x] Parsing des codes OBIS
- [x] Application Flask de base
- [x] Dashboard responsive
- [x] Stockage SQLite
- [x] API REST

### 🚧 En développement
- [ ] Intégration MQTT complète
- [ ] Dashboard Grafana
- [ ] Algorithmes de détection d'anomalies
- [ ] Prévision avec Prophet

### 🔮 Planifiées
- [ ] IA embarquée sur Raspberry Pi
- [ ] Support Zigbee/LoRa
- [ ] Application mobile
- [ ] Cloud Azure/AWS

## 📖 Guide d'utilisation

### Lecture du compteur

```python
from lecteur_compteur import LecteurE450

# Configuration
config = {
    'port': 'COM3',  # ou '/dev/ttyUSB0' sous Linux
    'baudrate': 9600,
    'timeout': 5
}

# Lecture
lecteur = LecteurE450(config)
donnees = lecteur.lire_donnees()

print(f"Énergie active: {donnees['1.8.0']} kWh")
print(f"Puissance instantanée: {donnees['16.7.0']} W")
```

### API REST

```bash
# Récupérer les dernières données
curl http://localhost:5000/api/data

# Historique des 24 dernières heures
curl "http://localhost:5000/api/data?period=24h"

# Calcul des coûts
curl "http://localhost:5000/api/costs?tarif=0.15"
```

## 🤝 Contribution

Les contributions sont les bienvenues ! Voir [CONTRIBUTING.md](CONTRIBUTING.md) pour les guidelines.

### Types de contributions
- **🐛 Corrections de bugs**
- **✨ Nouvelles fonctionnalités**
- **📚 Amélioration de la documentation**
- **🧪 Tests supplémentaires**
- **🌐 Traductions**

## 📝 Licence

Ce projet est sous licence MIT - voir le fichier [LICENSE](LICENSE) pour plus de détails.

## 🙏 Remerciements

- **Landis+Gyr** pour la documentation technique du E450
- **Gurux** pour la bibliothèque DLMS open-source
- **Home Assistant** pour l'écosystème IoT
- **InfluxDB/Grafana** pour les outils de visualisation

## 📊 Dashboard

![Dashboard de monitoring énergétique](asset/dashboard.jpg)

*Interface de visualisation des données du compteur E450 avec graphiques temps réel et métriques énergétiques.*

---

> **💡 Astuce** : Commencez par la Partie I pour comprendre les fondamentaux, puis explorez les scripts dans `code_source/` pour des exemples pratiques.

*Dernière mise à jour : Novembre 2025*
