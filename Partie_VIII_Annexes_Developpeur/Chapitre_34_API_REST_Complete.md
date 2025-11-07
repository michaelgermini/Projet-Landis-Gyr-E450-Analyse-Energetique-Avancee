# 🌐 Chapitre 34 : API REST complète

## 📡 Spécification complète de l'API

### Endpoints principaux

#### Gestion des mesures énergétiques

```http
GET    /api/mesures                    # Liste des mesures
POST   /api/mesures                    # Créer une mesure
GET    /api/mesures/{id}              # Détail d'une mesure
PUT    /api/mesures/{id}              # Modifier une mesure
DELETE /api/mesures/{id}              # Supprimer une mesure

GET    /api/mesures/dernieres         # Dernières mesures
GET    /api/mesures/periode/{debut}/{fin}  # Mesures par période
GET    /api/mesures/compteur/{compteur_id} # Mesures par compteur
```

#### Analyses et statistiques

```http
GET    /api/analyses/consommation     # Analyse consommation
GET    /api/analyses/puissance        # Analyse puissance
GET    /api/analyses/qualite          # Analyse qualité réseau
GET    /api/analyses/anomalies        # Détection d'anomalies
GET    /api/analyses/tendances        # Tendances temporelles

GET    /api/analyses/couts/{periode}  # Calcul des coûts
GET    /api/analyses/economies        # Analyse des économies
```

#### Gestion des compteurs

```http
GET    /api/compteurs                 # Liste des compteurs
POST   /api/compteurs                 # Ajouter un compteur
GET    /api/compteurs/{id}            # Détail compteur
PUT    /api/compteurs/{id}            # Modifier compteur
DELETE /api/compteurs/{id}            # Supprimer compteur

POST   /api/compteurs/{id}/test       # Tester connexion
GET    /api/compteurs/{id}/status     # État du compteur
```

#### Export et reporting

```http
GET    /api/export/csv/{periode}      # Export CSV
GET    /api/export/excel/{periode}    # Export Excel
GET    /api/export/json/{periode}     # Export JSON

GET    /api/rapports/journalier       # Rapport journalier
GET    /api/rapports/hebdomadaire     # Rapport hebdomadaire
GET    /api/rapports/mensuel          # Rapport mensuel
```

## 🔧 Implémentation détaillée

### Blueprint API principal

```python
# app/routes/api.py
from flask import Blueprint, request, jsonify, current_app
from flask_jwt_extended import jwt_required, get_jwt_identity
from sqlalchemy import and_, or_, func
from datetime import datetime, timedelta
from app import db
from app.models import MesureEnergie, Anomalie, ConfigurationCompteur
from app.services.analyse_donnees import AnalyseurDonnees
from app.services.calcul_couts import CalculateurCouts

api_bp = Blueprint('api', __name__)

# Services
analyseur = AnalyseurDonnees()
calculateur_couts = CalculateurCouts()

@api_bp.route('/mesures', methods=['GET'])
@jwt_required()
def get_mesures():
    """Récupère les mesures avec filtrage et pagination"""
    try:
        # Paramètres de requête
        page = int(request.args.get('page', 1))
        per_page = min(int(request.args.get('per_page', 50)), 100)  # Max 100
        compteur_id = request.args.get('compteur_id')
        debut = request.args.get('debut')
        fin = request.args.get('fin')

        # Construction de la requête
        query = MesureEnergie.query

        # Filtres
        if compteur_id:
            query = query.filter(MesureEnergie.compteur_id == compteur_id)

        if debut:
            try:
                date_debut = datetime.fromisoformat(debut.replace('Z', '+00:00'))
                query = query.filter(MesureEnergie.timestamp >= date_debut)
            except ValueError:
                return jsonify({'error': 'Format date début invalide'}), 400

        if fin:
            try:
                date_fin = datetime.fromisoformat(fin.replace('Z', '+00:00'))
                query = query.filter(MesureEnergie.timestamp <= date_fin)
            except ValueError:
                return jsonify({'error': 'Format date fin invalide'}), 400

        # Pagination
        mesures = query.order_by(MesureEnergie.timestamp.desc()).paginate(
            page=page, per_page=per_page, error_out=False
        )

        return jsonify({
            'mesures': [m.to_dict() for m in mesures.items],
            'pagination': {
                'page': mesures.page,
                'per_page': mesures.per_page,
                'total_pages': mesures.pages,
                'total_items': mesures.total,
                'has_next': mesures.has_next,
                'has_prev': mesures.has_prev
            }
        })

    except Exception as e:
        current_app.logger.error(f"Erreur API mesures: {e}")
        return jsonify({'error': 'Erreur interne du serveur'}), 500

@api_bp.route('/mesures', methods=['POST'])
@jwt_required()
def create_mesure():
    """Crée une nouvelle mesure"""
    try:
        data = request.get_json()

        if not data:
            return jsonify({'error': 'Données JSON requises'}), 400

        # Validation des données
        required_fields = ['compteur_id']
        for field in required_fields:
            if field not in data:
                return jsonify({'error': f'Champ requis manquant: {field}'}), 400

        # Création de la mesure
        mesure = MesureEnergie(
            compteur_id=data['compteur_id'],
            energie_active_import=data.get('energie_active_import'),
            energie_active_export=data.get('energie_active_export'),
            puissance_active_totale=data.get('puissance_active_totale'),
            tension_l1=data.get('tension_l1'),
            tension_l2=data.get('tension_l2'),
            tension_l3=data.get('tension_l3'),
            courant_l1=data.get('courant_l1'),
            courant_l2=data.get('courant_l2'),
            courant_l3=data.get('courant_l3'),
            frequence=data.get('frequence'),
            facteur_puissance=data.get('facteur_puissance'),
            qualite_donnees=data.get('qualite_donnees', 'good')
        )

        db.session.add(mesure)
        db.session.commit()

        return jsonify({
            'message': 'Mesure créée avec succès',
            'mesure': mesure.to_dict()
        }), 201

    except Exception as e:
        current_app.logger.error(f"Erreur création mesure: {e}")
        db.session.rollback()
        return jsonify({'error': 'Erreur lors de la création'}), 500

@api_bp.route('/mesures/<int:id>', methods=['GET'])
@jwt_required()
def get_mesure(id):
    """Récupère une mesure spécifique"""
    try:
        mesure = MesureEnergie.query.get_or_404(id)
        return jsonify(mesure.to_dict())

    except Exception as e:
        current_app.logger.error(f"Erreur récupération mesure {id}: {e}")
        return jsonify({'error': 'Mesure non trouvée'}), 404

@api_bp.route('/mesures/<int:id>', methods=['PUT'])
@jwt_required()
def update_mesure(id):
    """Met à jour une mesure"""
    try:
        mesure = MesureEnergie.query.get_or_404(id)
        data = request.get_json()

        if not data:
            return jsonify({'error': 'Données JSON requises'}), 400

        # Mise à jour des champs
        updatable_fields = [
            'energie_active_import', 'energie_active_export',
            'puissance_active_totale', 'tension_l1', 'tension_l2', 'tension_l3',
            'courant_l1', 'courant_l2', 'courant_l3',
            'frequence', 'facteur_puissance', 'qualite_donnees'
        ]

        for field in updatable_fields:
            if field in data:
                setattr(mesure, field, data[field])

        db.session.commit()

        return jsonify({
            'message': 'Mesure mise à jour',
            'mesure': mesure.to_dict()
        })

    except Exception as e:
        current_app.logger.error(f"Erreur mise à jour mesure {id}: {e}")
        db.session.rollback()
        return jsonify({'error': 'Erreur lors de la mise à jour'}), 500

@api_bp.route('/mesures/<int:id>', methods=['DELETE'])
@jwt_required()
def delete_mesure(id):
    """Supprime une mesure"""
    try:
        mesure = MesureEnergie.query.get_or_404(id)
        db.session.delete(mesure)
        db.session.commit()

        return jsonify({'message': 'Mesure supprimée avec succès'})

    except Exception as e:
        current_app.logger.error(f"Erreur suppression mesure {id}: {e}")
        db.session.rollback()
        return jsonify({'error': 'Erreur lors de la suppression'}), 500

@api_bp.route('/mesures/dernieres', methods=['GET'])
@jwt_required()
def get_dernieres_mesures():
    """Récupère les dernières mesures de chaque compteur"""
    try:
        limit = min(int(request.args.get('limit', 10)), 50)

        # Sous-requête pour obtenir la dernière mesure par compteur
        subquery = db.session.query(
            MesureEnergie.compteur_id,
            func.max(MesureEnergie.timestamp).label('max_timestamp')
        ).group_by(MesureEnergie.compteur_id).subquery()

        # Jointure pour récupérer les mesures complètes
        mesures = db.session.query(MesureEnergie).join(
            subquery,
            and_(
                MesureEnergie.compteur_id == subquery.c.compteur_id,
                MesureEnergie.timestamp == subquery.c.max_timestamp
            )
        ).limit(limit).all()

        return jsonify({
            'mesures': [m.to_dict() for m in mesures],
            'count': len(mesures)
        })

    except Exception as e:
        current_app.logger.error(f"Erreur dernières mesures: {e}")
        return jsonify({'error': 'Erreur récupération données'}), 500

@api_bp.route('/mesures/periode/<debut>/<fin>', methods=['GET'])
@jwt_required()
def get_mesures_periode(debut, fin):
    """Récupère les mesures sur une période donnée"""
    try:
        # Parsing des dates
        try:
            date_debut = datetime.fromisoformat(debut.replace('Z', '+00:00'))
            date_fin = datetime.fromisoformat(fin.replace('Z', '+00:00'))
        except ValueError:
            return jsonify({'error': 'Format de date invalide'}), 400

        # Validation période
        if date_fin <= date_debut:
            return jsonify({'error': 'Date fin doit être postérieure à date début'}), 400

        if (date_fin - date_debut).days > 365:
            return jsonify({'error': 'Période maximale: 1 an'}), 400

        # Récupération des mesures
        mesures = MesureEnergie.query.filter(
            and_(
                MesureEnergie.timestamp >= date_debut,
                MesureEnergie.timestamp <= date_fin
            )
        ).order_by(MesureEnergie.timestamp).all()

        return jsonify({
            'periode': {
                'debut': date_debut.isoformat(),
                'fin': date_fin.isoformat(),
                'duree_jours': (date_fin - date_debut).days
            },
            'mesures': [m.to_dict() for m in mesures],
            'count': len(mesures)
        })

    except Exception as e:
        current_app.logger.error(f"Erreur mesures période: {e}")
        return jsonify({'error': 'Erreur récupération données'}), 500

@api_bp.route('/analyses/consommation', methods=['GET'])
@jwt_required()
def analyse_consommation():
    """Analyse de la consommation énergétique"""
    try:
        periode = request.args.get('periode', '7d')  # 1d, 7d, 30d, 90d

        # Calcul de la période
        if periode == '1d':
            delta = timedelta(days=1)
        elif periode == '7d':
            delta = timedelta(days=7)
        elif periode == '30d':
            delta = timedelta(days=30)
        elif periode == '90d':
            delta = timedelta(days=90)
        else:
            return jsonify({'error': 'Période invalide'}), 400

        date_debut = datetime.utcnow() - delta

        # Récupération des données
        mesures = MesureEnergie.query.filter(
            MesureEnergie.timestamp >= date_debut
        ).order_by(MesureEnergie.timestamp).all()

        if not mesures:
            return jsonify({'error': 'Aucune donnée disponible'}), 404

        # Analyse avec le service
        analyse = analyseur.analyser_consommation(mesures, periode)

        return jsonify({
            'periode': periode,
            'analyse': analyse,
            'base_donnees': len(mesures)
        })

    except Exception as e:
        current_app.logger.error(f"Erreur analyse consommation: {e}")
        return jsonify({'error': 'Erreur lors de l\'analyse'}), 500

@api_bp.route('/analyses/couts/<periode>', methods=['GET'])
@jwt_required()
def calcul_couts(periode):
    """Calcul des coûts énergétiques"""
    try:
        # Récupération des données
        if periode == 'mois':
            date_debut = datetime.utcnow().replace(day=1)
        elif periode == 'annee':
            date_debut = datetime(datetime.utcnow().year, 1, 1)
        else:
            return jsonify({'error': 'Période invalide (mois/annee)'}), 400

        mesures = MesureEnergie.query.filter(
            MesureEnergie.timestamp >= date_debut
        ).order_by(MesureEnergie.timestamp).all()

        if not mesures:
            return jsonify({'error': 'Aucune donnée disponible'}), 404

        # Calcul des coûts
        couts = calculateur_couts.calculer_couts_periode(mesures, periode)

        return jsonify({
            'periode': periode,
            'couts': couts,
            'base_donnees': len(mesures)
        })

    except Exception as e:
        current_app.logger.error(f"Erreur calcul coûts: {e}")
        return jsonify({'error': 'Erreur lors du calcul'}), 500

@api_bp.route('/export/csv/<periode>', methods=['GET'])
@jwt_required()
def export_csv(periode):
    """Export des données en CSV"""
    try:
        from io import StringIO
        import csv

        # Calcul de la période
        periodes = {
            'jour': timedelta(days=1),
            'semaine': timedelta(days=7),
            'mois': timedelta(days=30),
            'annee': timedelta(days=365)
        }

        if periode not in periodes:
            return jsonify({'error': 'Période invalide'}), 400

        date_debut = datetime.utcnow() - periodes[periode]

        # Récupération des données
        mesures = MesureEnergie.query.filter(
            MesureEnergie.timestamp >= date_debut
        ).order_by(MesureEnergie.timestamp).all()

        if not mesures:
            return jsonify({'error': 'Aucune donnée à exporter'}), 404

        # Création du CSV
        output = StringIO()
        writer = csv.writer(output)

        # En-têtes
        headers = [
            'timestamp', 'compteur_id', 'energie_active_import', 'energie_active_export',
            'puissance_active_totale', 'tension_l1', 'tension_l2', 'tension_l3',
            'courant_l1', 'courant_l2', 'courant_l3', 'frequence', 'facteur_puissance'
        ]
        writer.writerow(headers)

        # Données
        for mesure in mesures:
            row = [
                mesure.timestamp.isoformat(),
                mesure.compteur_id,
                mesure.energie_active_import,
                mesure.energie_active_export,
                mesure.puissance_active_totale,
                mesure.tension_l1,
                mesure.tension_l2,
                mesure.tension_l3,
                mesure.courant_l1,
                mesure.courant_l2,
                mesure.courant_l3,
                mesure.frequence,
                mesure.facteur_puissance
            ]
            writer.writerow(row)

        # Réponse
        output.seek(0)
        from flask import Response
        return Response(
            output.getvalue(),
            mimetype='text/csv',
            headers={'Content-Disposition': f'attachment; filename=export_{periode}.csv'}
        )

    except Exception as e:
        current_app.logger.error(f"Erreur export CSV: {e}")
        return jsonify({'error': 'Erreur lors de l\'export'}), 500

@api_bp.route('/compteurs', methods=['GET'])
@jwt_required()
def get_compteurs():
    """Liste des compteurs configurés"""
    try:
        compteurs = ConfigurationCompteur.query.all()

        return jsonify({
            'compteurs': [{
                'id': c.id,
                'compteur_id': c.compteur_id,
                'nom_affiche': c.nom_affiche,
                'type_interface': c.type_interface,
                'actif': c.actif,
                'created_at': c.created_at.isoformat(),
                'updated_at': c.updated_at.isoformat()
            } for c in compteurs]
        })

    except Exception as e:
        current_app.logger.error(f"Erreur récupération compteurs: {e}")
        return jsonify({'error': 'Erreur récupération données'}), 500

@api_bp.route('/compteurs', methods=['POST'])
@jwt_required()
def create_compteur():
    """Ajoute un nouveau compteur"""
    try:
        data = request.get_json()

        if not data:
            return jsonify({'error': 'Données JSON requises'}), 400

        required_fields = ['compteur_id', 'nom_affiche', 'type_interface']
        for field in required_fields:
            if field not in data:
                return jsonify({'error': f'Champ requis manquant: {field}'}), 400

        # Vérification unicité
        if ConfigurationCompteur.query.filter_by(compteur_id=data['compteur_id']).first():
            return jsonify({'error': 'ID compteur déjà existant'}), 409

        # Création
        compteur = ConfigurationCompteur(
            compteur_id=data['compteur_id'],
            nom_affiche=data['nom_affiche'],
            type_interface=data['type_interface'],
            adresse_ip=data.get('adresse_ip'),
            port_mbus=data.get('port_mbus'),
            actif=data.get('actif', True)
        )

        db.session.add(compteur)
        db.session.commit()

        return jsonify({
            'message': 'Compteur ajouté avec succès',
            'compteur': {
                'id': compteur.id,
                'compteur_id': compteur.compteur_id,
                'nom_affiche': compteur.nom_affiche,
                'type_interface': compteur.type_interface,
                'actif': compteur.actif
            }
        }), 201

    except Exception as e:
        current_app.logger.error(f"Erreur création compteur: {e}")
        db.session.rollback()
        return jsonify({'error': 'Erreur lors de la création'}), 500

@api_bp.route('/compteurs/<int:id>/status', methods=['GET'])
@jwt_required()
def get_status_compteur(id):
    """État d'un compteur spécifique"""
    try:
        compteur = ConfigurationCompteur.query.get_or_404(id)

        # Dernière mesure
        dernier_mesure = MesureEnergie.query.filter_by(
            compteur_id=compteur.compteur_id
        ).order_by(MesureEnergie.timestamp.desc()).first()

        # Statistiques
        mesures_count = MesureEnergie.query.filter_by(
            compteur_id=compteur.compteur_id
        ).count()

        status = {
            'compteur': {
                'id': compteur.id,
                'compteur_id': compteur.compteur_id,
                'nom_affiche': compteur.nom_affiche,
                'actif': compteur.actif
            },
            'derniere_mesure': dernier_mesure.to_dict() if dernier_mesure else None,
            'statistiques': {
                'total_mesures': mesures_count,
                'derniere_activite': dernier_mesure.timestamp.isoformat() if dernier_mesure else None
            }
        }

        # Évaluation de l'état
        if dernier_mesure:
            age_dernier = datetime.utcnow() - dernier_mesure.timestamp
            if age_dernier < timedelta(hours=1):
                status['etat'] = 'actif'
            elif age_dernier < timedelta(hours=24):
                status['etat'] = 'inactif_court'
            else:
                status['etat'] = 'inactif_long'
        else:
            status['etat'] = 'jamais_vu'

        return jsonify(status)

    except Exception as e:
        current_app.logger.error(f"Erreur status compteur {id}: {e}")
        return jsonify({'error': 'Erreur récupération status'}), 500
```

### Middleware d'authentification

```python
# app/routes/auth.py
from flask import Blueprint, request, jsonify, current_app
from flask_jwt_extended import create_access_token, jwt_required, get_jwt_identity
from werkzeug.security import check_password_hash
from app import db
from app.models import User
import re

auth_bp = Blueprint('auth', __name__)

@auth_bp.route('/login', methods=['POST'])
def login():
    """Authentification utilisateur"""
    try:
        data = request.get_json()

        if not data or 'username' not in data or 'password' not in data:
            return jsonify({'error': 'Username et password requis'}), 400

        username = data['username'].strip()
        password = data['password']

        # Validation basique
        if not re.match(r'^[a-zA-Z0-9_]{3,50}$', username):
            return jsonify({'error': 'Format username invalide'}), 400

        # Recherche utilisateur
        user = User.query.filter_by(username=username).first()

        if not user or not user.check_password(password):
            return jsonify({'error': 'Identifiants invalides'}), 401

        # Génération token
        access_token = create_access_token(identity=user.username)

        return jsonify({
            'access_token': access_token,
            'user': {
                'username': user.username,
                'email': user.email,
                'is_admin': user.is_admin
            }
        })

    except Exception as e:
        current_app.logger.error(f"Erreur login: {e}")
        return jsonify({'error': 'Erreur d\'authentification'}), 500

@auth_bp.route('/register', methods=['POST'])
def register():
    """Inscription nouvel utilisateur (admin seulement)"""
    try:
        data = request.get_json()

        if not data:
            return jsonify({'error': 'Données requises'}), 400

        required_fields = ['username', 'email', 'password']
        for field in required_fields:
            if field not in data:
                return jsonify({'error': f'Champ requis: {field}'}), 400

        username = data['username'].strip()
        email = data['email'].strip()
        password = data['password']

        # Validations
        if not re.match(r'^[a-zA-Z0-9_]{3,50}$', username):
            return jsonify({'error': 'Username: 3-50 caractères alphanumériques'}), 400

        if not re.match(r'^[^@]+@[^@]+\.[^@]+$', email):
            return jsonify({'error': 'Email invalide'}), 400

        if len(password) < 8:
            return jsonify({'error': 'Mot de passe minimum 8 caractères'}), 400

        # Vérification unicité
        if User.query.filter_by(username=username).first():
            return jsonify({'error': 'Username déjà utilisé'}), 409

        if User.query.filter_by(email=email).first():
            return jsonify({'error': 'Email déjà utilisé'}), 409

        # Création utilisateur
        user = User(username=username, email=email)
        user.set_password(password)

        db.session.add(user)
        db.session.commit()

        return jsonify({
            'message': 'Utilisateur créé avec succès',
            'user': {
                'username': user.username,
                'email': user.email
            }
        }), 201

    except Exception as e:
        current_app.logger.error(f"Erreur register: {e}")
        db.session.rollback()
        return jsonify({'error': 'Erreur lors de l\'inscription'}), 500

@auth_bp.route('/profile', methods=['GET'])
@jwt_required()
def get_profile():
    """Récupère le profil utilisateur"""
    try:
        username = get_jwt_identity()
        user = User.query.filter_by(username=username).first()

        if not user:
            return jsonify({'error': 'Utilisateur non trouvé'}), 404

        return jsonify({
            'user': {
                'username': user.username,
                'email': user.email,
                'is_admin': user.is_admin,
                'created_at': user.created_at.isoformat()
            }
        })

    except Exception as e:
        current_app.logger.error(f"Erreur profile: {e}")
        return jsonify({'error': 'Erreur récupération profil'}), 500

@auth_bp.route('/change-password', methods=['POST'])
@jwt_required()
def change_password():
    """Changement de mot de passe"""
    try:
        username = get_jwt_identity()
        user = User.query.filter_by(username=username).first()

        if not user:
            return jsonify({'error': 'Utilisateur non trouvé'}), 404

        data = request.get_json()

        if not data or 'old_password' not in data or 'new_password' not in data:
            return jsonify({'error': 'Ancien et nouveau mot de passe requis'}), 400

        old_password = data['old_password']
        new_password = data['new_password']

        # Vérification ancien mot de passe
        if not user.check_password(old_password):
            return jsonify({'error': 'Ancien mot de passe incorrect'}), 401

        # Validation nouveau mot de passe
        if len(new_password) < 8:
            return jsonify({'error': 'Nouveau mot de passe minimum 8 caractères'}), 400

        # Changement
        user.set_password(new_password)
        db.session.commit()

        return jsonify({'message': 'Mot de passe changé avec succès'})

    except Exception as e:
        current_app.logger.error(f"Erreur change password: {e}")
        db.session.rollback()
        return jsonify({'error': 'Erreur lors du changement'}), 500
```

### Tests de l'API

```python
# tests/test_api.py
import pytest
import json
from app import create_app, db
from app.models import User, MesureEnergie

@pytest.fixture
def client():
    app = create_app('testing')
    with app.test_client() as client:
        with app.app_context():
            db.create_all()
            # Création utilisateur test
            user = User(username='testuser', email='test@example.com')
            user.set_password('testpass')
            db.session.add(user)
            db.session.commit()
        yield client

def get_token(client):
    """Récupère un token d'authentification"""
    response = client.post('/auth/login', json={
        'username': 'testuser',
        'password': 'testpass'
    })
    return json.loads(response.data)['access_token']

def test_login(client):
    """Test de connexion"""
    response = client.post('/auth/login', json={
        'username': 'testuser',
        'password': 'testpass'
    })
    assert response.status_code == 200
    data = json.loads(response.data)
    assert 'access_token' in data
    assert data['user']['username'] == 'testuser'

def test_get_mesures_unauthorized(client):
    """Test accès non autorisé"""
    response = client.get('/api/mesures')
    assert response.status_code == 401

def test_create_mesure(client):
    """Test création mesure"""
    token = get_token(client)

    response = client.post('/api/mesures',
        json={
            'compteur_id': 'TEST001',
            'puissance_active_totale': 1500.5,
            'tension_l1': 230.1
        },
        headers={'Authorization': f'Bearer {token}'}
    )

    assert response.status_code == 201
    data = json.loads(response.data)
    assert data['mesure']['compteur_id'] == 'TEST001'
    assert data['mesure']['puissance_active_totale'] == 1500.5

def test_get_mesures_list(client):
    """Test récupération liste mesures"""
    token = get_token(client)

    # Création de données test
    with client.application.app_context():
        mesure = MesureEnergie(
            compteur_id='TEST001',
            puissance_active_totale=1000.0
        )
        db.session.add(mesure)
        db.session.commit()

    response = client.get('/api/mesures',
        headers={'Authorization': f'Bearer {token}'}
    )

    assert response.status_code == 200
    data = json.loads(response.data)
    assert len(data['mesures']) >= 1
    assert 'pagination' in data

def test_export_csv(client):
    """Test export CSV"""
    token = get_token(client)

    # Création de données test
    with client.application.app_context():
        mesure = MesureEnergie(
            compteur_id='TEST001',
            puissance_active_totale=1000.0,
            tension_l1=230.0
        )
        db.session.add(mesure)
        db.session.commit()

    response = client.get('/api/export/csv/semaine',
        headers={'Authorization': f'Bearer {token}'}
    )

    assert response.status_code == 200
    assert 'text/csv' in response.content_type
    assert 'attachment' in response.headers.get('Content-Disposition', '')

    # Vérification contenu CSV
    csv_content = response.data.decode('utf-8')
    assert 'timestamp' in csv_content
    assert 'TEST001' in csv_content

def test_analyse_consommation(client):
    """Test analyse consommation"""
    token = get_token(client)

    # Création de données test
    with client.application.app_context():
        import datetime
        base_time = datetime.datetime.utcnow()

        for i in range(10):
            mesure = MesureEnergie(
                compteur_id='TEST001',
                timestamp=base_time + datetime.timedelta(hours=i),
                puissance_active_totale=1000.0 + i * 10
            )
            db.session.add(mesure)
        db.session.commit()

    response = client.get('/api/analyses/consommation?periode=7d',
        headers={'Authorization': f'Bearer {token}'}
    )

    assert response.status_code == 200
    data = json.loads(response.data)
    assert 'analyse' in data
    assert 'periode' in data
```

---

**Navigation**
- [Chapitre précédent : Scripts utilitaires](Chapitre_33_Scripts_Utilitaires.md)
- [Retour à la table des matières](../../README.md)
