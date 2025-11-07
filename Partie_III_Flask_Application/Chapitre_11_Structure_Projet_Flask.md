# 💻 Chapitre 11 : Structure du projet Flask

## 🏗️ Architecture applicative MVC

Notre application Flask suit une architecture **MVC (Modèle-Vue-Contrôleur)** adaptée aux besoins d'une application de monitoring énergétique.

### Séparation des responsabilités

```
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   Modèles       │    │  Contrôleurs    │    │     Vues        │
│   (SQLAlchemy)  │────│   (Routes)      │────│   (Jinja2)      │
├─────────────────┤    ├─────────────────┤    ├─────────────────┤
│ • MesureEnergie │    │ • /dashboard    │    │ • dashboard.html│
│ • Compteur      │    │ • /api/data     │    │ • historique.html│
│ • Alerte        │    │ • /historique   │    │ • api.json      │
│ • Configuration │    │ • /analyse      │    │ • charts.js     │
└─────────────────┘    └─────────────────┘    └─────────────────┘
                   │                        │
                   └────────────────────────┘
                            │
                   ┌─────────────────┐
                   │   Services      │
                   │   métier        │
                   ├─────────────────┤
                   │ • LecteurE450   │
                   │ • Calculateur   │
                   │ • Detecteur     │
                   │ • Predicteur    │
                   └─────────────────┘
```

## 📦 Organisation modulaire

### Package principal `app/`

Le dossier `app/` contient le code principal de l'application, organisé en modules spécialisés.

#### `__init__.py` - Factory de l'application

```python
# app/__init__.py
from flask import Flask
from config import get_config

def create_app(config_name=None):
    """Application factory pattern"""

    app = Flask(__name__)

    # Configuration
    app.config.from_object(get_config(config_name))

    # Extensions
    from app.extensions import db, migrate, login_manager, scheduler
    db.init_app(app)
    migrate.init_app(app, db)
    login_manager.init_app(app)
    scheduler.init_app(app)

    # Blueprints
    from app.routes import register_blueprints
    register_blueprints(app)

    # Gestionnaires d'erreurs
    register_error_handlers(app)

    # Contexte global
    register_context_processors(app)

    # Démarrage du scheduler
    if not app.config['TESTING']:
        scheduler.start()

    return app

def register_error_handlers(app):
    """Gestionnaires d'erreurs"""

    @app.errorhandler(404)
    def not_found(error):
        return render_template('errors/404.html'), 404

    @app.errorhandler(500)
    def internal_error(error):
        return render_template('errors/500.html'), 500

def register_context_processors(app):
    """Variables globales dans les templates"""

    @app.context_processor
    def inject_globals():
        from flask_login import current_user
        return {
            'current_user': current_user,
            'config': app.config
        }
```

#### `models.py` - Couche de données

```python
# app/models.py
from app.extensions import db
from flask_login import UserMixin
from werkzeug.security import generate_password_hash, check_password_hash
from datetime import datetime

class User(UserMixin, db.Model):
    """Utilisateur de l'application"""
    __tablename__ = 'users'

    id = db.Column(db.Integer, primary_key=True)
    username = db.Column(db.String(64), unique=True, nullable=False)
    email = db.Column(db.String(120), unique=True, nullable=False)
    password_hash = db.Column(db.String(128), nullable=False)
    is_admin = db.Column(db.Boolean, default=False)
    created_at = db.Column(db.DateTime, default=datetime.utcnow)
    last_login = db.Column(db.DateTime)

    # Relations
    mesures = db.relationship('MesureEnergie', backref='user', lazy='dynamic')

    def set_password(self, password):
        self.password_hash = generate_password_hash(password)

    def check_password(self, password):
        return check_password_hash(self.password_hash, password)

    def __repr__(self):
        return f'<User {self.username}>'

class MesureEnergie(db.Model):
    """Mesure énergétique avec métadonnées"""
    __tablename__ = 'mesures_energie'

    id = db.Column(db.Integer, primary_key=True)
    timestamp = db.Column(db.DateTime, default=datetime.utcnow, index=True)

    # Utilisateur (optionnel pour multi-utilisateurs)
    user_id = db.Column(db.Integer, db.ForeignKey('users.id'))

    # Énergies cumulées (kWh)
    energie_active_import = db.Column(db.Float)
    energie_active_export = db.Column(db.Float)
    energie_reactive_import = db.Column(db.Float)
    energie_reactive_export = db.Column(db.Float)

    # Puissances instantanées (W)
    puissance_active_totale = db.Column(db.Float)
    puissance_reactive_totale = db.Column(db.Float)
    puissance_apparente_totale = db.Column(db.Float)

    # Par phase - Tensions (V)
    tension_l1 = db.Column(db.Float)
    tension_l2 = db.Column(db.Float)
    tension_l3 = db.Column(db.Float)

    # Par phase - Courants (A)
    courant_l1 = db.Column(db.Float)
    courant_l2 = db.Column(db.Float)
    courant_l3 = db.Column(db.Float)
    courant_neutre = db.Column(db.Float)

    # Qualité du réseau
    frequence = db.Column(db.Float)  # Hz
    facteur_puissance = db.Column(db.Float)  # 0-1
    thd_tension = db.Column(db.Float)  # %
    thd_courant = db.Column(db.Float)  # %

    # Métadonnées de mesure
    source = db.Column(db.String(20), default='e450')  # e450, manuel, calculé
    qualite = db.Column(db.String(10), default='good')  # good, suspect, bad
    compteur_id = db.Column(db.String(20), default='E450001234')

    # Index pour les performances
    __table_args__ = (
        db.Index('idx_mesure_timestamp_qualite', 'timestamp', 'qualite'),
        db.Index('idx_mesure_source', 'source'),
    )

    def to_dict(self):
        """Conversion en dictionnaire pour l'API"""
        return {
            'id': self.id,
            'timestamp': self.timestamp.isoformat() + 'Z',
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
                'neutre': self.courant_neutre,
            },
            'qualite_reseau': {
                'frequence': self.frequence,
                'facteur_puissance': self.facteur_puissance,
                'thd_tension': self.thd_tension,
                'thd_courant': self.thd_courant,
            },
            'source': self.source,
            'qualite': self.qualite,
        }

    def __repr__(self):
        return f'<MesureEnergie {self.timestamp} {self.puissance_active_totale}W>'

class EvenementCompteur(db.Model):
    """Événement système ou alerte"""
    __tablename__ = 'evenements_compteur'

    id = db.Column(db.Integer, primary_key=True)
    timestamp = db.Column(db.DateTime, default=datetime.utcnow, index=True)

    # Classification
    type_evenement = db.Column(db.String(50), nullable=False, index=True)
    severite = db.Column(db.String(20), default='info')  # info, warning, error, critical
    categorie = db.Column(db.String(30), default='systeme')  # systeme, reseau, mesure

    # Description
    titre = db.Column(db.String(100))
    description = db.Column(db.Text)

    # Données techniques
    valeur_mesuree = db.Column(db.Float)
    unite = db.Column(db.String(10))
    code_obis = db.Column(db.String(20))
    seuil_declencheur = db.Column(db.Float)

    # Contexte
    compteur_id = db.Column(db.String(20), default='E450001234')
    user_id = db.Column(db.Integer, db.ForeignKey('users.id'))

    # État
    acquitte = db.Column(db.Boolean, default=False)
    timestamp_acquittement = db.Column(db.DateTime)

    def to_dict(self):
        """Conversion pour l'API"""
        return {
            'id': self.id,
            'timestamp': self.timestamp.isoformat() + 'Z',
            'type': self.type_evenement,
            'severite': self.severite,
            'categorie': self.categorie,
            'titre': self.titre,
            'description': self.description,
            'valeur_mesuree': self.valeur_mesuree,
            'unite': self.unite,
            'code_obis': self.code_obis,
            'seuil_declencheur': self.seuil_declencheur,
            'acquitte': self.acquitte,
        }

    def __repr__(self):
        return f'<Evenement {self.type_evenement} {self.severite}>'

class Configuration(db.Model):
    """Configuration persistante de l'application"""
    __tablename__ = 'configuration'

    id = db.Column(db.Integer, primary_key=True)
    parametre = db.Column(db.String(100), unique=True, nullable=False)
    valeur = db.Column(db.String(255))
    type_valeur = db.Column(db.String(20), default='string')  # string, int, float, bool
    categorie = db.Column(db.String(30), default='general')
    description = db.Column(db.Text)
    updated_at = db.Column(db.DateTime, default=datetime.utcnow, onupdate=datetime.utcnow)

    def get_valeur(self):
        """Retourne la valeur typée"""
        if self.type_valeur == 'int':
            return int(self.valeur)
        elif self.type_valeur == 'float':
            return float(self.valeur)
        elif self.type_valeur == 'bool':
            return self.valeur.lower() in ('true', '1', 'yes', 'on')
        else:
            return self.valeur

    def set_valeur(self, valeur):
        """Définit la valeur en la convertissant en string"""
        self.valeur = str(valeur)

    def __repr__(self):
        return f'<Configuration {self.parametre}={self.valeur}>'
```

#### `routes.py` - Contrôleurs et API

```python
# app/routes.py
from flask import Blueprint, render_template, request, jsonify, redirect, url_for, flash
from flask_login import login_required, current_user
from app.models import MesureEnergie, EvenementCompteur, Configuration, db
from app.services import lecteur_compteur, calculateur_energie, detecteur_anomalies
from datetime import datetime, timedelta
import json

# Blueprints
main_bp = Blueprint('main', __name__)
api_bp = Blueprint('api', __name__)

def register_blueprints(app):
    """Enregistrement de tous les blueprints"""
    app.register_blueprint(main_bp)
    app.register_blueprint(api_bp, url_prefix='/api')

# Routes principales
@main_bp.route('/')
@login_required
def index():
    """Page d'accueil"""
    return redirect(url_for('main.dashboard'))

@main_bp.route('/dashboard')
@login_required
def dashboard():
    """Dashboard principal avec données temps réel"""

    # Mesure la plus récente
    mesure_actuelle = MesureEnergie.query.order_by(
        MesureEnergie.timestamp.desc()
    ).first()

    # Statistiques des dernières 24h
    debut_24h = datetime.utcnow() - timedelta(hours=24)
    stats_24h = db.session.query(
        db.func.count(MesureEnergie.id).label('nb_mesures'),
        db.func.avg(MesureEnergie.puissance_active_totale).label('puissance_moyenne'),
        db.func.max(MesureEnergie.puissance_active_totale).label('puissance_max'),
        db.func.min(MesureEnergie.puissance_active_totale).label('puissance_min'),
    ).filter(MesureEnergie.timestamp >= debut_24h).first()

    # Événements récents non acquittés
    evenements_recents = EvenementCompteur.query.filter_by(
        acquitte=False
    ).order_by(EvenementCompteur.timestamp.desc()).limit(5).all()

    # Calculs pour le dashboard
    if mesure_actuelle:
        cout_estime = calculateur_energie.calculer_cout_estime(
            mesure_actuelle.energie_active_import,
            tarif_horaire=True
        )
    else:
        cout_estime = 0

    return render_template('dashboard.html',
                         mesure_actuelle=mesure_actuelle,
                         stats_24h=stats_24h,
                         evenements=evenements_recents,
                         cout_estime=cout_estime)

@main_bp.route('/historique')
@login_required
def historique():
    """Page historique avec filtres"""

    # Paramètres de filtrage
    jours = int(request.args.get('jours', 7))
    debut = datetime.utcnow() - timedelta(days=jours)

    # Récupération des mesures
    mesures = MesureEnergie.query.filter(
        MesureEnergie.timestamp >= debut
    ).order_by(MesureEnergie.timestamp).all()

    # Statistiques périodiques
    stats = calculateur_energie.calculer_stats_periode(mesures)

    return render_template('historique.html',
                         mesures=mesures,
                         stats=stats,
                         jours=jours)

@main_bp.route('/analyse')
@login_required
def analyse():
    """Page d'analyse avancée"""

    # Analyses disponibles
    analyses = {
        'consommation_quotidienne': calculateur_energie.analyse_consommation_quotidienne(),
        'profil_puissance': calculateur_energie.analyse_profil_puissance(),
        'anomalies_detectees': detecteur_anomalies.detecter_anomalies_recentes(),
        'previsions': calculateur_energie.prevoir_consommation_jour_suivant(),
    }

    return render_template('analyse.html', analyses=analyses)

# API REST
@api_bp.route('/data')
@login_required
def get_data():
    """API pour récupérer les données de mesure"""

    # Paramètres
    limit = int(request.args.get('limit', 100))
    heures = int(request.args.get('hours', 24))
    format_data = request.args.get('format', 'json')

    debut = datetime.utcnow() - timedelta(hours=heures)

    # Récupération
    mesures = MesureEnergie.query.filter(
        MesureEnergie.timestamp >= debut
    ).order_by(MesureEnergie.timestamp.desc()).limit(limit).all()

    # Formatage
    data = [mesure.to_dict() for mesure in mesures]

    if format_data == 'csv':
        # Export CSV
        import csv
        from io import StringIO

        output = StringIO()
        writer = csv.DictWriter(output, fieldnames=data[0].keys() if data else [])
        writer.writeheader()
        writer.writerows(data)

        from flask import Response
        return Response(output.getvalue(),
                       mimetype='text/csv',
                       headers={'Content-Disposition': 'attachment; filename=data.csv'})

    return jsonify({
        'success': True,
        'count': len(data),
        'data': data,
        'periode': {
            'debut': debut.isoformat() + 'Z',
            'fin': datetime.utcnow().isoformat() + 'Z'
        }
    })

@api_bp.route('/data/current')
@login_required
def get_current_data():
    """Données de la mesure la plus récente"""

    mesure = MesureEnergie.query.order_by(MesureEnergie.timestamp.desc()).first()

    if mesure:
        return jsonify({
            'success': True,
            'data': mesure.to_dict()
        })
    else:
        return jsonify({
            'success': False,
            'error': 'Aucune donnée disponible'
        }), 404

@api_bp.route('/events')
@login_required
def get_events():
    """API des événements"""

    # Filtres
    severite = request.args.get('severite')
    type_evt = request.args.get('type')
    acquitte = request.args.get('acquitte', 'false').lower() == 'true'
    limit = int(request.args.get('limit', 50))

    query = EvenementCompteur.query

    if severite:
        query = query.filter_by(severite=severite)
    if type_evt:
        query = query.filter_by(type_evenement=type_evt)
    if not acquitte:
        query = query.filter_by(acquitte=False)

    evenements = query.order_by(EvenementCompteur.timestamp.desc()).limit(limit).all()

    return jsonify({
        'success': True,
        'count': len(evenements),
        'events': [evt.to_dict() for evt in evenements]
    })

@api_bp.route('/stats')
@login_required
def get_stats():
    """Statistiques générales"""

    # Statistiques générales
    nb_mesures = MesureEnergie.query.count()
    nb_evenements = EvenementCompteur.query.count()

    # Période couverte
    premiere_mesure = MesureEnergie.query.order_by(MesureEnergie.timestamp).first()
    derniere_mesure = MesureEnergie.query.order_by(MesureEnergie.timestamp.desc()).first()

    periode_jours = 0
    if premiere_mesure and derniere_mesure:
        periode_jours = (derniere_mesure.timestamp - premiere_mesure.timestamp).days

    # Statistiques des dernières 24h
    debut_24h = datetime.utcnow() - timedelta(hours=24)
    mesures_24h = MesureEnergie.query.filter(MesureEnergie.timestamp >= debut_24h).all()

    puissance_moyenne = sum(m.puissance_active_totale or 0 for m in mesures_24h) / len(mesures_24h) if mesures_24h else 0
    puissance_max = max((m.puissance_active_totale or 0) for m in mesures_24h) if mesures_24h else 0

    return jsonify({
        'success': True,
        'general': {
            'nb_mesures': nb_mesures,
            'nb_evenements': nb_evenements,
            'periode_jours': periode_jours,
            'date_derniere_mesure': derniere_mesure.timestamp.isoformat() + 'Z' if derniere_mesure else None,
        },
        'dernières_24h': {
            'nb_mesures': len(mesures_24h),
            'puissance_moyenne': round(puissance_moyenne, 1),
            'puissance_max': puissance_max,
        }
    })

@api_bp.route('/config', methods=['GET', 'POST'])
@login_required
def manage_config():
    """Gestion de la configuration via API"""

    if request.method == 'GET':
        # Récupération de toutes les configurations
        configs = Configuration.query.all()
        return jsonify({
            'success': True,
            'configurations': [{
                'parametre': c.parametre,
                'valeur': c.get_valeur(),
                'type': c.type_valeur,
                'categorie': c.categorie,
                'description': c.description,
            } for c in configs]
        })

    elif request.method == 'POST':
        # Mise à jour d'une configuration
        data = request.get_json()

        if not data or 'parametre' not in data:
            return jsonify({'success': False, 'error': 'Paramètre requis'}), 400

        config = Configuration.query.filter_by(parametre=data['parametre']).first()
        if config:
            config.set_valeur(data.get('valeur', config.valeur))
            db.session.commit()
            return jsonify({'success': True, 'message': 'Configuration mise à jour'})
        else:
            return jsonify({'success': False, 'error': 'Configuration introuvable'}), 404
```

### Dossier `services/` - Logique métier

Le dossier `services/` contient la logique métier réutilisable, indépendante de l'interface web.

#### `services/__init__.py`

```python
# app/services/__init__.py
from .lecteur_compteur import LecteurCompteur
from .calculateur_energie import CalculateurEnergie
from .detecteur_anomalies import DetecteurAnomalies

# Instances singleton
lecteur_compteur = LecteurCompteur()
calculateur_energie = CalculateurEnergie()
detecteur_anomalies = DetecteurAnomalies()

__all__ = ['lecteur_compteur', 'calculateur_energie', 'detecteur_anomalies']
```

#### `services/lecteur_compteur.py`

```python
# app/services/lecteur_compteur.py
from app.models import MesureEnergie, EvenementCompteur, db
from app.extensions import scheduler
import logging
from datetime import datetime

logger = logging.getLogger(__name__)

class LecteurCompteur:
    """Service de lecture du compteur E450"""

    def __init__(self):
        self.lecteurs = {}  # Cache des instances de lecteur
        self._initialiser_lecteurs()

    def _initialiser_lecteurs(self):
        """Initialisation des différents types de lecteurs"""
        from flask import current_app

        config = current_app.config

        # Lecteur port optique
        if config.get('PORT_OPTICAL'):
            try:
                from app.services.lecteurs.lecteur_optique import LecteurOptique
                self.lecteurs['optique'] = LecteurOptique(config['PORT_OPTICAL'])
            except ImportError:
                logger.warning("Lecteur optique non disponible")

        # Lecteur M-Bus
        if config.get('MBUS_PORT'):
            try:
                from app.services.lecteurs.lecteur_mbus import LecteurMBus
                self.lecteurs['mbus'] = LecteurMBus(
                    config['MBUS_PORT'],
                    config['MBUS_BAUDRATE'],
                    config['MBUS_ADDRESSE_COMPTEUR']
                )
            except ImportError:
                logger.warning("Lecteur M-Bus non disponible")

    def lire_donnees(self, source_preferee=None):
        """
        Lecture des données du compteur

        Args:
            source_preferee: 'optique', 'mbus', ou None (auto)

        Returns:
            dict: Données lues ou None si erreur
        """
        sources_essai = []

        if source_preferee and source_preferee in self.lecteurs:
            sources_essai = [source_preferee]
        else:
            # Ordre de préférence
            sources_essai = ['optique', 'mbus']

        for source in sources_essai:
            if source in self.lecteurs:
                try:
                    logger.info(f"Tentative de lecture via {source}")
                    donnees = self.lecteurs[source].lire_donnees()

                    if donnees:
                        donnees['source'] = source
                        logger.info(f"Données lues avec succès via {source}")
                        return donnees
                    else:
                        logger.warning(f"Échec de lecture via {source}")

                except Exception as e:
                    logger.error(f"Erreur lecture {source} : {e}")
                    continue

        logger.error("Toutes les méthodes de lecture ont échoué")
        return None

    def sauvegarder_mesure(self, donnees):
        """
        Sauvegarde d'une mesure en base

        Args:
            donnees: dict des données du compteur

        Returns:
            MesureEnergie: Instance créée
        """
        try:
            # Création de la mesure
            mesure = MesureEnergie(
                energie_active_import=donnees.get('energie_active_import'),
                energie_active_export=donnees.get('energie_active_export'),
                puissance_active_totale=donnees.get('puissance_active_totale'),
                tension_l1=donnees.get('tension_l1'),
                tension_l2=donnees.get('tension_l2'),
                tension_l3=donnees.get('tension_l3'),
                courant_l1=donnees.get('courant_l1'),
                courant_l2=donnees.get('courant_l2'),
                courant_l3=donnees.get('courant_l3'),
                frequence=donnees.get('frequence'),
                facteur_puissance=donnees.get('facteur_puissance'),
                source=donnees.get('source', 'manuel'),
                compteur_id=donnees.get('compteur_id', 'E450001234'),
            )

            db.session.add(mesure)
            db.session.commit()

            logger.info(f"Mesure sauvegardée : {mesure}")
            return mesure

        except Exception as e:
            db.session.rollback()
            logger.error(f"Erreur sauvegarde mesure : {e}")
            raise

    def demarrer_lecture_automatique(self, intervalle_minutes=5):
        """
        Démarrage de la lecture automatique planifiée

        Args:
            intervalle_minutes: Intervalle entre les lectures
        """
        @scheduler.task('interval', id='lecture_compteur',
                       minutes=intervalle_minutes, max_instances=1)
        def lecture_automatique():
            with scheduler.app.app_context():
                logger.info("Lecture automatique du compteur")

                try:
                    # Lecture des données
                    donnees = self.lire_donnees()

                    if donnees:
                        # Sauvegarde
                        mesure = self.sauvegarder_mesure(donnees)

                        # Détection d'anomalies
                        from app.services import detecteur_anomalies
                        anomalies = detecteur_anomalies.detecter_anomalie_mesure(mesure)

                        if anomalies:
                            for anomalie in anomalies:
                                self._creer_evenement_anomalie(anomalie, mesure)

                        logger.info("Lecture automatique terminée avec succès")
                    else:
                        self._creer_evenement_erreur("Échec de lecture du compteur")

                except Exception as e:
                    logger.error(f"Erreur lecture automatique : {e}")
                    self._creer_evenement_erreur(f"Erreur lecture automatique : {str(e)}")

        logger.info(f"Planification des lectures automatiques : toutes les {intervalle_minutes} minutes")

    def _creer_evenement_erreur(self, message):
        """Création d'un événement d'erreur"""
        try:
            evenement = EvenementCompteur(
                type_evenement='erreur_lecture',
                severite='error',
                categorie='systeme',
                titre='Erreur de lecture',
                description=message,
            )
            db.session.add(evenement)
            db.session.commit()
        except Exception as e:
            logger.error(f"Erreur création événement : {e}")

    def _creer_evenement_anomalie(self, anomalie, mesure):
        """Création d'un événement d'anomalie"""
        try:
            evenement = EvenementCompteur(
                type_evenement=anomalie['type'],
                severite=anomalie['severite'],
                categorie='mesure',
                titre=anomalie['titre'],
                description=anomalie['description'],
                valeur_mesuree=mesure.puissance_active_totale,
                unite='W',
                seuil_declencheur=anomalie.get('seuil'),
            )
            db.session.add(evenement)
            db.session.commit()
        except Exception as e:
            logger.error(f"Erreur création événement anomalie : {e}")
```

## 📊 Tests et qualité

### Structure des tests

```
tests/
├── __init__.py
├── conftest.py              # Configuration pytest
├── test_models.py           # Tests des modèles
├── test_routes.py           # Tests des routes
├── test_services.py         # Tests des services
├── test_lecteurs.py         # Tests des lecteurs
├── fixtures/                # Données de test
│   ├── mesures_sample.json
│   └── config_test.json
└── integration/             # Tests d'intégration
    ├── test_full_workflow.py
    └── test_api_endpoints.py
```

#### `tests/conftest.py`

```python
# tests/conftest.py
import pytest
import os
import tempfile
from app import create_app
from app.extensions import db
from app.models import User

@pytest.fixture(scope='session')
def app():
    """Application de test"""
    # Base de données temporaire
    db_fd, db_path = tempfile.mkstemp()

    app = create_app('testing')
    app.config['SQLALCHEMY_DATABASE_URI'] = f'sqlite:///{db_path}'
    app.config['TESTING'] = True

    with app.app_context():
        db.create_all()

        # Création d'un utilisateur de test
        user = User(username='test', email='test@example.com')
        user.set_password('test')
        db.session.add(user)
        db.session.commit()

        yield app

        # Nettoyage
        db.session.remove()
        os.close(db_fd)
        os.unlink(db_path)

@pytest.fixture
def client(app):
    """Client de test"""
    return app.test_client()

@pytest.fixture
def runner(app):
    """Runner de commandes CLI"""
    return app.test_cli_runner()

@pytest.fixture
def auth_client(client, app):
    """Client authentifié"""
    with app.app_context():
        user = User.query.filter_by(username='test').first()

        client.post('/auth/login', data={
            'username': 'test',
            'password': 'test'
        })

        return client
```

#### Exemple de test

```python
# tests/test_models.py
import pytest
from app.models import MesureEnergie, User

def test_creer_mesure(app):
    """Test de création d'une mesure"""
    with app.app_context():
        mesure = MesureEnergie(
            puissance_active_totale=2345.0,
            tension_l1=231.5,
            source='test'
        )

        from app.extensions import db
        db.session.add(mesure)
        db.session.commit()

        assert mesure.id is not None
        assert mesure.puissance_active_totale == 2345.0
        assert mesure.source == 'test'

def test_user_password(app):
    """Test de la gestion des mots de passe"""
    with app.app_context():
        user = User(username='testuser', email='test@example.com')
        user.set_password('secret')

        assert user.check_password('secret')
        assert not user.check_password('wrong')
```

### Qualité du code

#### `setup.cfg` pour flake8 et black

```ini
[flake8]
max-line-length = 120
exclude = .git,__pycache__,venv,migrations,tests
ignore = E203, W503

[tool:black]
line-length = 120
target-version = ['py38']
include = \.py$
exclude = migrations|venv
```

#### Pré-commit hooks

```yaml
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v4.4.0
    hooks:
      - id: trailing-whitespace
      - id: end-of-file-fixer
      - id: check-yaml
      - id: check-added-large-files

  - repo: https://github.com/psf/black
    rev: 23.11.0
    hooks:
      - id: black
        language_version: python3.8

  - repo: https://github.com/pycqa/flake8
    rev: 6.1.0
    hooks:
      - id: flake8
```

> **💡 À retenir** : Une structure modulaire bien pensée facilite les tests, la maintenance et l'évolution du code. Chaque composant a une responsabilité claire et peut être testé indépendamment.

> **⚠️ Astuce** : Utilisez des fixtures pytest pour créer des données de test réalistes et éviter la duplication de code dans vos tests.

Dans le prochain chapitre, nous concevrons l'**interface utilisateur** avec un dashboard futuriste et des graphiques interactifs !

---

**Navigation**
- [Chapitre précédent : Installation et configuration de Flask](Chapitre_10_Installation_Configuration.md)
- [Chapitre suivant : Conception de l'interface utilisateur](Chapitre_12_Interface_Utilisateur.md)
- [Retour à la table des matières](../../README.md)
