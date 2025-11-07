# 📊 Chapitre 21 : Courbes de consommation

## 📈 Visualisation des données temporelles

Les courbes de consommation permettent de visualiser l'évolution de votre usage énergétique dans le temps.

### Service de génération de graphiques

```python
# app/services/visualisation.py
from app.models import MesureEnergie
from datetime import datetime, timedelta
import matplotlib.pyplot as plt
import matplotlib.dates as mdates
import io
import base64
from typing import List, Dict, Optional, Tuple
import seaborn as sns
import pandas as pd
import numpy as np

class ServiceVisualisation:
    """Service de génération de graphiques pour les données énergétiques"""

    def __init__(self):
        # Configuration matplotlib
        plt.style.use('seaborn-v0_8')
        sns.set_palette("husl")

        # Configuration pour les graphiques en mémoire
        plt.ioff()  # Mode non-interactif

    def generer_courbe_consommation(self, debut: datetime, fin: datetime,
                                   format_sortie: str = 'base64') -> Dict:
        """
        Génère une courbe de consommation sur une période

        Args:
            debut, fin: Période de visualisation
            format_sortie: 'base64', 'bytes', 'file'

        Returns:
            Dictionnaire avec l'image et les métadonnées
        """
        # Récupération des données
        mesures = MesureEnergie.query.filter(
            MesureEnergie.timestamp.between(debut, fin)
        ).order_by(MesureEnergie.timestamp).all()

        if not mesures:
            return {'erreur': 'Aucune donnée disponible'}

        # Conversion en DataFrame
        df = pd.DataFrame([{
            'timestamp': m.timestamp,
            'puissance': m.puissance_active_totale or 0,
            'tension_l1': m.tension_l1 or 230,
            'courant_l1': m.courant_l1 or 0
        } for m in mesures])

        # Création du graphique
        fig, (ax1, ax2) = plt.subplots(2, 1, figsize=(12, 8), sharex=True)

        # Graphique de puissance
        ax1.plot(df['timestamp'], df['puissance'], 'b-', linewidth=1.5, alpha=0.8)
        ax1.fill_between(df['timestamp'], df['puissance'], alpha=0.3)
        ax1.set_ylabel('Puissance (W)', color='b')
        ax1.tick_params(axis='y', labelcolor='b')
        ax1.grid(True, alpha=0.3)

        # Titre et formatage
        ax1.set_title(f'Consommation Électrique - {debut.date()} à {fin.date()}', fontsize=14, fontweight='bold')

        # Graphique de tension
        ax2.plot(df['timestamp'], df['tension_l1'], 'r-', linewidth=1, alpha=0.7)
        ax2.set_ylabel('Tension (V)', color='r')
        ax2.tick_params(axis='y', labelcolor='r')
        ax2.grid(True, alpha=0.3)

        # Ligne de référence 230V
        ax2.axhline(y=230, color='r', linestyle='--', alpha=0.5, label='230V')
        ax2.legend()

        # Formatage des dates
        ax2.xaxis.set_major_formatter(mdates.DateFormatter('%d/%m %H:%M'))
        plt.setp(ax2.xaxis.get_majorticklabels(), rotation=45, ha='right')

        # Ajustement des marges
        plt.tight_layout()

        # Génération de l'image
        image_data = self._generer_image(fig, format_sortie)

        # Statistiques
        stats = {
            'puissance_moyenne': df['puissance'].mean(),
            'puissance_max': df['puissance'].max(),
            'puissance_min': df['puissance'].min(),
            'nb_points': len(df),
            'periode_heures': (fin - debut).total_seconds() / 3600
        }

        plt.close(fig)

        return {
            'image': image_data,
            'format': format_sortie,
            'stats': stats,
            'periode': {
                'debut': debut.isoformat(),
                'fin': fin.isoformat()
            }
        }

    def generer_histogramme_journalier(self, jours: int = 7) -> Dict:
        """
        Génère un histogramme de la consommation par jour

        Args:
            jours: Nombre de jours à analyser

        Returns:
            Graphique et statistiques
        """
        debut = datetime.utcnow() - timedelta(days=jours)
        fin = datetime.utcnow()

        # Agrégation par jour
        mesures = MesureEnergie.query.filter(
            MesureEnergie.timestamp.between(debut, fin)
        ).all()

        if not mesures:
            return {'erreur': 'Aucune donnée disponible'}

        # Grouper par jour
        conso_journaliere = {}
        for mesure in mesures:
            jour = mesure.timestamp.date()
            if jour not in conso_journaliere:
                conso_journaliere[jour] = []

            # Estimation de l'énergie consommée (approximation)
            conso_journaliere[jour].append(mesure.puissance_active_totale or 0)

        # Calcul des moyennes journalières
        jours_labels = []
        conso_moyenne = []

        for jour in sorted(conso_journaliere.keys()):
            valeurs = conso_journaliere[jour]
            if valeurs:
                jours_labels.append(jour.strftime('%d/%m'))
                conso_moyenne.append(sum(valeurs) / len(valeurs))

        # Création de l'histogramme
        fig, ax = plt.subplots(figsize=(10, 6))

        bars = ax.bar(jours_labels, conso_moyenne, color='skyblue', alpha=0.7, edgecolor='navy', linewidth=1)

        # Ajout des valeurs sur les barres
        for bar, valeur in zip(bars, conso_moyenne):
            height = bar.get_height()
            ax.text(bar.get_x() + bar.get_width()/2., height + 5,
                   '.0f', ha='center', va='bottom', fontweight='bold')

        ax.set_title(f'Consommation Moyenne Journalière - {jours} derniers jours',
                    fontsize=14, fontweight='bold')
        ax.set_ylabel('Puissance Moyenne (W)')
        ax.grid(True, alpha=0.3, axis='y')

        # Rotation des labels si nécessaire
        plt.setp(ax.xaxis.get_majorticklabels(), rotation=45, ha='right')
        plt.tight_layout()

        # Génération de l'image
        image_data = self._generer_image(fig, 'base64')
        plt.close(fig)

        return {
            'image': image_data,
            'stats': {
                'jours_analyses': len(jours_labels),
                'conso_max': max(conso_moyenne) if conso_moyenne else 0,
                'conso_min': min(conso_moyenne) if conso_moyenne else 0,
                'conso_moyenne_generale': sum(conso_moyenne) / len(conso_moyenne) if conso_moyenne else 0
            }
        }

    def generer_profil_horaire(self, jours: int = 7) -> Dict:
        """
        Génère un profil de consommation moyen par heure

        Args:
            jours: Nombre de jours pour le calcul de la moyenne

        Returns:
            Profil horaire visualisé
        """
        debut = datetime.utcnow() - timedelta(days=jours)

        mesures = MesureEnergie.query.filter(
            MesureEnergie.timestamp >= debut
        ).all()

        if not mesures:
            return {'erreur': 'Aucune donnée disponible'}

        # Agrégation par heure
        conso_par_heure = {}
        for mesure in mesures:
            heure = mesure.timestamp.hour
            if heure not in conso_par_heure:
                conso_par_heure[heure] = []

            conso_par_heure[heure].append(mesure.puissance_active_totale or 0)

        # Calcul des moyennes
        heures = sorted(conso_par_heure.keys())
        moyennes = [sum(conso_par_heure[h]) / len(conso_par_heure[h]) for h in heures]

        # Création du graphique radar
        fig, ax = plt.subplots(figsize=(8, 8), subplot_kw=dict(projection='polar'))

        # Conversion en radians pour le radar
        angles = np.linspace(0, 2 * np.pi, len(heures), endpoint=False).tolist()
        moyennes_circular = moyennes + [moyennes[0]]  # Fermer le cercle
        angles_circular = angles + [angles[0]]

        ax.plot(angles_circular, moyennes_circular, 'o-', linewidth=2, label='Consommation moyenne', color='darkblue')
        ax.fill(angles_circular, moyennes_circular, alpha=0.25, color='lightblue')

        # Configuration des axes
        ax.set_xticks(angles)
        ax.set_xticklabels([f'{h:02d}h' for h in heures])
        ax.set_ylim(0, max(moyennes) * 1.1)

        ax.set_title(f'Profil Horaire Moyen - {jours} derniers jours',
                    size=14, fontweight='bold', pad=20)
        ax.grid(True, alpha=0.3)

        # Légende avec statistiques
        conso_max = max(moyennes)
        heure_max = heures[moyennes.index(conso_max)]
        ax.legend([f'Pic à {heure_max:02d}h: {conso_max:.0f}W'], loc='upper right')

        plt.tight_layout()

        # Génération de l'image
        image_data = self._generer_image(fig, 'base64')
        plt.close(fig)

        return {
            'image': image_data,
            'stats': {
                'heure_pic': heure_max,
                'conso_pic': conso_max,
                'conso_moyenne': sum(moyennes) / len(moyennes),
                'jours_analyse': jours
            }
        }

    def generer_heatmap_consommation(self, jours: int = 30) -> Dict:
        """
        Génère une heatmap de la consommation par jour et par heure

        Args:
            jours: Nombre de jours à analyser

        Returns:
            Heatmap journalière
        """
        debut = datetime.utcnow() - timedelta(days=jours)

        mesures = MesureEnergie.query.filter(
            MesureEnergie.timestamp >= debut
        ).all()

        if not mesures:
            return {'erreur': 'Aucune donnée disponible'}

        # Création d'une matrice jour x heure
        consommation_matrix = np.zeros((24, jours))  # 24h x N jours

        for mesure in mesures:
            jour_idx = (mesure.timestamp.date() - debut.date()).days
            heure_idx = mesure.timestamp.hour

            if 0 <= jour_idx < jours:
                consommation_matrix[heure_idx, jour_idx] = mesure.puissance_active_totale or 0

        # Création de la heatmap
        fig, ax = plt.subplots(figsize=(12, 8))

        # Heatmap avec seaborn pour de meilleurs résultats
        sns.heatmap(consommation_matrix,
                   cmap='YlOrRd',
                   ax=ax,
                   cbar_kws={'label': 'Puissance (W)'})

        # Configuration des axes
        ax.set_title(f'Heatmap de Consommation - {jours} derniers jours',
                    fontsize=14, fontweight='bold', pad=20)

        ax.set_xlabel('Jour')
        ax.set_ylabel('Heure')

        # Labels des jours (tous les 5 jours)
        jour_labels = [(debut + timedelta(days=i)).strftime('%d/%m') for i in range(0, jours, 5)]
        ax.set_xticks(range(0, jours, 5))
        ax.set_xticklabels(jour_labels, rotation=45, ha='right')

        # Labels des heures
        heure_labels = [f'{h:02d}h' for h in range(24)]
        ax.set_yticks(range(24))
        ax.set_yticklabels(heure_labels)

        plt.tight_layout()

        # Génération de l'image
        image_data = self._generer_image(fig, 'base64')
        plt.close(fig)

        return {
            'image': image_data,
            'stats': {
                'jours_analyse': jours,
                'conso_max_matrix': float(np.max(consommation_matrix)),
                'conso_moyenne_matrix': float(np.mean(consommation_matrix)),
                'heure_plus_consommee': int(np.argmax(np.mean(consommation_matrix, axis=1)))
            }
        }

    def _generer_image(self, fig, format_sortie: str):
        """Génère l'image dans le format demandé"""
        buf = io.BytesIO()

        if format_sortie == 'base64':
            fig.savefig(buf, format='png', dpi=100, bbox_inches='tight')
            buf.seek(0)
            image_base64 = base64.b64encode(buf.getvalue()).decode('utf-8')
            return f"data:image/png;base64,{image_base64}"

        elif format_sortie == 'bytes':
            fig.savefig(buf, format='png', dpi=100, bbox_inches='tight')
            buf.seek(0)
            return buf.getvalue()

        else:
            raise ValueError(f"Format de sortie non supporté: {format_sortie}")

    def generer_rapport_visuel(self, debut: datetime, fin: datetime) -> Dict:
        """
        Génère un rapport visuel complet avec plusieurs graphiques

        Args:
            debut, fin: Période du rapport

        Returns:
            Rapport avec multiple visualisations
        """
        rapport = {
            'periode': {
                'debut': debut.isoformat(),
                'fin': fin.isoformat()
            },
            'graphiques': {},
            'statistiques': {}
        }

        # Graphique de tendance
        tendance = self.generer_courbe_consommation(debut, fin)
        if 'image' in tendance:
            rapport['graphiques']['tendance'] = tendance['image']
            rapport['statistiques']['tendance'] = tendance['stats']

        # Profil horaire (sur la période)
        profil = self.generer_profil_horaire((fin - debut).days or 7)
        if 'image' in profil:
            rapport['graphiques']['profil_horaire'] = profil['image']
            rapport['statistiques']['profil'] = profil['stats']

        return rapport
```

### API pour les visualisations

```python
# app/routes/api_charts.py
from flask import Blueprint, request, jsonify, send_file
from app.services.visualisation import ServiceVisualisation
from datetime import datetime, timedelta
import io

charts_bp = Blueprint('charts', __name__)
visualiseur = ServiceVisualisation()

@charts_bp.route('/api/charts/consumption')
def get_consumption_chart():
    """Graphique de consommation sur une période"""

    # Paramètres
    heures = int(request.args.get('hours', 24))
    format_img = request.args.get('format', 'base64')

    debut = datetime.utcnow() - timedelta(hours=heures)
    fin = datetime.utcnow()

    # Génération
    chart = visualiseur.generer_courbe_consommation(debut, fin, format_img)

    if 'erreur' in chart:
        return jsonify({'error': chart['erreur']}), 400

    return jsonify({
        'chart': chart['image'],
        'stats': chart['stats'],
        'periode': chart['periode']
    })

@charts_bp.route('/api/charts/daily-bars')
def get_daily_bars():
    """Histogramme journalier"""

    jours = int(request.args.get('days', 7))
    format_img = request.args.get('format', 'base64')

    # Modification temporaire du format dans la méthode
    chart = visualiseur.generer_histogramme_journalier(jours)

    if 'erreur' in chart:
        return jsonify({'error': chart['erreur']}), 400

    # Adaptation du format si nécessaire
    if format_img != 'base64' and 'image' in chart:
        # Pour les autres formats, on pourrait implémenter une conversion
        pass

    return jsonify({
        'chart': chart['image'],
        'stats': chart['stats']
    })

@charts_bp.route('/api/charts/hourly-profile')
def get_hourly_profile():
    """Profil horaire"""

    jours = int(request.args.get('days', 7))

    profile = visualiseur.generer_profil_horaire(jours)

    if 'erreur' in profile:
        return jsonify({'error': profile['erreur']}), 400

    return jsonify({
        'chart': profile['image'],
        'stats': profile['stats']
    })

@charts_bp.route('/api/charts/heatmap')
def get_heatmap():
    """Heatmap de consommation"""

    jours = int(request.args.get('days', 30))

    heatmap = visualiseur.generer_heatmap_consommation(jours)

    if 'erreur' in heatmap:
        return jsonify({'error': heatmap['erreur']}), 400

    return jsonify({
        'chart': heatmap['image'],
        'stats': heatmap['stats']
    })

@charts_bp.route('/api/charts/report')
def get_visual_report():
    """Rapport visuel complet"""

    heures = int(request.args.get('hours', 168))  # 7 jours par défaut

    debut = datetime.utcnow() - timedelta(hours=heures)
    fin = datetime.utcnow()

    rapport = visualiseur.generer_rapport_visuel(debut, fin)

    return jsonify(rapport)

@charts_bp.route('/api/charts/export/<chart_type>')
def export_chart(chart_type):
    """Export d'un graphique au format PNG"""

    # Paramètres communs
    format_img = 'bytes'  # Pour l'export fichier

    if chart_type == 'consumption':
        heures = int(request.args.get('hours', 24))
        debut = datetime.utcnow() - timedelta(hours=heures)
        fin = datetime.utcnow()
        chart = visualiseur.generer_courbe_consommation(debut, fin, format_img)

    elif chart_type == 'daily':
        jours = int(request.args.get('days', 7))
        chart = visualiseur.generer_histogramme_journalier(jours)
        # Conversion du format base64 vers bytes (simplifié)
        if 'image' in chart:
            import base64
            image_data = chart['image'].split(',')[1]  # Supprimer le préfixe data:
            chart_bytes = base64.b64decode(image_data)
        else:
            return jsonify({'error': 'Erreur génération graphique'}), 500

    else:
        return jsonify({'error': 'Type de graphique non supporté'}), 400

    if 'erreur' in chart:
        return jsonify({'error': chart['erreur']}), 400

    # Retour du fichier PNG
    filename = f"chart_{chart_type}_{datetime.utcnow().strftime('%Y%m%d_%H%M%S')}.png"

    return send_file(
        io.BytesIO(chart_bytes),
        mimetype='image/png',
        as_attachment=True,
        download_name=filename
    )
```

> **💡 À retenir** : Les visualisations transforment vos données numériques en représentations graphiques intuitives, facilitant l'identification des patterns de consommation.

> **⚠️ Astuce** : Utilisez différents types de graphiques (courbes, histogrammes, heatmaps) pour révéler différents aspects de vos données énergétiques.

Dans le prochain chapitre, nous explorerons l'**export CSV/Excel** pour l'analyse externe de vos données !

---

**Navigation**
- [Chapitre précédent : Tendances et statistiques](Chapitre_20_Tendances_Statistiques.md)
- [Chapitre suivant : Export CSV/Excel](Chapitre_22_Export_CSV_Excel.md)
- [Retour à la table des matières](../../README.md)
