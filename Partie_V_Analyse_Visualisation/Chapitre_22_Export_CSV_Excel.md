# 📊 Chapitre 22 : Export CSV/Excel

## 📤 Export des données pour analyse externe

L'export de vos données permet de les analyser avec des outils spécialisés comme Excel, R, ou des logiciels de business intelligence.

### Service d'export de données

```python
# app/services/export_service.py
from app.models import MesureEnergie, EvenementCompteur
from datetime import datetime, timedelta
import csv
import io
import json
from typing import List, Dict, Optional, Tuple
import pandas as pd
import xlsxwriter
from flask import current_app

class ServiceExport:
    """Service d'export des données au format CSV/Excel"""

    def __init__(self):
        self.delimiter = ';'
        self.quotechar = '"'

    def exporter_mesures_csv(self, debut: datetime, fin: datetime,
                           inclure_stats: bool = True) -> Tuple[str, Dict]:
        """
        Export des mesures au format CSV

        Args:
            debut, fin: Période d'export
            inclure_stats: Inclure des statistiques en tête

        Returns:
            Tuple (contenu_csv, metadata)
        """
        # Récupération des données
        mesures = MesureEnergie.query.filter(
            MesureEnergie.timestamp.between(debut, fin)
        ).order_by(MesureEnergie.timestamp).all()

        if not mesures:
            return "", {'error': 'Aucune donnée disponible'}

        # Création du buffer CSV
        output = io.StringIO()

        # En-têtes
        fieldnames = [
            'timestamp', 'puissance_active_totale', 'energie_active_import',
            'energie_active_export', 'tension_l1', 'tension_l2', 'tension_l3',
            'courant_l1', 'courant_l2', 'courant_l3', 'courant_neutre',
            'frequence', 'facteur_puissance', 'thd_tension', 'thd_courant',
            'source', 'qualite', 'compteur_id'
        ]

        writer = csv.DictWriter(output, fieldnames=fieldnames,
                              delimiter=self.delimiter, quotechar=self.quotechar,
                              quoting=csv.QUOTE_MINIMAL)

        # Écriture des en-têtes
        writer.writeheader()

        # Statistiques si demandées
        if inclure_stats:
            stats = self._calculer_stats_mesures(mesures)
            writer.writerow({})  # Ligne vide
            writer.writerow({'timestamp': 'STATISTIQUES'})
            for key, value in stats.items():
                writer.writerow({'timestamp': key, 'puissance_active_totale': value})

        # Écriture des données
        for mesure in mesures:
            row = {
                'timestamp': mesure.timestamp.isoformat(),
                'puissance_active_totale': mesure.puissance_active_totale,
                'energie_active_import': mesure.energie_active_import,
                'energie_active_export': mesure.energie_active_export,
                'tension_l1': mesure.tension_l1,
                'tension_l2': mesure.tension_l2,
                'tension_l3': mesure.tension_l3,
                'courant_l1': mesure.courant_l1,
                'courant_l2': mesure.courant_l2,
                'courant_l3': mesure.courant_l3,
                'courant_neutre': mesure.courant_neutre,
                'frequence': mesure.frequence,
                'facteur_puissance': mesure.facteur_puissance,
                'thd_tension': mesure.thd_tension,
                'thd_courant': mesure.thd_courant,
                'source': mesure.source,
                'qualite': mesure.qualite,
                'compteur_id': mesure.compteur_id
            }
            writer.writerow(row)

        contenu_csv = output.getvalue()
        output.close()

        # Métadonnées
        metadata = {
            'format': 'csv',
            'nb_lignes': len(mesures),
            'periode': {
                'debut': debut.isoformat(),
                'fin': fin.isoformat()
            },
            'colonnes': len(fieldnames),
            'taille_bytes': len(contenu_csv.encode('utf-8')),
            'stats': stats if inclure_stats else None
        }

        return contenu_csv, metadata

    def exporter_mesures_excel(self, debut: datetime, fin: datetime) -> Tuple[bytes, Dict]:
        """
        Export des mesures au format Excel

        Args:
            debut, fin: Période d'export

        Returns:
            Tuple (contenu_excel_bytes, metadata)
        """
        # Récupération des données
        mesures = MesureEnergie.query.filter(
            MesureEnergie.timestamp.between(debut, fin)
        ).order_by(MesureEnergie.timestamp).all()

        if not mesures:
            return b"", {'error': 'Aucune donnée disponible'}

        # Création du DataFrame pandas
        data = []
        for mesure in mesures:
            data.append({
                'Timestamp': mesure.timestamp,
                'Puissance Active (W)': mesure.puissance_active_totale,
                'Énergie Import (kWh)': mesure.energie_active_import,
                'Énergie Export (kWh)': mesure.energie_active_export,
                'Tension L1 (V)': mesure.tension_l1,
                'Tension L2 (V)': mesure.tension_l2,
                'Tension L3 (V)': mesure.tension_l3,
                'Courant L1 (A)': mesure.courant_l1,
                'Courant L2 (A)': mesure.courant_l2,
                'Courant L3 (A)': mesure.courant_l3,
                'Courant Neutre (A)': mesure.courant_neutre,
                'Fréquence (Hz)': mesure.frequence,
                'Facteur Puissance': mesure.facteur_puissance,
                'THD Tension (%)': mesure.thd_tension,
                'THD Courant (%)': mesure.thd_courant,
                'Source': mesure.source,
                'Qualité': mesure.qualite,
                'ID Compteur': mesure.compteur_id
            })

        df = pd.DataFrame(data)

        # Création du fichier Excel en mémoire
        output = io.BytesIO()
        with pd.ExcelWriter(output, engine='xlsxwriter') as writer:
            # Feuille principale des données
            df.to_excel(writer, sheet_name='Mesures', index=False)

            # Feuille des statistiques
            stats = self._calculer_stats_mesures(mesures)
            stats_df = pd.DataFrame(list(stats.items()), columns=['Statistique', 'Valeur'])
            stats_df.to_excel(writer, sheet_name='Statistiques', index=False)

            # Configuration du formatage
            workbook = writer.book
            worksheet = writer.sheets['Mesures']

            # Format des en-têtes
            header_format = workbook.add_format({
                'bold': True,
                'bg_color': '#4F81BD',
                'color': 'white',
                'border': 1
            })

            # Application du format aux en-têtes
            for col_num, value in enumerate(df.columns.values):
                worksheet.write(0, col_num, value, header_format)

            # Ajustement automatique des colonnes
            for i, col in enumerate(df.columns):
                max_len = max(df[col].astype(str).apply(len).max(), len(col)) + 2
                worksheet.set_column(i, i, min(max_len, 50))  # Largeur max 50

            # Format des dates
            date_format = workbook.add_format({'num_format': 'yyyy-mm-dd hh:mm:ss'})
            worksheet.set_column(0, 0, 18, date_format)  # Colonne timestamp

        contenu_excel = output.getvalue()
        output.close()

        # Métadonnées
        metadata = {
            'format': 'excel',
            'nb_lignes': len(mesures),
            'nb_colonnes': len(df.columns),
            'periode': {
                'debut': debut.isoformat(),
                'fin': fin.isoformat()
            },
            'taille_bytes': len(contenu_excel),
            'feuilles': ['Mesures', 'Statistiques']
        }

        return contenu_excel, metadata

    def exporter_evenements_csv(self, debut: datetime, fin: datetime) -> Tuple[str, Dict]:
        """
        Export des événements au format CSV

        Args:
            debut, fin: Période d'export

        Returns:
            Tuple (contenu_csv, metadata)
        """
        evenements = EvenementCompteur.query.filter(
            EvenementCompteur.timestamp_detection.between(debut, fin)
        ).order_by(EvenementCompteur.timestamp_detection).all()

        output = io.StringIO()

        fieldnames = [
            'timestamp_detection', 'type_anomalie', 'categorie', 'severite',
            'titre', 'description', 'valeur_mesuree', 'unite_mesure',
            'seuil_declencheur', 'code_obis', 'statut', 'alerte_envoyee'
        ]

        writer = csv.DictWriter(output, fieldnames=fieldnames,
                              delimiter=self.delimiter, quotechar=self.quotechar,
                              quoting=csv.QUOTE_MINIMAL)

        writer.writeheader()

        for evt in evenements:
            row = {
                'timestamp_detection': evt.timestamp_detection.isoformat(),
                'type_anomalie': evt.type_anomalie,
                'categorie': evt.categorie,
                'severite': evt.severite,
                'titre': evt.titre,
                'description': evt.description,
                'valeur_mesuree': evt.valeur_mesuree,
                'unite_mesure': evt.unite_mesure,
                'seuil_declencheur': evt.seuil_declencheur,
                'code_obis': evt.code_obis,
                'statut': evt.statut,
                'alerte_envoyee': evt.alerte_envoyee
            }
            writer.writerow(row)

        contenu_csv = output.getvalue()
        output.close()

        metadata = {
            'format': 'csv',
            'type': 'evenements',
            'nb_lignes': len(evenements),
            'periode': {
                'debut': debut.isoformat(),
                'fin': fin.isoformat()
            }
        }

        return contenu_csv, metadata

    def _calculer_stats_mesures(self, mesures: List[MesureEnergie]) -> Dict:
        """Calcule les statistiques des mesures"""
        if not mesures:
            return {}

        puissances = [m.puissance_active_totale for m in mesures if m.puissance_active_totale is not None]
        tensions = [m.tension_l1 for m in mesures if m.tension_l1 is not None]

        stats = {
            'Nombre de mesures': len(mesures),
            'Période': f"{mesures[0].timestamp.date()} à {mesures[-1].timestamp.date()}",
            'Durée': f"{(mesures[-1].timestamp - mesures[0].timestamp).total_seconds() / 3600:.1f} heures"
        }

        if puissances:
            stats.update({
                'Puissance moyenne (W)': f"{sum(puissances) / len(puissances):.1f}",
                'Puissance max (W)': f"{max(puissances):.1f}",
                'Puissance min (W)': f"{min(puissances):.1f}"
            })

        if tensions:
            stats.update({
                'Tension moyenne (V)': f"{sum(tensions) / len(tensions):.1f}",
                'Tension max (V)': f"{max(tensions):.1f}",
                'Tension min (V)': f"{min(tensions):.1f}"
            })

        return stats

    def exporter_rapport_complet(self, debut: datetime, fin: datetime) -> Tuple[bytes, Dict]:
        """
        Export d'un rapport complet avec mesures, événements et analyses

        Args:
            debut, fin: Période du rapport

        Returns:
            Tuple (contenu_excel, metadata)
        """
        output = io.BytesIO()

        with pd.ExcelWriter(output, engine='xlsxwriter') as writer:
            workbook = writer.book

            # Feuille 1: Mesures
            mesures = MesureEnergie.query.filter(
                MesureEnergie.timestamp.between(debut, fin)
            ).order_by(MesureEnergie.timestamp).all()

            if mesures:
                mesures_data = [{
                    'Timestamp': m.timestamp,
                    'Puissance (W)': m.puissance_active_totale,
                    'Énergie Import (kWh)': m.energie_active_import,
                    'Tension L1 (V)': m.tension_l1,
                    'Courant L1 (A)': m.courant_l1,
                    'Fréquence (Hz)': m.frequence,
                    'Source': m.source,
                    'Qualité': m.qualite
                } for m in mesures]

                mesures_df = pd.DataFrame(mesures_data)
                mesures_df.to_excel(writer, sheet_name='Mesures', index=False)

            # Feuille 2: Événements
            evenements = EvenementCompteur.query.filter(
                EvenementCompteur.timestamp_detection.between(debut, fin)
            ).order_by(EvenementCompteur.timestamp_detection).all()

            if evenements:
                evenements_data = [{
                    'Timestamp': evt.timestamp_detection,
                    'Type': evt.type_anomalie,
                    'Sévérité': evt.severite,
                    'Titre': evt.titre,
                    'Description': evt.description,
                    'Valeur': evt.valeur_mesuree,
                    'Unité': evt.unite_mesure,
                    'Statut': evt.statut
                } for evt in evenements]

                evenements_df = pd.DataFrame(evenements_data)
                evenements_df.to_excel(writer, sheet_name='Événements', index=False)

            # Feuille 3: Analyses
            analyses_data = self._generer_analyses_rapport(mesures, debut, fin)
            analyses_df = pd.DataFrame(list(analyses_data.items()), columns=['Analyse', 'Valeur'])
            analyses_df.to_excel(writer, sheet_name='Analyses', index=False)

            # Formatage
            self._formater_rapport_excel(writer, workbook)

        contenu_rapport = output.getvalue()
        output.close()

        metadata = {
            'format': 'excel',
            'type': 'rapport_complet',
            'feuilles': ['Mesures', 'Événements', 'Analyses'],
            'periode': {
                'debut': debut.isoformat(),
                'fin': fin.isoformat()
            },
            'taille_bytes': len(contenu_rapport)
        }

        return contenu_rapport, metadata

    def _generer_analyses_rapport(self, mesures: List[MesureEnergie],
                                debut: datetime, fin: datetime) -> Dict:
        """Génère les analyses pour le rapport"""
        if not mesures:
            return {'Erreur': 'Aucune donnée disponible'}

        analyses = {
            'Période d\'analyse': f"{debut.date()} à {fin.date()}",
            'Nombre de mesures': len(mesures),
            'Durée totale': f"{(fin - debut).total_seconds() / 3600:.1f} heures"
        }

        # Statistiques de puissance
        puissances = [m.puissance_active_totale for m in mesures if m.puissance_active_totale]
        if puissances:
            analyses.update({
                'Puissance moyenne': f"{sum(puissances) / len(puissances):.1f} W",
                'Puissance maximale': f"{max(puissances):.1f} W",
                'Puissance minimale': f"{min(puissances):.1f} W"
            })

        # Statistiques d'énergie
        if len(mesures) > 1:
            energie_debut = mesures[0].energie_active_import or 0
            energie_fin = mesures[-1].energie_active_import or 0
            energie_consommee = energie_fin - energie_debut

            analyses['Énergie consommée'] = f"{energie_consommee:.3f} kWh"

        # Nombre d'événements
        nb_evenements = EvenementCompteur.query.filter(
            EvenementCompteur.timestamp_detection.between(debut, fin)
        ).count()

        analyses['Nombre d\'événements'] = nb_evenements

        return analyses

    def _formater_rapport_excel(self, writer, workbook):
        """Formatage du rapport Excel"""
        # Format des en-têtes
        header_format = workbook.add_format({
            'bold': True,
            'bg_color': '#4F81BD',
            'color': 'white',
            'border': 1
        })

        # Application aux feuilles
        for sheet_name in writer.sheets:
            worksheet = writer.sheets[sheet_name]
            # Formatage des en-têtes (première ligne)
            worksheet.set_row(0, None, header_format)
```

### API d'export

```python
# app/routes/api_export.py
from flask import Blueprint, request, jsonify, send_file
from app.services.export_service import ServiceExport
from datetime import datetime, timedelta
import io

export_bp = Blueprint('export', __name__)
export_service = ServiceExport()

@export_bp.route('/api/export/csv')
def export_csv():
    """Export CSV des mesures"""

    # Paramètres
    heures = int(request.args.get('hours', 24))
    inclure_stats = request.args.get('stats', 'true').lower() == 'true'

    debut = datetime.utcnow() - timedelta(hours=heures)
    fin = datetime.utcnow()

    # Génération
    csv_content, metadata = export_service.exporter_mesures_csv(debut, fin, inclure_stats)

    if not csv_content:
        return jsonify({'error': metadata.get('error', 'Erreur export')}), 400

    # Retour du fichier
    filename = f"mesures_e450_{debut.date()}_{heures}h.csv"

    return send_file(
        io.BytesIO(csv_content.encode('utf-8-sig')),  # BOM pour Excel
        mimetype='text/csv',
        as_attachment=True,
        download_name=filename
    )

@export_bp.route('/api/export/excel')
def export_excel():
    """Export Excel des mesures"""

    heures = int(request.args.get('hours', 24))

    debut = datetime.utcnow() - timedelta(hours=heures)
    fin = datetime.utcnow()

    excel_content, metadata = export_service.exporter_mesures_excel(debut, fin)

    if not excel_content:
        return jsonify({'error': metadata.get('error', 'Erreur export')}), 400

    filename = f"mesures_e450_{debut.date()}_{heures}h.xlsx"

    return send_file(
        io.BytesIO(excel_content),
        mimetype='application/vnd.openxmlformats-officedocument.spreadsheetml.sheet',
        as_attachment=True,
        download_name=filename
    )

@export_bp.route('/api/export/events/csv')
def export_events_csv():
    """Export CSV des événements"""

    jours = int(request.args.get('days', 7))

    debut = datetime.utcnow() - timedelta(days=jours)
    fin = datetime.utcnow()

    csv_content, metadata = export_service.exporter_evenements_csv(debut, fin)

    if not csv_content:
        return jsonify({'error': metadata.get('error', 'Erreur export')}), 400

    filename = f"evenements_e450_{debut.date()}_{jours}j.csv"

    return send_file(
        io.BytesIO(csv_content.encode('utf-8-sig')),
        mimetype='text/csv',
        as_attachment=True,
        download_name=filename
    )

@export_bp.route('/api/export/report')
def export_full_report():
    """Export du rapport complet Excel"""

    heures = int(request.args.get('hours', 168))  # 7 jours par défaut

    debut = datetime.utcnow() - timedelta(hours=heures)
    fin = datetime.utcnow()

    report_content, metadata = export_service.exporter_rapport_complet(debut, fin)

    if not report_content:
        return jsonify({'error': metadata.get('error', 'Erreur génération rapport')}), 400

    filename = f"rapport_e450_{debut.date()}_{heures}h.xlsx"

    return send_file(
        io.BytesIO(report_content),
        mimetype='application/vnd.openxmlformats-officedocument.spreadsheetml.sheet',
        as_attachment=True,
        download_name=filename
    )

@export_bp.route('/api/export/info')
def export_info():
    """Informations sur les exports disponibles"""

    return jsonify({
        'formats_disponibles': ['csv', 'excel'],
        'types_export': [
            {
                'endpoint': '/api/export/csv',
                'description': 'Mesures au format CSV',
                'params': ['hours', 'stats']
            },
            {
                'endpoint': '/api/export/excel',
                'description': 'Mesures au format Excel',
                'params': ['hours']
            },
            {
                'endpoint': '/api/export/events/csv',
                'description': 'Événements au format CSV',
                'params': ['days']
            },
            {
                'endpoint': '/api/export/report',
                'description': 'Rapport complet Excel (mesures + événements + analyses)',
                'params': ['hours']
            }
        ],
        'limites': {
            'max_heures': 8760,  # 1 an
            'max_jours': 365
        }
    })
```

### Interface utilisateur pour l'export

```html
<!-- templates/export.html -->
{% extends "base.html" %}

{% block title %}Export des Données - Compteur E450{% endblock %}

{% block extra_head %}
<link rel="stylesheet" href="{{ url_for('static', filename='css/export.css') }}">
{% endblock %}

{% block content %}
<div class="export-container">
    <div class="export-header">
        <h1>Export des Données</h1>
        <p>Exportez vos données de consommation au format CSV ou Excel pour analyse externe</p>
    </div>

    <div class="export-options">
        <!-- Export des mesures -->
        <div class="export-card">
            <div class="card-header">
                <h3>📊 Mesures de Consommation</h3>
                <p>Export des données brutes de mesure</p>
            </div>

            <div class="card-body">
                <div class="form-group">
                    <label>Période:</label>
                    <select id="mesures-period">
                        <option value="24">Dernières 24 heures</option>
                        <option value="168">Dernière semaine</option>
                        <option value="720">Dernier mois</option>
                        <option value="8760">Dernière année</option>
                    </select>
                </div>

                <div class="form-group">
                    <label>Inclure statistiques:</label>
                    <input type="checkbox" id="mesures-stats" checked>
                </div>

                <div class="export-buttons">
                    <button class="btn btn-primary" onclick="exportMesures('csv')">
                        📄 Exporter CSV
                    </button>
                    <button class="btn btn-success" onclick="exportMesures('excel')">
                        📊 Exporter Excel
                    </button>
                </div>
            </div>
        </div>

        <!-- Export des événements -->
        <div class="export-card">
            <div class="card-header">
                <h3>🚨 Événements et Anomalies</h3>
                <p>Export de l'historique des événements détectés</p>
            </div>

            <div class="card-body">
                <div class="form-group">
                    <label>Période:</label>
                    <select id="events-period">
                        <option value="7">7 derniers jours</option>
                        <option value="30">30 derniers jours</option>
                        <option value="90">90 derniers jours</option>
                    </select>
                </div>

                <div class="export-buttons">
                    <button class="btn btn-warning" onclick="exportEvenements()">
                        📄 Exporter CSV
                    </button>
                </div>
            </div>
        </div>

        <!-- Rapport complet -->
        <div class="export-card featured">
            <div class="card-header">
                <h3>📋 Rapport Complet</h3>
                <p>Rapport Excel complet avec mesures, événements et analyses</p>
            </div>

            <div class="card-body">
                <div class="form-group">
                    <label>Période:</label>
                    <select id="report-period">
                        <option value="168">Dernière semaine</option>
                        <option value="720">Dernier mois</option>
                        <option value="2160">3 derniers mois</option>
                    </select>
                </div>

                <div class="export-buttons">
                    <button class="btn btn-premium" onclick="exportRapport()">
                        📈 Générer Rapport
                    </button>
                </div>
            </div>
        </div>
    </div>

    <!-- Historique des exports -->
    <div class="export-history">
        <h3>Historique des Exports</h3>
        <div id="export-history-list">
            <!-- Chargé dynamiquement -->
        </div>
    </div>
</div>
{% endblock %}

{% block extra_scripts %}
<script src="{{ url_for('static', filename='js/export.js') }}"></script>
{% endblock %}
```

> **💡 À retenir** : L'export de données permet d'exploiter vos mesures dans des outils d'analyse avancés tout en gardant l'historique complet dans votre application.

> **⚠️ Astuce** : Utilisez les rapports Excel complets pour partager vos analyses avec des collègues ou pour archiver vos données de manière structurée.

Félicitations ! La Partie V sur l'analyse et la visualisation avancée est maintenant complète. Vous maîtrisez maintenant l'analyse statistique, les visualisations graphiques et l'export de données. La Partie VI va explorer les cas d'usage pratiques !

---

**Navigation**
- [Chapitre précédent : Courbes de consommation](Chapitre_21_Courbes_Consommation.md)
- [Chapitre suivant : Prévision énergétique avec Prophet](../Partie_V_Analyse_Visualisation/Chapitre_23_Preuve_Energetique_Prophet.md)
- [Retour à la table des matières](../../README.md)
