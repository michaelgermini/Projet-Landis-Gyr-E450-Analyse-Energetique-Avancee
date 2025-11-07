# 💻 Chapitre 10 : Installation et configuration de Flask

## 🛠️ Prérequis système

Avant de commencer le développement de l'application Flask, assurons-nous que votre environnement est correctement configuré.

### Python 3.8+

#### Vérification de la version

```bash
# Windows
python --version
# Ou si python3 est installé
python3 --version

# Linux/Mac
python3 --version

# Sortie attendue : Python 3.8.0 ou supérieur
```

#### Installation si nécessaire

**Windows :**
```bash
# Téléchargement depuis python.org
# Installation standard
# Ajout de Python au PATH pendant l'installation
```

**Linux (Ubuntu/Debian) :**
```bash
# Mise à jour des paquets
sudo apt update

# Installation de Python 3.8+
sudo apt install python3.8 python3.8-venv python3-pip

# Vérification
python3 --version
```

**macOS :**
```bash
# Via Homebrew
brew install python@3.8

# Ou téléchargement depuis python.org
```

### Outils de développement

#### Git

```bash
# Vérification
git --version

# Installation si nécessaire
# Windows : https://git-scm.com/download/win
# Linux : sudo apt install git
# macOS : brew install git
```

#### Éditeur de code

**Recommandations :**
- **VS Code** : Extension Python, intégration Git
- **PyCharm Community** : IDE complet gratuit
- **Sublime Text** : Léger et rapide

## 📦 Structure du projet Flask

### Arborescence recommandée

```
compteur-e450/
├── app.py                      # Application principale
├── config.py                   # Configuration
├── requirements.txt            # Dépendances
├── .env                        # Variables d'environnement (gitignoré)
├── .gitignore                  # Fichiers à ignorer
├── README.md                   # Documentation
│
├── app/                        # Package principal
│   ├── __init__.py
│   ├── models.py               # Modèles de données
│   ├── routes.py               # Routes et contrôleurs
│   ├── forms.py                # Formulaires WTForms
│   ├── utils.py                # Utilitaires
│   └── services/               # Services métier
│       ├── __init__.py
│       ├── lecteur_compteur.py
│       ├── calculateur_energie.py
│       └── detecteur_anomalies.py
│
├── templates/                  # Templates HTML
│   ├── base.html
│   ├── dashboard.html
│   ├── historique.html
│   ├── analyse.html
│   └── parametres.html
│
├── static/                     # Ressources statiques
│   ├── css/
│   │   ├── style.css
│   │   └── themes/
│   ├── js/
│   │   ├── app.js
│   │   └── charts.js
│   └── img/
│       └── logo.png
│
├── migrations/                 # Migrations base de données
├── tests/                      # Tests unitaires
│   ├── __init__.py
│   ├── test_models.py
│   ├── test_routes.py
│   └── test_services.py
│
├── logs/                       # Logs de l'application
├── data/                       # Données locales (SQLite, JSON)
└── scripts/                    # Scripts utilitaires
    ├── init_db.py
    ├── import_data.py
    └── backup.py
```

### Création de la structure

```bash
# Création des répertoires
mkdir -p app/services templates static/css/themes static/js static/img
mkdir -p migrations tests logs data scripts

# Création des fichiers de base
touch app/__init__.py app/models.py app/routes.py app/forms.py app/utils.py
touch app/services/__init__.py
touch templates/base.html
touch static/css/style.css static/js/app.js
touch config.py .env requirements.txt
touch scripts/init_db.py

# Rendre les scripts exécutables (Linux/Mac)
chmod +x scripts/*.py
```

## 📋 Dépendances Python

### Fichier requirements.txt

```txt
# Framework web
Flask==2.3.3
Flask-WTF==1.1.1
Flask-SQLAlchemy==3.0.5
Flask-Migrate==4.0.5
Flask-Login==0.6.3

# Base de données
SQLAlchemy==2.0.23

# Communication compteur
pyserial==3.5
gurux-dlms==1.0.0
pymbus==1.0.0

# Traitement des données
pandas==2.1.3
numpy==1.24.3
matplotlib==3.8.0

# Visualisation
plotly==5.17.0
chart.js==4.4.0

# Intelligence artificielle
scikit-learn==1.3.2
prophet==1.1.5

# IoT et communication
paho-mqtt==1.6.1
requests==2.31.0

# Utilitaires
python-dotenv==1.0.0
schedule==1.2.1
apscheduler==3.10.4

# Développement et tests
pytest==7.4.3
pytest-flask==1.2.0
coverage==7.3.2
black==23.11.0
flake8==6.1.0

# Production
gunicorn==21.2.0
python-decouple==3.8
```

### Installation des dépendances

```bash
# Création d'un environnement virtuel
python -m venv venv

# Activation
# Windows
venv\Scripts\activate
# Linux/Mac
source venv/bin/activate

# Mise à jour de pip
pip install --upgrade pip

# Installation des dépendances
pip install -r requirements.txt

# Vérification
pip list
```

## ⚙️ Configuration de l'application

### Fichier config.py

```python
import os
from datetime import timedelta

class Config:
    """Configuration de base"""

    # Sécurité
    SECRET_KEY = os.environ.get('SECRET_KEY') or 'dev-secret-key-change-in-production'

    # Base de données
    SQLALCHEMY_DATABASE_URI = os.environ.get('DATABASE_URL') or 'sqlite:///data/compteur_e450.db'
    SQLALCHEMY_TRACK_MODIFICATIONS = False

    # Session
    PERMANENT_SESSION_LIFETIME = timedelta(days=7)

    # Upload
    MAX_CONTENT_LENGTH = 16 * 1024 * 1024  # 16MB max file size

    # Compteur
    COMPTEUR_MODEL = 'E450'
    COMPTEUR_ID = os.environ.get('COMPTEUR_ID') or 'E450001234'

    # Communication
    PORT_OPTICAL = os.environ.get('PORT_OPTICAL') or 'COM3'  # ou /dev/ttyUSB0
    BAUDRATE_OPTICAL = 9600

    # M-Bus
    MBUS_PORT = os.environ.get('MBUS_PORT') or '/dev/ttyUSB1'
    MBUS_BAUDRATE = 2400
    MBUS_ADDRESSE_COMPTEUR = int(os.environ.get('MBUS_ADDRESS') or '1')

    # MQTT (Home Assistant)
    MQTT_BROKER = os.environ.get('MQTT_BROKER') or 'localhost'
    MQTT_PORT = int(os.environ.get('MQTT_PORT') or '1883')
    MQTT_USERNAME = os.environ.get('MQTT_USERNAME')
    MQTT_PASSWORD = os.environ.get('MQTT_PASSWORD')
    MQTT_TOPIC_BASE = f"homeassistant/sensor/compteur_{COMPTEUR_ID}"

    # InfluxDB
    INFLUXDB_URL = os.environ.get('INFLUXDB_URL') or 'http://localhost:8086'
    INFLUXDB_TOKEN = os.environ.get('INFLUXDB_TOKEN')
    INFLUXDB_ORG = os.environ.get('INFLUXDB_ORG') or 'e450'
    INFLUXDB_BUCKET = os.environ.get('INFLUXDB_BUCKET') or 'energie'

    # Application
    DEBUG = os.environ.get('FLASK_DEBUG') == 'True'
    TESTING = os.environ.get('FLASK_TESTING') == 'True'

    # Planification
    LECTURE_INTERVALLE = int(os.environ.get('LECTURE_INTERVALLE') or '300')  # 5 minutes
    NETTOYAGE_ANCIEN = int(os.environ.get('NETTOYAGE_ANCIEN') or '365')  # jours

    # Seuils d'alerte
    SEUIL_PUISSANCE_MAX = float(os.environ.get('SEUIL_PUISSANCE_MAX') or '5000')  # W
    SEUIL_TENSION_MIN = float(os.environ.get('SEUIL_TENSION_MIN') or '200')  # V
    SEUIL_TENSION_MAX = float(os.environ.get('SEUIL_TENSION_MAX') or '250')  # V

    # Interface web
    THEME_DEFAUT = 'neon'
    LANGUE_DEFAUT = 'fr'

class DevelopmentConfig(Config):
    """Configuration développement"""
    DEBUG = True
    SQLALCHEMY_ECHO = True

class TestingConfig(Config):
    """Configuration tests"""
    TESTING = True
    SQLALCHEMY_DATABASE_URI = 'sqlite:///:memory:'
    WTF_CSRF_ENABLED = False

class ProductionConfig(Config):
    """Configuration production"""
    DEBUG = False
    TESTING = False

    # Sécurité renforcée
    SESSION_COOKIE_SECURE = True
    SESSION_COOKIE_HTTPONLY = True
    REMEMBER_COOKIE_SECURE = True
    REMEMBER_COOKIE_HTTPONLY = True

# Sélection de la configuration
config = {
    'development': DevelopmentConfig,
    'testing': TestingConfig,
    'production': ProductionConfig,
    'default': DevelopmentConfig
}

def get_config(config_name=None):
    """Récupération de la configuration active"""
    if config_name is None:
        config_name = os.environ.get('FLASK_ENV') or 'development'

    return config.get(config_name, config['default'])
```

### Variables d'environnement (.env)

```bash
# Sécurité
SECRET_KEY=votre-cle-secrete-très-longue-et-complexe

# Base de données
DATABASE_URL=sqlite:///data/compteur_e450.db

# Compteur
COMPTEUR_ID=E450001234

# Communication
PORT_OPTICAL=COM3
MBUS_PORT=COM4
MBUS_ADDRESS=1

# MQTT
MQTT_BROKER=192.168.1.100
MQTT_PORT=1883
MQTT_USERNAME=compteur
MQTT_PASSWORD=mon_mot_de_passe

# InfluxDB
INFLUXDB_URL=http://localhost:8086
INFLUXDB_TOKEN=mon_token_influxdb
INFLUXDB_ORG=e450
INFLUXDB_BUCKET=energie

# Application
FLASK_ENV=development
FLASK_DEBUG=True

# Seuils
SEUIL_PUISSANCE_MAX=5000
SEUIL_TENSION_MIN=200
SEUIL_TENSION_MAX=250
```

### Chargement de la configuration

```python
# Dans app/__init__.py
from flask import Flask
from config import get_config

def create_app(config_name=None):
    """Factory function pour créer l'application Flask"""

    app = Flask(__name__)

    # Chargement de la configuration
    app.config.from_object(get_config(config_name))

    # Chargement des variables d'environnement si .env existe
    try:
        from dotenv import load_dotenv
        load_dotenv()
        app.config.from_envvar('FLASK_ENV_FILE', silent=True)
    except ImportError:
        pass  # python-dotenv non installé

    # Initialisation des extensions
    from app.extensions import db, migrate, login_manager
    db.init_app(app)
    migrate.init_app(app, db)
    login_manager.init_app(app)

    # Configuration du login
    login_manager.login_view = 'auth.login'
    login_manager.login_message = 'Veuillez vous connecter pour accéder à cette page.'

    # Enregistrement des blueprints
    from app.routes import main_bp, api_bp, auth_bp
    app.register_blueprint(main_bp)
    app.register_blueprint(api_bp, url_prefix='/api')
    app.register_blueprint(auth_bp, url_prefix='/auth')

    # Gestionnaire d'erreurs
    @app.errorhandler(404)
    def page_not_found(e):
        return render_template('404.html'), 404

    @app.errorhandler(500)
    def internal_error(e):
        return render_template('500.html'), 500

    # Contexte global pour les templates
    @app.context_processor
    def inject_global_vars():
        return {
            'now': datetime.utcnow(),
            'config': app.config
        }

    return app
```

## 🗄️ Initialisation de la base de données

### Modèles SQLAlchemy

```python
# app/models.py
from app.extensions import db
from flask_login import UserMixin
from werkzeug.security import generate_password_hash, check_password_hash
from datetime import datetime

class User(UserMixin, db.Model):
    """Modèle utilisateur"""
    __tablename__ = 'users'

    id = db.Column(db.Integer, primary_key=True)
    username = db.Column(db.String(64), unique=True, nullable=False)
    email = db.Column(db.String(120), unique=True, nullable=False)
    password_hash = db.Column(db.String(128))
    is_admin = db.Column(db.Boolean, default=False)
    created_at = db.Column(db.DateTime, default=datetime.utcnow)
    last_login = db.Column(db.DateTime)

    def set_password(self, password):
        self.password_hash = generate_password_hash(password)

    def check_password(self, password):
        return check_password_hash(self.password_hash, password)

class MesureEnergie(db.Model):
    """Modèle pour les mesures énergétiques"""
    __tablename__ = 'mesures_energie'

    id = db.Column(db.Integer, primary_key=True)
    timestamp = db.Column(db.DateTime, default=datetime.utcnow, index=True)

    # Énergies cumulées
    energie_active_import = db.Column(db.Float)
    energie_active_export = db.Column(db.Float)
    energie_reactive_import = db.Column(db.Float)
    energie_reactive_export = db.Column(db.Float)

    # Puissances instantanées
    puissance_active_totale = db.Column(db.Float)
    puissance_reactive_totale = db.Column(db.Float)
    puissance_apparente_totale = db.Column(db.Float)

    # Tensions
    tension_l1 = db.Column(db.Float)
    tension_l2 = db.Column(db.Float)
    tension_l3 = db.Column(db.Float)

    # Courants
    courant_l1 = db.Column(db.Float)
    courant_l2 = db.Column(db.Float)
    courant_l3 = db.Column(db.Float)
    courant_neutre = db.Column(db.Float)

    # Qualité réseau
    frequence = db.Column(db.Float)
    facteur_puissance = db.Column(db.Float)
    thd_tension = db.Column(db.Float)
    thd_courant = db.Column(db.Float)

    # Métadonnées
    source = db.Column(db.String(20), default='e450')
    qualite = db.Column(db.String(10), default='good')

class EvenementCompteur(db.Model):
    """Modèle pour les événements"""
    __tablename__ = 'evenements_compteur'

    id = db.Column(db.Integer, primary_key=True)
    timestamp = db.Column(db.DateTime, default=datetime.utcnow, index=True)

    type_evenement = db.Column(db.String(50), nullable=False)
    severite = db.Column(db.String(20), default='info')
    description = db.Column(db.Text)

    valeur_mesuree = db.Column(db.Float)
    unite = db.Column(db.String(10))
    code_obis = db.Column(db.String(20))
    seuil_declencheur = db.Column(db.Float)

class Configuration(db.Model):
    """Modèle pour la configuration persistante"""
    __tablename__ = 'configuration'

    id = db.Column(db.Integer, primary_key=True)
    parametre = db.Column(db.String(100), unique=True, nullable=False)
    valeur = db.Column(db.String(255))
    type_valeur = db.Column(db.String(20), default='string')
    description = db.Column(db.Text)
    updated_at = db.Column(db.DateTime, default=datetime.utcnow, onupdate=datetime.utcnow)
```

### Extensions Flask

```python
# app/extensions.py
from flask_sqlalchemy import SQLAlchemy
from flask_migrate import Migrate
from flask_login import LoginManager

# Initialisation des extensions
db = SQLAlchemy()
migrate = Migrate()
login_manager = LoginManager()

@login_manager.user_loader
def load_user(user_id):
    from app.models import User
    return User.query.get(int(user_id))
```

### Script d'initialisation

```python
# scripts/init_db.py
#!/usr/bin/env python3
"""
Script d'initialisation de la base de données
"""

import sys
import os

# Ajout du répertoire parent au path
sys.path.insert(0, os.path.dirname(os.path.dirname(os.path.abspath(__file__))))

from app import create_app
from app.extensions import db
from app.models import User

def init_database():
    """Initialisation complète de la base de données"""

    app = create_app('development')

    with app.app_context():
        # Création de toutes les tables
        print("Création des tables...")
        db.create_all()

        # Création d'un utilisateur administrateur par défaut
        if not User.query.filter_by(username='admin').first():
            admin = User(
                username='admin',
                email='admin@compteur-e450.local',
                is_admin=True
            )
            admin.set_password('admin123')

            db.session.add(admin)
            db.session.commit()

            print("Utilisateur admin créé : admin / admin123")
        else:
            print("Utilisateur admin existe déjà")

        # Vérification des tables
        inspector = db.inspect(db.engine)
        tables = inspector.get_table_names()

        print(f"Tables créées : {', '.join(tables)}")

        print("Base de données initialisée avec succès !")

if __name__ == "__main__":
    init_database()
```

## 🚀 Application principale

### Fichier app.py

```python
# app.py
from app import create_app

app = create_app()

if __name__ == '__main__':
    app.run(
        host='0.0.0.0',
        port=5000,
        debug=app.config['DEBUG']
    )
```

### Blueprint de base

```python
# app/routes.py
from flask import Blueprint, render_template, redirect, url_for, flash
from flask_login import login_required, current_user
from app.models import MesureEnergie, EvenementCompteur
from datetime import datetime, timedelta

main_bp = Blueprint('main', __name__)

@main_bp.route('/')
@login_required
def index():
    """Page d'accueil - redirection vers le dashboard"""
    return redirect(url_for('main.dashboard'))

@main_bp.route('/dashboard')
@login_required
def dashboard():
    """Dashboard principal"""

    # Récupération des dernières mesures
    mesure_reciente = MesureEnergie.query.order_by(
        MesureEnergie.timestamp.desc()
    ).first()

    # Mesures des dernières 24h
    debut = datetime.utcnow() - timedelta(hours=24)
    mesures_24h = MesureEnergie.query.filter(
        MesureEnergie.timestamp >= debut
    ).order_by(MesureEnergie.timestamp).all()

    # Événements récents
    evenements_recents = EvenementCompteur.query.order_by(
        EvenementCompteur.timestamp.desc()
    ).limit(10).all()

    return render_template(
        'dashboard.html',
        mesure_actuelle=mesure_reciente,
        mesures_24h=mesures_24h,
        evenements=evenements_recents,
        user=current_user
    )

@main_bp.route('/historique')
@login_required
def historique():
    """Page historique"""
    return render_template('historique.html')

@main_bp.route('/analyse')
@login_required
def analyse():
    """Page d'analyse"""
    return render_template('analyse.html')

@main_bp.route('/parametres')
@login_required
def parametres():
    """Page des paramètres"""
    return render_template('parametres.html')
```

### Template de base

```html
<!-- templates/base.html -->
<!DOCTYPE html>
<html lang="fr">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>{% block title %}Compteur E450 - Dashboard{% endblock %}</title>

    <!-- CSS -->
    <link href="https://cdn.jsdelivr.net/npm/bootstrap@5.3.0/dist/css/bootstrap.min.css" rel="stylesheet">
    <link href="{{ url_for('static', filename='css/style.css') }}" rel="stylesheet">

    {% block extra_head %}{% endblock %}
</head>
<body>
    <!-- Navigation -->
    <nav class="navbar navbar-expand-lg navbar-dark bg-primary">
        <div class="container">
            <a class="navbar-brand" href="{{ url_for('main.dashboard') }}">
                ⚡ Compteur E450
            </a>

            <button class="navbar-toggler" type="button" data-bs-toggle="collapse" data-bs-target="#navbarNav">
                <span class="navbar-toggler-icon"></span>
            </button>

            <div class="collapse navbar-collapse" id="navbarNav">
                <ul class="navbar-nav me-auto">
                    <li class="nav-item">
                        <a class="nav-link" href="{{ url_for('main.dashboard') }}">Dashboard</a>
                    </li>
                    <li class="nav-item">
                        <a class="nav-link" href="{{ url_for('main.historique') }}">Historique</a>
                    </li>
                    <li class="nav-item">
                        <a class="nav-link" href="{{ url_for('main.analyse') }}">Analyse</a>
                    </li>
                    <li class="nav-item">
                        <a class="nav-link" href="{{ url_for('main.parametres') }}">Paramètres</a>
                    </li>
                </ul>

                <ul class="navbar-nav">
                    <li class="nav-item dropdown">
                        <a class="nav-link dropdown-toggle" href="#" data-bs-toggle="dropdown">
                            {{ current_user.username }}
                        </a>
                        <ul class="dropdown-menu">
                            <li><a class="dropdown-item" href="{{ url_for('auth.logout') }}">Déconnexion</a></li>
                        </ul>
                    </li>
                </ul>
            </div>
        </div>
    </nav>

    <!-- Contenu principal -->
    <main class="container mt-4">
        {% block content %}{% endblock %}
    </main>

    <!-- Footer -->
    <footer class="bg-light text-center py-3 mt-5">
        <div class="container">
            <p class="mb-0">Compteur E450 - Dashboard Énergétique © 2025</p>
        </div>
    </footer>

    <!-- JavaScript -->
    <script src="https://cdn.jsdelivr.net/npm/bootstrap@5.3.0/dist/js/bootstrap.bundle.min.js"></script>
    <script src="{{ url_for('static', filename='js/app.js') }}"></script>

    {% block extra_scripts %}{% endblock %}
</body>
</html>
```

## 🧪 Tests et validation

### Tests de base

```python
# tests/test_basic.py
import pytest
from app import create_app

@pytest.fixture
def app():
    app = create_app('testing')
    return app

@pytest.fixture
def client(app):
    return app.test_client()

def test_home_page(client):
    """Test de la page d'accueil"""
    response = client.get('/')
    assert response.status_code == 302  # Redirection vers login

def test_config(app):
    """Test de la configuration"""
    assert app.config['TESTING'] == True
    assert app.config['SECRET_KEY'] is not None
```

### Lancement des tests

```bash
# Installation des dépendances de test
pip install pytest pytest-flask

# Lancement des tests
pytest

# Avec couverture
coverage run -m pytest
coverage report
```

## 🚀 Lancement de l'application

### Développement

```bash
# Activation de l'environnement virtuel
source venv/bin/activate  # Linux/Mac
# ou venv\Scripts\activate  # Windows

# Initialisation de la base de données
python scripts/init_db.py

# Lancement de l'application
python app.py
```

### Production

```bash
# Installation de Gunicorn
pip install gunicorn

# Lancement avec Gunicorn
gunicorn -w 4 -b 0.0.0.0:8000 app:app

# Ou avec configuration
gunicorn --config gunicorn.conf.py app:app
```

### Vérification

```bash
# Test de l'application
curl http://localhost:5000

# Vérification de la base de données
python -c "from app import create_app; app = create_app(); app.app_context().push(); from app.models import User; print('Utilisateurs:', User.query.count())"
```

> **💡 À retenir** : Une bonne structure de projet Flask facilite la maintenance et l'évolution du code. Commencez simple et ajoutez des fonctionnalités progressivement.

> **⚠️ Astuce** : Utilisez toujours des variables d'environnement pour les informations sensibles (clés secrètes, mots de passe) et versionnez-les dans un fichier `.env` exclu de Git.

Dans le prochain chapitre, nous explorerons plus en détail la **structure du projet Flask** et l'organisation du code !

---

**Navigation**
- [Chapitre précédent : Stockage et persistance des données](../Partie_II_Lecture_Acquisition/Chapitre_9_Stockage_Persistance.md)
- [Chapitre suivant : Structure du projet Flask](Chapitre_11_Structure_Projet_Flask.md)
- [Retour à la table des matières](../../README.md)
