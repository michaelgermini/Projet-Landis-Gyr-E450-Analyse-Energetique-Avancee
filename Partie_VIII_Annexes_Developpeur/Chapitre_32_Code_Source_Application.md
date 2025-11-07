# 💻 Chapitre 32 : Code source de l'application

## 🏗️ Architecture de l'application Flask

### Structure du projet

```
compteur_e450/
├── app/
│   ├── __init__.py          # Initialisation Flask
│   ├── models.py            # Modèles de données SQLAlchemy
│   ├── routes/
│   │   ├── api.py           # Routes API REST
│   │   ├── web.py           # Routes web (interface)
│   │   └── auth.py          # Routes authentification
│   ├── services/
│   │   ├── lecteur_compteur.py    # Service lecture compteur
│   │   ├── analyse_donnees.py     # Service analyse
│   │   └── mqtt_client.py          # Service MQTT
│   ├── templates/           # Templates Jinja2
│   └── static/              # Assets CSS/JS
├── config.py                # Configuration
├── requirements.txt         # Dépendances Python
├── run.py                   # Point d'entrée
└── migrations/              # Migrations base de données
```

### Fichier principal d'initialisation

```python
# app/__init__.py
from flask import Flask
from flask_sqlalchemy import SQLAlchemy
from flask_migrate import Migrate
from flask_login import LoginManager
from flask_cors import CORS
import os

db = SQLAlchemy()
migrate = Migrate()
login_manager = LoginManager()

def create_app(config_class='config.DevelopmentConfig'):
    app = Flask(__name__)
    app.config.from_object(config_class)

    # Extensions
    db.init_app(app)
    migrate.init_app(app, db)
    login_manager.init_app(app)
    CORS(app)

    # Blueprints
    from app.routes.api import api_bp
    from app.routes.web import web_bp
    from app.routes.auth import auth_bp

    app.register_blueprint(api_bp, url_prefix='/api')
    app.register_blueprint(web_bp)
    app.register_blueprint(auth_bp, url_prefix='/auth')

    # Services
    from app.services import init_services
    init_services(app)

    return app
```

### Modèles de données

```python
# app/models.py
from app import db
from datetime import datetime
from werkzeug.security import generate_password_hash, check_password_hash
from flask_login import UserMixin

class User(db.Model, UserMixin):
    id = db.Column(db.Integer, primary_key=True)
    username = db.Column(db.String(64), unique=True, nullable=False)
    email = db.Column(db.String(120), unique=True, nullable=False)
    password_hash = db.Column(db.String(128))
    is_admin = db.Column(db.Boolean, default=False)
    created_at = db.Column(db.DateTime, default=datetime.utcnow)

    def set_password(self, password):
        self.password_hash = generate_password_hash(password)

    def check_password(self, password):
        return check_password_hash(self.password_hash, password)

class MesureEnergie(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    timestamp = db.Column(db.DateTime, default=datetime.utcnow, index=True)
    compteur_id = db.Column(db.String(20), index=True, default='E450_DEFAULT')

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

    # Qualité réseau
    frequence = db.Column(db.Float)
    facteur_puissance = db.Column(db.Float)
    thd_tension = db.Column(db.Float)
    thd_courant = db.Column(db.Float)

    # Métadonnées
    qualite_donnees = db.Column(db.String(10), default='good')  # good/warning/error

    def to_dict(self):
        return {
            'id': self.id,
            'timestamp': self.timestamp.isoformat(),
            'compteur_id': self.compteur_id,
            'energies': {
                'active_import': self.energie_active_import,
                'active_export': self.energie_active_export,
                'reactive_import': self.energie_reactive_import,
                'reactive_export': self.energie_reactive_export,
            },
            'puissances': {
                'active': self.puissance_active_totale,
                'reactive': self.puissance_reactive_totale,
                'apparente': self.puissance_apparente_totale,
            },
            'tensions': {
                'l1': self.tension_l1,
                'l2': self.tension_l2,
                'l3': self.tension_l3,
            },
            'courants': {
                'l1': self.courant_l1,
                'l2': self.courant_l2,
                'l3': self.courant_l3,
            },
            'qualite': {
                'frequence': self.frequence,
                'facteur_puissance': self.facteur_puissance,
                'thd_tension': self.thd_tension,
                'thd_courant': self.thd_courant,
            },
            'qualite_donnees': self.qualite_donnees
        }

class Anomalie(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    timestamp = db.Column(db.DateTime, default=datetime.utcnow, index=True)
    compteur_id = db.Column(db.String(20), index=True)
    type_anomalie = db.Column(db.String(50))  # surcharge, fuite, tension, etc.
    severite = db.Column(db.String(20))  # faible, moyenne, haute, critique
    description = db.Column(db.Text)
    valeur_mesuree = db.Column(db.Float)
    seuil_declencheur = db.Column(db.Float)
    statut = db.Column(db.String(20), default='active')  # active, resolue, ignoree
    resolue_le = db.Column(db.DateTime, nullable=True)

class ConfigurationCompteur(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    compteur_id = db.Column(db.String(20), primary_key=True)
    nom_affiche = db.Column(db.String(100))
    type_interface = db.Column(db.String(20))  # optique, mbus, g3plc
    adresse_ip = db.Column(db.String(15), nullable=True)
    port_mbus = db.Column(db.Integer, nullable=True)
    actif = db.Column(db.Boolean, default=True)
    created_at = db.Column(db.DateTime, default=datetime.utcnow)
    updated_at = db.Column(db.DateTime, default=datetime.utcnow, onupdate=datetime.utcnow)
```

### Service de lecture du compteur

```python
# app/services/lecteur_compteur.py
import serial
import time
import logging
from datetime import datetime
from typing import Dict, Optional, List
from app import db
from app.models import MesureEnergie

logger = logging.getLogger(__name__)

class LecteurCompteurE450:
    """Service de lecture du compteur Landis+Gyr E450"""

    def __init__(self, config_compteur: Dict):
        self.config = config_compteur
        self.interface = config_compteur.get('interface', 'optique')
        self.port = config_compteur.get('port', '/dev/ttyUSB0')
        self.compteur_id = config_compteur.get('compteur_id', 'E450_DEFAULT')

        # Configuration série selon l'interface
        self.serial_config = self._get_serial_config()

        # Codes OBIS essentiels
        self.codes_essentiels = [
            '1.8.0',   # Énergie active importée totale
            '2.8.0',   # Énergie active exportée totale
            '16.7.0',  # Puissance active totale
            '32.7.0',  # Tension L1
            '52.7.0',  # Tension L2
            '72.7.0',  # Tension L3
            '31.7.0',  # Courant L1
            '51.7.0',  # Courant L2
            '71.7.0',  # Courant L3
            '14.7.0',  # Fréquence
            '13.21.0', # Facteur de puissance
        ]

    def _get_serial_config(self) -> Dict:
        """Configuration série selon l'interface"""
        configs = {
            'optique': {
                'baudrate': 9600,
                'bytesize': serial.EIGHTBITS,
                'parity': serial.PARITY_NONE,
                'stopbits': serial.STOPBITS_ONE,
                'timeout': 5
            },
            'mbus': {
                'baudrate': 9600,
                'bytesize': serial.EIGHTBITS,
                'parity': serial.PARITY_NONE,
                'stopbits': serial.STOPBITS_ONE,
                'timeout': 2
            }
        }
        return configs.get(self.interface, configs['optique'])

    def connexion_compteur(self) -> bool:
        """Établit la connexion avec le compteur"""
        try:
            with serial.Serial(self.port, **self.serial_config) as ser:
                if self.interface == 'optique':
                    return self._connexion_optique(ser)
                elif self.interface == 'mbus':
                    return self._connexion_mbus(ser)
                else:
                    logger.error(f"Interface non supportée: {self.interface}")
                    return False
        except Exception as e:
            logger.error(f"Erreur connexion {self.interface}: {e}")
            return False

    def _connexion_optique(self, ser) -> bool:
        """Connexion via port optique (IEC 62056-21)"""
        try:
            # Mode A - Identification
            ser.write(b'/?!\r\n')
            time.sleep(0.5)
            reponse = ser.read(100)

            if b'/' in reponse:  # Réponse d'identification valide
                # Mode C - Communication complète
                ser.write(b'\x06000\r\n')  # ACK + numéro client
                time.sleep(0.5)
                return True
            else:
                logger.warning(f"Réponse identification inattendue: {reponse}")
                return False

        except Exception as e:
            logger.error(f"Erreur connexion optique: {e}")
            return False

    def _connexion_mbus(self, ser) -> bool:
        """Connexion via bus M-Bus"""
        try:
            # Test de communication M-Bus
            adresse = self.config.get('adresse_mbus', 1)
            requete_ping = self._construire_ping_mbus(adresse)

            ser.write(requete_ping)
            time.sleep(1)

            reponse = ser.read(20)
            return len(reponse) > 5  # Réponse minimale valide

        except Exception as e:
            logger.error(f"Erreur connexion M-Bus: {e}")
            return False

    def lire_mesures(self) -> Optional[MesureEnergie]:
        """Lit une série complète de mesures"""
        try:
            donnees = {}

            with serial.Serial(self.port, **self.serial_config) as ser:
                for code in self.codes_essentiels:
                    valeur = self._lire_code_obis(ser, code)
                    if valeur is not None:
                        donnees[code] = valeur
                    time.sleep(0.1)  # Pause entre lectures

            if donnees:
                mesure = self._convertir_mesure(donnees)
                mesure.compteur_id = self.compteur_id

                # Validation des données
                if self._valider_mesure(mesure):
                    return mesure
                else:
                    logger.warning("Mesure invalidée par validation")
                    return None
            else:
                logger.warning("Aucune donnée lue")
                return None

        except Exception as e:
            logger.error(f"Erreur lecture mesures: {e}")
            return None

    def _lire_code_obis(self, ser, code_obis: str) -> Optional[float]:
        """Lit un code OBIS spécifique"""
        try:
            if self.interface == 'optique':
                return self._lire_code_optique(ser, code_obis)
            elif self.interface == 'mbus':
                return self._lire_code_mbus(ser, code_obis)
            else:
                return None

        except Exception as e:
            logger.debug(f"Erreur lecture code {code_obis}: {e}")
            return None

    def _lire_code_optique(self, ser, code_obis: str) -> Optional[float]:
        """Lit un code via interface optique"""
        # Format DLMS/COSEM simplifié
        requete = f'R1\r\n{code_obis}()\r\n'.encode()

        ser.write(requete)
        time.sleep(0.2)

        reponse = ser.read(200)
        return self._parser_reponse_dlms(reponse)

    def _lire_code_mbus(self, ser, code_obis: str) -> Optional[float]:
        """Lit un code via bus M-Bus"""
        adresse = self.config.get('adresse_mbus', 1)
        requete = self._construire_requete_mbus(adresse, code_obis)

        ser.write(requete)
        time.sleep(0.5)

        reponse = ser.read(200)
        return self._parser_reponse_mbus(reponse)

    def _convertir_mesure(self, donnees: Dict[str, float]) -> MesureEnergie:
        """Convertit les données brutes en objet MesureEnergie"""
        mesure = MesureEnergie()

        # Énergies (convertir en kWh si nécessaire)
        mesure.energie_active_import = donnees.get('1.8.0')
        mesure.energie_active_export = donnees.get('2.8.0')

        # Puissances (convertir en W si nécessaire)
        mesure.puissance_active_totale = donnees.get('16.7.0')

        # Tensions (convertir en V si nécessaire)
        mesure.tension_l1 = donnees.get('32.7.0')
        mesure.tension_l2 = donnees.get('52.7.0')
        mesure.tension_l3 = donnees.get('72.7.0')

        # Courants (convertir en A si nécessaire)
        mesure.courant_l1 = donnees.get('31.7.0')
        mesure.courant_l2 = donnees.get('51.7.0')
        mesure.courant_l3 = donnees.get('71.7.0')

        # Qualité réseau
        mesure.frequence = donnees.get('14.7.0')
        mesure.facteur_puissance = donnees.get('13.21.0')

        return mesure

    def _valider_mesure(self, mesure: MesureEnergie) -> bool:
        """Valide la cohérence d'une mesure"""
        # Vérifications de base
        if mesure.puissance_active_totale and mesure.puissance_active_totale < 0:
            return False

        # Tensions dans plage réaliste
        tensions = [mesure.tension_l1, mesure.tension_l2, mesure.tension_l3]
        for tension in tensions:
            if tension and not (180 < tension < 260):
                return False

        # Fréquence réseau
        if mesure.frequence and not (49.5 < mesure.frequence < 50.5):
            return False

        return True

    def sauvegarder_mesure(self, mesure: MesureEnergie) -> bool:
        """Sauvegarde une mesure en base"""
        try:
            db.session.add(mesure)
            db.session.commit()
            logger.info(f"Mesure sauvegardée: {mesure.timestamp}")
            return True
        except Exception as e:
            logger.error(f"Erreur sauvegarde: {e}")
            db.session.rollback()
            return False

    # Méthodes utilitaires M-Bus (implémentations simplifiées)
    def _construire_ping_mbus(self, adresse: int) -> bytes:
        """Construit une requête ping M-Bus"""
        # Implémentation simplifiée
        return bytes([0x68, 0x03, 0x03, 0x68, adresse, 0x00, 0x00])

    def _construire_requete_mbus(self, adresse: int, code_obis: str) -> bytes:
        """Construit une requête M-Bus pour un code OBIS"""
        # Implémentation simplifiée - à adapter selon spécifications
        return bytes([0x68, 0x0B, 0x0B, 0x68, adresse, 0x00, 0x00, 0x00])

    def _parser_reponse_dlms(self, reponse: bytes) -> Optional[float]:
        """Parse une réponse DLMS/COSEM"""
        try:
            texte = reponse.decode('ascii', errors='ignore')
            # Logique de parsing simplifiée
            # En production: utiliser une vraie bibliothèque DLMS
            return 0.0  # Valeur temporaire
        except:
            return None

    def _parser_reponse_mbus(self, reponse: bytes) -> Optional[float]:
        """Parse une réponse M-Bus"""
        try:
            if len(reponse) < 10:
                return None
            # Logique de décodage M-Bus (BCD, etc.)
            return 0.0  # Valeur temporaire
        except:
            return None
```

### Configuration

```python
# config.py
import os
from datetime import timedelta

class Config:
    SECRET_KEY = os.environ.get('SECRET_KEY') or 'dev-key-change-in-production'
    SQLALCHEMY_DATABASE_URI = os.environ.get('DATABASE_URL') or 'sqlite:///app.db'
    SQLALCHEMY_TRACK_MODIFICATIONS = False

    # JWT
    JWT_SECRET_KEY = os.environ.get('JWT_SECRET_KEY') or 'jwt-secret-key'
    JWT_ACCESS_TOKEN_EXPIRE = timedelta(hours=1)

    # MQTT
    MQTT_BROKER = os.environ.get('MQTT_BROKER') or 'localhost'
    MQTT_PORT = int(os.environ.get('MQTT_PORT') or 1883)
    MQTT_USERNAME = os.environ.get('MQTT_USERNAME')
    MQTT_PASSWORD = os.environ.get('MQTT_PASSWORD')

    # InfluxDB (optionnel)
    INFLUXDB_URL = os.environ.get('INFLUXDB_URL')
    INFLUXDB_TOKEN = os.environ.get('INFLUXDB_TOKEN')
    INFLUXDB_ORG = os.environ.get('INFLUXDB_ORG')
    INFLUXDB_BUCKET = os.environ.get('INFLUXDB_BUCKET')

    # Compteurs
    COMPTEURS_CONFIG = {
        'E450_PRINCIPAL': {
            'interface': 'optique',
            'port': '/dev/ttyUSB0',
            'actif': True
        },
        'E450_SECONDAIRE': {
            'interface': 'mbus',
            'port': '/dev/ttyUSB1',
            'adresse_mbus': 1,
            'actif': False
        }
    }

class DevelopmentConfig(Config):
    DEBUG = True
    SQLALCHEMY_DATABASE_URI = 'sqlite:///dev.db'

class ProductionConfig(Config):
    DEBUG = False
    # Configuration production avec variables d'environnement
```

### Point d'entrée

```python
# run.py
from app import create_app, db
from app.models import User, ConfigurationCompteur
import os

app = create_app(os.environ.get('FLASK_ENV') or 'development')

@app.shell_context_processor
def make_shell_context():
    return {'db': db, 'User': User, 'Config': ConfigurationCompteur}

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000, debug=True)
```

---

**Navigation**
- [Chapitre suivant : Scripts utilitaires](Chapitre_33_Scripts_Utilitaires.md)
- [Retour à la table des matières](../../README.md)
