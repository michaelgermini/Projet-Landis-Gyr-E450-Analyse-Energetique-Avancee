# 🔌 Chapitre 19 : Historique des anomalies détectées

## 📊 Stockage et gestion de l'historique

L'historique des anomalies permet d'analyser les tendances et d'optimiser la détection au fil du temps.

### Modèle de données pour l'historique

```python
# app/models.py (extension)
from sqlalchemy import func

class HistoriqueAnomalies(db.Model):
    """Historique détaillé des anomalies détectées"""
    __tablename__ = 'historique_anomalies'

    id = db.Column(db.Integer, primary_key=True)
    timestamp_detection = db.Column(db.DateTime, default=datetime.utcnow, index=True)

    # Référence à la mesure qui a déclenché l'anomalie
    mesure_id = db.Column(db.Integer, db.ForeignKey('mesures_energie.id'))

    # Classification de l'anomalie
    type_anomalie = db.Column(db.String(50), nullable=False, index=True)
    categorie = db.Column(db.String(30), default='mesure')
    severite = db.Column(db.String(20), nullable=False, index=True)

    # Description détaillée
    titre = db.Column(db.String(100), nullable=False)
    description = db.Column(db.Text, nullable=False)

    # Données techniques
    valeur_mesuree = db.Column(db.Float)
    unite_mesure = db.Column(db.String(10))
    seuil_declencheur = db.Column(db.Float)
    code_obis = db.Column(db.String(20))

    # Contexte
    algorithme_detection = db.Column(db.String(50), default='seuils_statiques')
    confiance_detection = db.Column(db.Float)  # 0-1, niveau de confiance

    # Actions entreprises
    alerte_envoyee = db.Column(db.Boolean, default=False)
    timestamp_alerte = db.Column(db.DateTime)

    # Statut et résolution
    statut = db.Column(db.String(20), default='detectee')  # detectee, analysee, resolue, ignoree
    resolue_par = db.Column(db.String(50))  # utilisateur ou automatique
    timestamp_resolution = db.Column(db.DateTime)
    commentaires_resolution = db.Column(db.Text)

    # Métadonnées
    version_algorithme = db.Column(db.String(20), default='1.0')
    parametres_detection = db.Column(db.Text)  # JSON des paramètres utilisés

    # Index pour les performances
    __table_args__ = (
        db.Index('idx_anomalie_type_timestamp', 'type_anomalie', 'timestamp_detection'),
        db.Index('idx_anomalie_severite_statut', 'severite', 'statut'),
    )

    def to_dict(self):
        """Conversion pour l'API"""
        return {
            'id': self.id,
            'timestamp_detection': self.timestamp_detection.isoformat() + 'Z',
            'type_anomalie': self.type_anomalie,
            'categorie': self.categorie,
            'severite': self.severite,
            'titre': self.titre,
            'description': self.description,
            'valeur_mesuree': self.valeur_mesuree,
            'unite_mesure': self.unite_mesure,
            'seuil_declencheur': self.seuil_declencheur,
            'code_obis': self.code_obis,
            'statut': self.statut,
            'alerte_envoyee': self.alerte_envoyee,
            'resolue_par': self.resolue_par,
            'timestamp_resolution': self.timestamp_resolution.isoformat() + 'Z' if self.timestamp_resolution else None,
            'algorithme_detection': self.algorithme_detection,
            'confiance_detection': self.confiance_detection
        }

    def marquer_resolue(self, resolue_par: str, commentaires: str = None):
        """Marque l'anomalie comme résolue"""
        self.statut = 'resolue'
        self.resolue_par = resolue_par
        self.timestamp_resolution = datetime.utcnow()
        self.commentaires_resolution = commentaires

    def ignorer(self, raison: str = None):
        """Marque l'anomalie comme ignorée"""
        self.statut = 'ignoree'
        self.resolue_par = 'automatique'
        self.timestamp_resolution = datetime.utcnow()
        self.commentaires_resolution = raison or 'Ignorée automatiquement'

    def __repr__(self):
        return f'<Anomalie {self.type_anomalie} {self.severite} {self.timestamp_detection}>'

class StatistiquesAnomalies(db.Model):
    """Statistiques agrégées des anomalies"""
    __tablename__ = 'statistiques_anomalies'

    id = db.Column(db.Integer, primary_key=True)
    date = db.Column(db.Date, nullable=False, index=True, unique=True)

    # Comptages par type
    nb_anomalies_total = db.Column(db.Integer, default=0)
    nb_anomalies_info = db.Column(db.Integer, default=0)
    nb_anomalies_warning = db.Column(db.Integer, default=0)
    nb_anomalies_error = db.Column(db.Integer, default=0)
    nb_anomalies_critical = db.Column(db.Integer, default=0)

    # Par catégorie
    nb_anomalies_mesure = db.Column(db.Integer, default=0)
    nb_anomalies_reseau = db.Column(db.Integer, default=0)
    nb_anomalies_systeme = db.Column(db.Integer, default=0)

    # Taux de résolution
    nb_anomalies_resolues = db.Column(db.Integer, default=0)
    nb_anomalies_ignorees = db.Column(db.Integer, default=0)

    # Temps de résolution moyen (minutes)
    temps_resolution_moyen = db.Column(db.Float)

    # Types les plus fréquents (JSON)
    types_frequents = db.Column(db.Text)  # JSON string

    def to_dict(self):
        """Conversion pour l'API"""
        return {
            'date': self.date.isoformat(),
            'total': self.nb_anomalies_total,
            'par_severite': {
                'info': self.nb_anomalies_info,
                'warning': self.nb_anomalies_warning,
                'error': self.nb_anomalies_error,
                'critical': self.nb_anomalies_critical
            },
            'par_categorie': {
                'mesure': self.nb_anomalies_mesure,
                'reseau': self.nb_anomalies_reseau,
                'systeme': self.nb_anomalies_systeme
            },
            'resolution': {
                'resolues': self.nb_anomalies_resolues,
                'ignorees': self.nb_anomalies_ignorees,
                'taux_resolution': (self.nb_anomalies_resolues / self.nb_anomalies_total * 100) if self.nb_anomalies_total > 0 else 0,
                'temps_moyen': self.temps_resolution_moyen
            },
            'types_frequents': json.loads(self.types_frequents) if self.types_frequents else {}
        }
```

### Service de gestion de l'historique

```python
# app/services/historique_anomalies.py
from app.models import HistoriqueAnomalies, StatistiquesAnomalies, db
from datetime import datetime, timedelta, date
import json
from collections import Counter
from typing import List, Dict, Optional

class GestionnaireHistoriqueAnomalies:
    """Gestionnaire de l'historique des anomalies"""

    def __init__(self):
        self.retention_jours = 365  # Conservation 1 an

    def enregistrer_anomalie(self, anomalie: Dict, mesure_id: int = None,
                           algorithme: str = 'seuils_statiques',
                           confiance: float = 0.8) -> HistoriqueAnomalies:
        """
        Enregistre une nouvelle anomalie dans l'historique

        Args:
            anomalie: Dictionnaire de l'anomalie détectée
            mesure_id: ID de la mesure associée
            algorithme: Algorithme de détection utilisé
            confiance: Niveau de confiance (0-1)

        Returns:
            Instance HistoriqueAnomalies créée
        """
        try:
            # Création de l'enregistrement
            hist_anomalie = HistoriqueAnomalies(
                mesure_id=mesure_id,
                type_anomalie=anomalie['type'],
                categorie=anomalie.get('categorie', 'mesure'),
                severite=anomalie['severite'],
                titre=anomalie['titre'],
                description=anomalie['description'],
                valeur_mesuree=anomalie.get('valeur'),
                unite_mesure=anomalie.get('unite'),
                seuil_declencheur=anomalie.get('seuil'),
                code_obis=anomalie.get('code_obis'),
                algorithme_detection=algorithme,
                confiance_detection=confiance,
                parametres_detection=json.dumps(anomalie)
            )

            db.session.add(hist_anomalie)
            db.session.commit()

            return hist_anomalie

        except Exception as e:
            db.session.rollback()
            raise Exception(f"Erreur enregistrement anomalie: {e}")

    def mettre_a_jour_statut(self, anomalie_id: int, statut: str,
                           resolue_par: str = None, commentaires: str = None):
        """
        Met à jour le statut d'une anomalie

        Args:
            anomalie_id: ID de l'anomalie
            statut: Nouveau statut (resolue, ignoree)
            resolue_par: Qui a résolu
            commentaires: Commentaires
        """
        anomalie = HistoriqueAnomalies.query.get(anomalie_id)
        if not anomalie:
            raise ValueError(f"Anomalie {anomalie_id} introuvable")

        if statut == 'resolue':
            anomalie.marquer_resolue(resolue_par or 'utilisateur', commentaires)
        elif statut == 'ignoree':
            anomalie.ignorer(commentaires)
        else:
            raise ValueError(f"Statut invalide: {statut}")

        db.session.commit()

    def nettoyer_historique(self):
        """Nettoie l'historique ancien selon la politique de rétention"""
        date_limite = datetime.utcnow() - timedelta(days=self.retention_jours)

        # Suppression des anomalies anciennes
        nb_supprimees = HistoriqueAnomalies.query.filter(
            HistoriqueAnomalies.timestamp_detection < date_limite
        ).delete()

        db.session.commit()

        return nb_supprimees

    def calculer_statistiques_journalieres(self, date_calcul: date = None):
        """
        Calcule les statistiques journalières des anomalies

        Args:
            date_calcul: Date à calculer (défaut: hier)
        """
        if date_calcul is None:
            date_calcul = (datetime.utcnow() - timedelta(days=1)).date()

        debut_journee = datetime.combine(date_calcul, datetime.min.time())
        fin_journee = datetime.combine(date_calcul, datetime.max.time())

        # Récupération des anomalies du jour
        anomalies = HistoriqueAnomalies.query.filter(
            HistoriqueAnomalies.timestamp_detection.between(debut_journee, fin_journee)
        ).all()

        if not anomalies:
            return  # Pas d'anomalies ce jour

        # Calculs des statistiques
        stats = {
            'nb_anomalies_total': len(anomalies),
            'nb_anomalies_info': len([a for a in anomalies if a.severite == 'info']),
            'nb_anomalies_warning': len([a for a in anomalies if a.severite == 'warning']),
            'nb_anomalies_error': len([a for a in anomalies if a.severite == 'error']),
            'nb_anomalies_critical': len([a for a in anomalies if a.severite == 'critical']),
            'nb_anomalies_mesure': len([a for a in anomalies if a.categorie == 'mesure']),
            'nb_anomalies_reseau': len([a for a in anomalies if a.categorie == 'reseau']),
            'nb_anomalies_systeme': len([a for a in anomalies if a.categorie == 'systeme']),
            'nb_anomalies_resolues': len([a for a in anomalies if a.statut == 'resolue']),
            'nb_anomalies_ignorees': len([a for a in anomalies if a.statut == 'ignoree']),
        }

        # Temps de résolution moyen
        temps_resolution = []
        for anomalie in anomalies:
            if anomalie.timestamp_resolution and anomalie.timestamp_detection:
                temps = (anomalie.timestamp_resolution - anomalie.timestamp_detection).total_seconds() / 60
                temps_resolution.append(temps)

        stats['temps_resolution_moyen'] = statistics.mean(temps_resolution) if temps_resolution else None

        # Types les plus fréquents
        types_counter = Counter([a.type_anomalie for a in anomalies])
        stats['types_frequents'] = dict(types_counter.most_common(5))

        # Mise à jour ou création des statistiques
        stat_existante = StatistiquesAnomalies.query.filter_by(date=date_calcul).first()

        if stat_existante:
            # Mise à jour
            for key, value in stats.items():
                setattr(stat_existante, key, value)
        else:
            # Création
            stat_existante = StatistiquesAnomalies(date=date_calcul, **stats)
            db.session.add(stat_existante)

        db.session.commit()

    def analyser_tendances_anomalies(self, periode_jours: int = 30) -> Dict:
        """
        Analyse les tendances des anomalies sur une période

        Args:
            periode_jours: Nombre de jours à analyser

        Returns:
            Analyse des tendances
        """
        date_limite = (datetime.utcnow() - timedelta(days=periode_jours)).date()

        # Récupération des statistiques
        stats = StatistiquesAnomalies.query.filter(
            StatistiquesAnomalies.date >= date_limite
        ).order_by(StatistiquesAnomalies.date).all()

        if not stats:
            return {'erreur': 'Pas assez de données'}

        # Tendances générales
        total_anomalies = sum(s.nb_anomalies_total for s in stats)
        jours_avec_anomalies = len([s for s in stats if s.nb_anomalies_total > 0])

        # Évolution par jour
        evolution = [{
            'date': s.date.isoformat(),
            'total': s.nb_anomalies_total,
            'critiques': s.nb_anomalies_critical + s.nb_anomalies_error,
            'resolus': s.nb_anomalies_resolues
        } for s in stats]

        # Types les plus problématiques
        tous_types = {}
        for s in stats:
            if s.types_frequents:
                types_dict = json.loads(s.types_frequents)
                for type_anomalie, count in types_dict.items():
                    tous_types[type_anomalie] = tous_types.get(type_anomalie, 0) + count

        types_problem = sorted(tous_types.items(), key=lambda x: x[1], reverse=True)[:5]

        # Taux de résolution global
        total_resolus = sum(s.nb_anomalies_resolues for s in stats)
        taux_resolution_global = (total_resolus / total_anomalies * 100) if total_anomalies > 0 else 0

        return {
            'periode_analyse': f"{periode_jours} jours",
            'total_anomalies': total_anomalies,
            'moyenne_par_jour': total_anomalies / periode_jours,
            'jours_avec_anomalies': jours_avec_anomalies,
            'taux_activite': jours_avec_anomalies / periode_jours * 100,
            'evolution': evolution,
            'types_problem': types_problem,
            'taux_resolution_global': taux_resolution_global,
            'recommandations': self._generer_recommandations(tous_types, taux_resolution_global)
        }

    def _generer_recommandations(self, types_frequents: Dict[str, int],
                               taux_resolution: float) -> List[str]:
        """Génère des recommandations basées sur l'analyse"""
        recommandations = []

        # Recommandations par type fréquent
        if 'pic_consommation' in types_frequents:
            recommandations.append("Considérer l'ajout d'équipements de régulation de puissance")

        if 'tension_anormale' in types_frequents:
            recommandations.append("Vérifier la stabilité du réseau électrique")

        if 'frequence_instable' in types_frequents:
            recommandations.append("Investiguer les perturbations du réseau")

        # Recommandations par taux de résolution
        if taux_resolution < 50:
            recommandations.append("Améliorer le processus de résolution des anomalies")

        if not recommandations:
            recommandations.append("Système d'anomalies fonctionnant correctement")

        return recommandations

    def exporter_historique(self, format_export: str = 'json',
                          periode_jours: int = 30) -> str:
        """
        Exporte l'historique des anomalies

        Args:
            format_export: Format d'export ('json', 'csv')
            periode_jours: Période à exporter

        Returns:
            Données exportées (string)
        """
        date_limite = datetime.utcnow() - timedelta(days=periode_jours)

        anomalies = HistoriqueAnomalies.query.filter(
            HistoriqueAnomalies.timestamp_detection >= date_limite
        ).order_by(HistoriqueAnomalies.timestamp_detection).all()

        if format_export == 'json':
            data = [a.to_dict() for a in anomalies]
            return json.dumps(data, indent=2, ensure_ascii=False)

        elif format_export == 'csv':
            import csv
            from io import StringIO

            output = StringIO()
            writer = csv.DictWriter(output, fieldnames=[
                'id', 'timestamp_detection', 'type_anomalie', 'severite',
                'titre', 'description', 'valeur_mesuree', 'statut'
            ])

            writer.writeheader()
            for anomalie in anomalies:
                writer.writerow({
                    'id': anomalie.id,
                    'timestamp_detection': anomalie.timestamp_detection.isoformat(),
                    'type_anomalie': anomalie.type_anomalie,
                    'severite': anomalie.severite,
                    'titre': anomalie.titre,
                    'description': anomalie.description,
                    'valeur_mesuree': anomalie.valeur_mesuree,
                    'statut': anomalie.statut
                })

            return output.getvalue()

        else:
            raise ValueError(f"Format d'export non supporté: {format_export}")
```

### API pour l'historique

```python
# app/routes/api_historique.py
from flask import Blueprint, request, jsonify, send_file
from app.services.historique_anomalies import GestionnaireHistoriqueAnomalies
from app.models import HistoriqueAnomalies, StatistiquesAnomalies
from datetime import datetime, timedelta
from io import BytesIO

historique_bp = Blueprint('historique', __name__)
gestionnaire_historique = GestionnaireHistoriqueAnomalies()

@historique_bp.route('/api/anomalies/historique')
def get_historique_anomalies():
    """Récupération de l'historique des anomalies"""

    # Paramètres de filtrage
    limit = int(request.args.get('limit', 100))
    offset = int(request.args.get('offset', 0))
    type_anomalie = request.args.get('type')
    severite = request.args.get('severite')
    statut = request.args.get('statut')
    jours = int(request.args.get('days', 30))

    # Construction de la requête
    query = HistoriqueAnomalies.query

    # Filtre de période
    date_limite = datetime.utcnow() - timedelta(days=jours)
    query = query.filter(HistoriqueAnomalies.timestamp_detection >= date_limite)

    # Filtres optionnels
    if type_anomalie:
        query = query.filter_by(type_anomalie=type_anomalie)
    if severite:
        query = query.filter_by(severite=severite)
    if statut:
        query = query.filter_by(statut=statut)

    # Pagination
    anomalies = query.order_by(HistoriqueAnomalies.timestamp_detection.desc())\
                    .offset(offset).limit(limit).all()

    # Statistiques
    total = query.count()

    return jsonify({
        'total': total,
        'limit': limit,
        'offset': offset,
        'anomalies': [a.to_dict() for a in anomalies]
    })

@historique_bp.route('/api/anomalies/statistiques')
def get_statistiques_anomalies():
    """Statistiques des anomalies"""

    jours = int(request.args.get('days', 30))
    date_limite = (datetime.utcnow() - timedelta(days=jours)).date()

    stats = StatistiquesAnomalies.query.filter(
        StatistiquesAnomalies.date >= date_limite
    ).order_by(StatistiquesAnomalies.date).all()

    return jsonify({
        'periode': f"{jours} jours",
        'nb_jours': len(stats),
        'statistiques': [s.to_dict() for s in stats]
    })

@historique_bp.route('/api/anomalies/tendances')
def get_tendances_anomalies():
    """Analyse des tendances des anomalies"""

    jours = int(request.args.get('days', 30))
    tendances = gestionnaire_historique.analyser_tendances_anomalies(jours)

    return jsonify(tendances)

@historique_bp.route('/api/anomalies/<int:anomalie_id>/statut', methods=['PUT'])
def update_statut_anomalie(anomalie_id):
    """Mise à jour du statut d'une anomalie"""

    data = request.get_json()
    if not data or 'statut' not in data:
        return jsonify({'error': 'statut requis'}), 400

    try:
        gestionnaire_historique.mettre_a_jour_statut(
            anomalie_id=anomalie_id,
            statut=data['statut'],
            resolue_par=data.get('resolue_par', 'api'),
            commentaires=data.get('commentaires')
        )

        return jsonify({'message': 'Statut mis à jour'})

    except ValueError as e:
        return jsonify({'error': str(e)}), 404
    except Exception as e:
        return jsonify({'error': str(e)}), 500

@historique_bp.route('/api/anomalies/export')
def export_historique():
    """Export de l'historique des anomalies"""

    format_export = request.args.get('format', 'json')
    jours = int(request.args.get('days', 30))

    try:
        data = gestionnaire_historique.exporter_historique(format_export, jours)

        # Création du fichier en mémoire
        buffer = BytesIO()
        buffer.write(data.encode('utf-8'))
        buffer.seek(0)

        filename = f"anomalies_{datetime.utcnow().strftime('%Y%m%d_%H%M%S')}.{format_export}"

        return send_file(
            buffer,
            as_attachment=True,
            download_name=filename,
            mimetype='application/json' if format_export == 'json' else 'text/csv'
        )

    except Exception as e:
        return jsonify({'error': str(e)}), 500

@historique_bp.route('/api/anomalies/maintenance', methods=['POST'])
def maintenance_historique():
    """Maintenance de l'historique (nettoyage + statistiques)"""

    try:
        # Nettoyage des anciennes anomalies
        nb_supprimees = gestionnaire_historique.nettoyer_historique()

        # Recalcul des statistiques pour les derniers jours
        for i in range(7):  # 7 derniers jours
            date_calcul = (datetime.utcnow() - timedelta(days=i)).date()
            gestionnaire_historique.calculer_statistiques_journalieres(date_calcul)

        return jsonify({
            'message': 'Maintenance terminée',
            'anomalies_supprimees': nb_supprimees,
            'statistiques_recalculees': 7
        })

    except Exception as e:
        return jsonify({'error': str(e)}), 500
```

### Interface de gestion des anomalies

```html
<!-- templates/anomalies_historique.html -->
{% extends "base.html" %}

{% block title %}Historique des Anomalies - Compteur E450{% endblock %}

{% block extra_head %}
<link rel="stylesheet" href="{{ url_for('static', filename='css/anomalies.css') }}">
{% endblock %}

{% block content %}
<div class="anomalies-container">
    <!-- Filtres -->
    <div class="filters-section">
        <div class="filter-group">
            <label>Période:</label>
            <select id="period-select">
                <option value="7">7 jours</option>
                <option value="30" selected>30 jours</option>
                <option value="90">90 jours</option>
            </select>
        </div>

        <div class="filter-group">
            <label>Type:</label>
            <select id="type-select">
                <option value="">Tous</option>
                <option value="pic_consommation">Pic consommation</option>
                <option value="tension_anormale">Tension anormale</option>
                <option value="frequence_instable">Fréquence instable</option>
            </select>
        </div>

        <div class="filter-group">
            <label>Sévérité:</label>
            <select id="severity-select">
                <option value="">Toutes</option>
                <option value="info">Info</option>
                <option value="warning">Warning</option>
                <option value="error">Error</option>
                <option value="critical">Critical</option>
            </select>
        </div>

        <button id="export-btn" class="btn btn-secondary">Exporter</button>
    </div>

    <!-- Statistiques -->
    <div class="stats-section">
        <div class="stat-card">
            <div class="stat-title">Total Anomalies</div>
            <div class="stat-value" id="total-anomalies">-</div>
        </div>

        <div class="stat-card">
            <div class="stat-title">Taux Résolution</div>
            <div class="stat-value" id="resolution-rate">-%</div>
        </div>

        <div class="stat-card">
            <div class="stat-title">Type Principal</div>
            <div class="stat-value" id="main-type">-</div>
        </div>
    </div>

    <!-- Liste des anomalies -->
    <div class="anomalies-list" id="anomalies-list">
        <div class="loading">Chargement des anomalies...</div>
    </div>

    <!-- Pagination -->
    <div class="pagination" id="pagination">
        <!-- Générée dynamiquement -->
    </div>
</div>

<!-- Modal de détails d'anomalie -->
<div class="modal fade" id="anomalie-modal" tabindex="-1">
    <div class="modal-dialog modal-lg">
        <div class="modal-content">
            <div class="modal-header">
                <h5 class="modal-title" id="anomalie-title">Détails de l'anomalie</h5>
                <button type="button" class="btn-close" data-bs-dismiss="modal"></button>
            </div>
            <div class="modal-body" id="anomalie-details">
                <!-- Contenu chargé dynamiquement -->
            </div>
            <div class="modal-footer">
                <button type="button" class="btn btn-secondary" data-bs-dismiss="modal">Fermer</button>
                <button type="button" class="btn btn-success" id="resolve-btn">Marquer résolue</button>
                <button type="button" class="btn btn-warning" id="ignore-btn">Ignorer</button>
            </div>
        </div>
    </div>
</div>
{% endblock %}

{% block extra_scripts %}
<script src="{{ url_for('static', filename='js/anomalies_historique.js') }}"></script>
{% endblock %}
```

> **💡 À retenir** : L'historique des anomalies permet non seulement de suivre les problèmes passés, mais aussi d'optimiser les algorithmes de détection et d'anticiper les pannes.

> **⚠️ Astuce** : Implémentez une politique de rétention des données adaptée à vos besoins - conserver trop d'historique peut ralentir les requêtes, pas assez empêche l'analyse des tendances.

Félicitations ! La Partie IV sur la connectivité et l'intégration intelligente est maintenant complète. Vous maîtrisez maintenant l'envoi de données vers Home Assistant, l'export InfluxDB/Grafana, les calculs de coûts, la détection d'anomalies et leur historique. La Partie V va explorer l'analyse et la visualisation avancée !

---

**Navigation**
- [Chapitre précédent : Détection automatique d'anomalies](Chapitre_18_Detection_Anomalies.md)
- [Partie V : Analyse & visualisation avancée](../Partie_V_Analyse_Visualisation/)
- [Retour à la table des matières](../../README.md)
