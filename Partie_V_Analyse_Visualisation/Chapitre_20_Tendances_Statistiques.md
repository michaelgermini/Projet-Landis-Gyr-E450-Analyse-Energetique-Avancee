# 📊 Chapitre 20 : Tendances et statistiques

## 📈 Analyse temporelle des données énergétiques

L'analyse des tendances permet de comprendre l'évolution de la consommation et d'identifier des patterns comportementaux.

### Service d'analyse statistique

```python
# app/services/analyse_statistique.py
from app.models import MesureEnergie
from datetime import datetime, timedelta, time
from typing import List, Dict, Tuple, Optional
import statistics
import numpy as np
from scipy import stats
import pandas as pd

class AnalyseurStatistique:
    """Analyseur statistique des données énergétiques"""

    def __init__(self):
        self.periodes_analyse = {
            'horaire': lambda dt: dt.replace(minute=0, second=0, microsecond=0),
            'journaliere': lambda dt: dt.date(),
            'hebdomadaire': lambda dt: dt.date() - timedelta(days=dt.weekday()),
            'mensuelle': lambda dt: dt.replace(day=1, hour=0, minute=0, second=0, microsecond=0)
        }

    def analyser_periode(self, debut: datetime, fin: datetime,
                        periode_groupement: str = 'horaire') -> Dict:
        """
        Analyse statistique d'une période donnée

        Args:
            debut, fin: Période d'analyse
            periode_groupement: 'horaire', 'journaliere', 'hebdomadaire', 'mensuelle'

        Returns:
            Analyse statistique complète
        """
        # Récupération des données
        mesures = MesureEnergie.query.filter(
            MesureEnergie.timestamp.between(debut, fin)
        ).order_by(MesureEnergie.timestamp).all()

        if not mesures:
            return {'erreur': 'Aucune donnée disponible'}

        # Conversion en DataFrame pour analyse
        df = pd.DataFrame([{
            'timestamp': m.timestamp,
            'puissance': m.puissance_active_totale or 0,
            'energie_import': m.energie_active_import or 0,
            'tension_l1': m.tension_l1 or 0,
            'courant_l1': m.courant_l1 or 0,
            'frequence': m.frequence or 50
        } for m in mesures])

        # Analyse par période
        analyse = self._analyser_par_periode(df, periode_groupement)

        # Statistiques globales
        stats_globales = self._calculer_stats_globales(df)

        # Détection de saisonnalité
        saisonnalite = self._detecter_saisonnalite(df)

        # Tendances
        tendances = self._analyser_tendances(df)

        return {
            'periode': {
                'debut': debut.isoformat(),
                'fin': fin.isoformat(),
                'duree_heures': (fin - debut).total_seconds() / 3600,
                'nb_mesures': len(mesures)
            },
            'analyse_par_periode': analyse,
            'statistiques_globales': stats_globales,
            'saisonnalite': saisonnalite,
            'tendances': tendances
        }

    def _analyser_par_periode(self, df: pd.DataFrame, periode: str) -> Dict:
        """Analyse groupée par période"""
        if periode not in self.periodes_analyse:
            raise ValueError(f"Période invalide: {periode}")

        # Grouper par période
        df_copy = df.copy()
        df_copy['periode'] = df_copy['timestamp'].apply(self.periodes_analyse[periode])

        grouped = df_copy.groupby('periode')

        resultats = []
        for nom_periode, groupe in grouped:
            stats_periode = {
                'periode': nom_periode.isoformat() if hasattr(nom_periode, 'isoformat') else str(nom_periode),
                'nb_mesures': len(groupe),
                'puissance_moyenne': groupe['puissance'].mean(),
                'puissance_max': groupe['puissance'].max(),
                'puissance_min': groupe['puissance'].min(),
                'puissance_ecart_type': groupe['puissance'].std(),
                'energie_consommee': groupe['energie_import'].max() - groupe['energie_import'].min() if len(groupe) > 1 else 0,
                'tension_moyenne': groupe['tension_l1'].mean(),
                'courant_moyen': groupe['courant_l1'].mean(),
            }
            resultats.append(stats_periode)

        return {
            'periode_type': periode,
            'nb_periodes': len(resultats),
            'periodes': resultats
        }

    def _calculer_stats_globales(self, df: pd.DataFrame) -> Dict:
        """Statistiques globales sur l'ensemble des données"""
        puissance = df['puissance']

        # Statistiques descriptives
        stats = {
            'puissance_moyenne': puissance.mean(),
            'puissance_mediane': puissance.median(),
            'puissance_max': puissance.max(),
            'puissance_min': puissance.min(),
            'puissance_ecart_type': puissance.std(),
            'puissance_skewness': puissance.skew(),  # Asymétrie
            'puissance_kurtosis': puissance.kurtosis(),  # Aplatissement
        }

        # Quartiles
        quartiles = puissance.quantile([0.25, 0.75])
        stats['puissance_q1'] = quartiles[0.25]
        stats['puissance_q3'] = quartiles[0.75]
        stats['puissance_iqr'] = quartiles[0.75] - quartiles[0.25]

        # Énergie totale consommée
        if len(df) > 1:
            energie_debut = df['energie_import'].iloc[0]
            energie_fin = df['energie_import'].iloc[-1]
            stats['energie_totale'] = energie_fin - energie_debut

            # Calcul du temps écoulé
            temps_total = (df['timestamp'].iloc[-1] - df['timestamp'].iloc[0]).total_seconds() / 3600
            stats['puissance_moyenne_ponderee'] = stats['energie_totale'] / temps_total if temps_total > 0 else 0

        # Analyse spectrale (FFT pour détecter les fréquences)
        if len(puissance) > 10:
            spectre = self._analyse_spectrale(puissance.values)
            stats['frequences_dominantes'] = spectre['frequences'][:5]  # Top 5 fréquences

        return stats

    def _detecter_saisonnalite(self, df: pd.DataFrame) -> Dict:
        """Détection de patterns saisonniers"""
        if len(df) < 24:  # Minimum 24h de données
            return {'detectee': False, 'raison': 'Données insuffisantes'}

        # Analyse par heure de la journée
        df_copy = df.copy()
        df_copy['heure'] = df_copy['timestamp'].dt.hour

        conso_par_heure = df_copy.groupby('heure')['puissance'].mean()

        # Calcul du coefficient de variation
        cv = conso_par_heure.std() / conso_par_heure.mean() if conso_par_heure.mean() != 0 else 0

        # Test de saisonnalité (coefficient de variation > 0.3 = saisonnier)
        saisonnier = cv > 0.3

        # Heures de pointe
        seuil_pointe = conso_par_heure.quantile(0.8)
        heures_pointe = conso_par_heure[conso_par_heure > seuil_pointe].index.tolist()

        return {
            'detectee': saisonnier,
            'coefficient_variation': cv,
            'profil_horaire': conso_par_heure.to_dict(),
            'heures_pointe': heures_pointe,
            'heure_max_conso': conso_par_heure.idxmax(),
            'heure_min_conso': conso_par_heure.idxmin()
        }

    def _analyser_tendances(self, df: pd.DataFrame) -> Dict:
        """Analyse des tendances temporelles"""
        if len(df) < 10:
            return {'erreur': 'Données insuffisantes'}

        # Test de tendance (régression linéaire)
        x = np.arange(len(df))
        y = df['puissance'].values

        slope, intercept, r_value, p_value, std_err = stats.linregress(x, y)

        # Tendance significative si p_value < 0.05
        tendance_significative = p_value < 0.05
        tendance_type = 'croissante' if slope > 0 else 'decroissante'

        # Calcul du taux de variation
        variation_relative = (slope * len(df)) / df['puissance'].mean() * 100

        # Analyse de la volatilité
        volatilite = df['puissance'].std() / df['puissance'].mean()

        return {
            'tendance_significative': tendance_significative,
            'type_tendance': tendance_type if tendance_significative else 'stable',
            'pente': slope,
            'coefficient_correlation': r_value**2,
            'variation_relative_pourcent': variation_relative,
            'volatilite': volatilite,
            'interpreation': self._interpreter_tendance(tendance_significative, slope, volatilite)
        }

    def _analyse_spectrale(self, signal: np.ndarray) -> Dict:
        """Analyse spectrale du signal (FFT)"""
        # FFT du signal
        fft = np.fft.fft(signal)
        freqs = np.fft.fftfreq(len(signal))

        # Magnitude
        magnitude = np.abs(fft)

        # Fréquences positives uniquement
        freqs_pos = freqs[:len(freqs)//2]
        magnitude_pos = magnitude[:len(magnitude)//2]

        # Fréquences dominantes
        indices_tries = np.argsort(magnitude_pos)[::-1]
        frequences_dom = []

        for idx in indices_tries[:10]:  # Top 10
            freq = freqs_pos[idx]
            mag = magnitude_pos[idx]

            # Conversion en fréquence temporelle (si échantillonnage connu)
            # Ici on retourne les indices relatifs
            frequences_dom.append({
                'frequence_relative': freq,
                'magnitude': mag,
                'periode_estimee': 1/abs(freq) if freq != 0 else 0
            })

        return {
            'frequences': frequences_dom,
            'frequence_nyquist': 0.5,  # Fréquence de Nyquist normalisée
            'puissance_totale': np.sum(magnitude_pos**2)
        }

    def _interpreter_tendance(self, significative: bool, slope: float, volatilite: float) -> str:
        """Interprétation de la tendance"""
        if not significative:
            if volatilite > 0.5:
                return "Consommation stable mais très variable"
            else:
                return "Consommation stable et régulière"

        if abs(slope) < 1:  # Faible variation
            tendance = "légèrement croissante" if slope > 0 else "légèrement décroissante"
        elif abs(slope) < 5:
            tendance = "croissance modérée" if slope > 0 else "décroissance modérée"
        else:
            tendance = "forte croissance" if slope > 0 else "forte décroissance"

        if volatilite > 0.3:
            tendance += " avec forte variabilité"

        return f"Consommation en {tendance}"

    def comparer_periodes(self, periode1: Tuple[datetime, datetime],
                         periode2: Tuple[datetime, datetime]) -> Dict:
        """
        Comparaison de deux périodes similaires

        Args:
            periode1, periode2: Tuples (debut, fin) des périodes à comparer

        Returns:
            Comparaison détaillée
        """
        analyse1 = self.analyser_periode(periode1[0], periode1[1])
        analyse2 = self.analyser_periode(periode2[0], periode2[1])

        if 'erreur' in analyse1 or 'erreur' in analyse2:
            return {'erreur': 'Données insuffisantes pour une période'}

        stats1 = analyse1['statistiques_globales']
        stats2 = analyse2['statistiques_globales']

        # Comparaisons
        comparaison = {
            'periode1': {
                'debut': periode1[0].isoformat(),
                'fin': periode1[1].isoformat(),
                'puissance_moyenne': stats1['puissance_moyenne']
            },
            'periode2': {
                'debut': periode2[0].isoformat(),
                'fin': periode2[1].isoformat(),
                'puissance_moyenne': stats2['puissance_moyenne']
            },
            'difference_puissance': stats2['puissance_moyenne'] - stats1['puissance_moyenne'],
            'variation_pourcent': ((stats2['puissance_moyenne'] - stats1['puissance_moyenne']) / stats1['puissance_moyenne']) * 100 if stats1['puissance_moyenne'] != 0 else 0,
            'difference_volatilite': stats2.get('volatilite', 0) - stats1.get('volatilite', 0)
        }

        # Interprétation
        if abs(comparaison['variation_pourcent']) < 5:
            comparaison['interpretation'] = "Consommation stable entre les périodes"
        elif comparaison['variation_pourcent'] > 5:
            comparaison['interpretation'] = f"Augmentation de {abs(comparaison['variation_pourcent']):.1f}% de la consommation"
        else:
            comparaison['interpretation'] = f"Diminution de {abs(comparaison['variation_pourcent']):.1f}% de la consommation"

        return comparaison

    def detecter_changements_comportement(self, historique_jours: int = 30) -> List[Dict]:
        """
        Détection de changements significatifs dans le comportement

        Args:
            historique_jours: Nombre de jours d'historique à analyser

        Returns:
            Liste des changements détectés
        """
        date_limite = datetime.utcnow() - timedelta(days=historique_jours)

        mesures = MesureEnergie.query.filter(
            MesureEnergie.timestamp >= date_limite
        ).order_by(MesureEnergie.timestamp).all()

        if len(mesures) < 48:  # Minimum 2 jours
            return []

        # Conversion en DataFrame
        df = pd.DataFrame([{
            'timestamp': m.timestamp,
            'puissance': m.puissance_active_totale or 0,
            'jour': m.timestamp.date()
        } for m in mesures])

        changements = []

        # Analyse par jour
        for jour in df['jour'].unique():
            data_jour = df[df['jour'] == jour]

            if len(data_jour) < 6:  # Minimum 6 mesures par jour
                continue

            puissance_moyenne = data_jour['puissance'].mean()

            # Comparaison avec les 7 jours précédents
            jour_prec = jour - timedelta(days=7)
            data_reference = df[(df['jour'] >= jour_prec) & (df['jour'] < jour)]

            if len(data_reference) > 0:
                puissance_ref = data_reference['puissance'].mean()
                variation = ((puissance_moyenne - puissance_ref) / puissance_ref) * 100 if puissance_ref != 0 else 0

                # Détection de changement significatif (>20%)
                if abs(variation) > 20:
                    changements.append({
                        'date': jour.isoformat(),
                        'type_changement': 'augmentation' if variation > 0 else 'diminution',
                        'variation_pourcent': variation,
                        'puissance_jour': puissance_moyenne,
                        'puissance_reference': puissance_ref,
                        'severite': 'majeur' if abs(variation) > 50 else 'modere'
                    })

        return changements
```

### API pour les analyses statistiques

```python
# app/routes/api_analyse.py
from flask import Blueprint, request, jsonify
from app.services.analyse_statistique import AnalyseurStatistique
from datetime import datetime, timedelta

analyse_bp = Blueprint('analyse', __name__)
analyseur = AnalyseurStatistique()

@analyse_bp.route('/api/analyse/periode')
def analyser_periode():
    """Analyse statistique d'une période"""

    # Paramètres
    heures = int(request.args.get('hours', 24))
    periode_groupement = request.args.get('group_by', 'horaire')

    debut = datetime.utcnow() - timedelta(hours=heures)
    fin = datetime.utcnow()

    # Validation des paramètres
    if periode_groupement not in ['horaire', 'journaliere', 'hebdomadaire', 'mensuelle']:
        return jsonify({'error': 'Période de groupement invalide'}), 400

    # Analyse
    resultats = analyseur.analyser_periode(debut, fin, periode_groupement)

    return jsonify(resultats)

@analyse_bp.route('/api/analyse/comparer')
def comparer_periodes():
    """Comparaison de deux périodes"""

    # Période 1
    p1_debut_str = request.args.get('p1_start')
    p1_fin_str = request.args.get('p1_end')

    # Période 2
    p2_debut_str = request.args.get('p2_start')
    p2_fin_str = request.args.get('p2_end')

    if not all([p1_debut_str, p1_fin_str, p2_debut_str, p2_fin_str]):
        return jsonify({'error': 'Toutes les périodes doivent être spécifiées'}), 400

    try:
        p1_debut = datetime.fromisoformat(p1_debut_str.replace('Z', '+00:00'))
        p1_fin = datetime.fromisoformat(p1_fin_str.replace('Z', '+00:00'))
        p2_debut = datetime.fromisoformat(p2_debut_str.replace('Z', '+00:00'))
        p2_fin = datetime.fromisoformat(p2_fin_str.replace('Z', '+00:00'))

        comparaison = analyseur.comparer_periodes(
            (p1_debut, p1_fin),
            (p2_debut, p2_fin)
        )

        return jsonify(comparaison)

    except ValueError as e:
        return jsonify({'error': f'Format de date invalide: {e}'}), 400

@analyse_bp.route('/api/analyse/tendances')
def analyser_tendances():
    """Analyse des tendances actuelles"""

    jours = int(request.args.get('days', 7))

    # Analyse sur les N derniers jours
    debut = datetime.utcnow() - timedelta(days=jours)
    fin = datetime.utcnow()

    analyse = analyseur.analyser_periode(debut, fin)

    if 'erreur' in analyse:
        return jsonify({'error': analyse['erreur']}), 400

    # Focus sur les tendances
    return jsonify({
        'periode_analyse': f"{jours} jours",
        'tendances': analyse['tendances'],
        'saisonnalite': analyse['saisonnalite'],
        'statistiques_cles': {
            'puissance_moyenne': analyse['statistiques_globales']['puissance_moyenne'],
            'volatilite': analyse['statistiques_globales'].get('volatilite', 0),
            'coefficient_variation': analyse['saisonnalite'].get('coefficient_variation', 0)
        }
    })

@analyse_bp.route('/api/analyse/changements')
def detecter_changements():
    """Détection des changements de comportement"""

    jours = int(request.args.get('days', 30))

    changements = analyseur.detecter_changements_comportement(jours)

    return jsonify({
        'periode_analyse': f"{jours} jours",
        'nb_changements': len(changements),
        'changements': changements
    })

@analyse_bp.route('/api/analyse/profil-quotidien')
def profil_quotidien():
    """Profil de consommation moyen par heure"""

    jours = int(request.args.get('days', 7))

    debut = datetime.utcnow() - timedelta(days=jours)
    fin = datetime.utcnow()

    analyse = analyseur.analyser_periode(debut, fin, 'horaire')

    if 'erreur' in analyse:
        return jsonify({'error': analyse['erreur']}), 400

    # Extraction du profil horaire moyen
    heures = {}
    for periode in analyse['analyse_par_periode']['periodes']:
        heure = datetime.fromisoformat(periode['periode']).hour
        if heure not in heures:
            heures[heure] = []
        heures[heure].append(periode['puissance_moyenne'])

    # Moyenne par heure
    profil = {}
    for heure, valeurs in heures.items():
        profil[heure] = sum(valeurs) / len(valeurs) if valeurs else 0

    return jsonify({
        'periode_reference': f"{jours} derniers jours",
        'profil_horaire': profil,
        'heure_pic': max(profil.items(), key=lambda x: x[1])[0] if profil else None,
        'heure_creux': min(profil.items(), key=lambda x: x[1])[0] if profil else None
    })
```

### Template pour les analyses

```html
<!-- templates/analyse.html -->
{% extends "base.html" %}

{% block title %}Analyse Avancée - Compteur E450{% endblock %}

{% block extra_head %}
<link rel="stylesheet" href="{{ url_for('static', filename='css/analyse.css') }}">
<script src="https://cdn.jsdelivr.net/npm/chart.js"></script>
<script src="https://cdn.jsdelivr.net/npm/chartjs-adapter-date-fns"></script>
{% endblock %}

{% block content %}
<div class="analyse-container">
    <!-- Sélecteurs d'analyse -->
    <div class="analyse-controls">
        <div class="control-group">
            <label>Période d'analyse:</label>
            <select id="periode-select">
                <option value="24">24 heures</option>
                <option value="168">7 jours</option>
                <option value="720">30 jours</option>
            </select>
        </div>

        <div class="control-group">
            <label>Type d'analyse:</label>
            <select id="analyse-select">
                <option value="tendances">Tendances</option>
                <option value="saisonnalite">Saisonnalité</option>
                <option value="comparaison">Comparaison</option>
                <option value="changements">Changements</option>
            </select>
        </div>

        <button id="analyser-btn" class="btn btn-primary">Analyser</button>
    </div>

    <!-- Résultats d'analyse -->
    <div class="analyse-resultats" id="analyse-resultats">
        <!-- Chargé dynamiquement -->
    </div>

    <!-- Graphiques -->
    <div class="analyse-charts">
        <div class="chart-container">
            <canvas id="analyse-chart"></canvas>
        </div>
    </div>
</div>
{% endblock %}

{% block extra_scripts %}
<script src="{{ url_for('static', filename='js/analyse.js') }}"></script>
{% endblock %}
```

> **💡 À retenir** : L'analyse statistique transforme vos données brutes en insights exploitables, révélant des patterns cachés dans votre consommation énergétique.

> **⚠️ Astuce** : Commencez par des analyses simples (moyennes, tendances) avant d'explorer des méthodes plus sophistiquées comme l'analyse spectrale ou les modèles prédictifs.

Dans le prochain chapitre, nous explorerons les **courbes de consommation** avec des visualisations avancées !

---

**Navigation**
- [Chapitre précédent : Historique des anomalies détectées](../Partie_IV_Connectivite_Integration/Chapitre_19_Historique_Anomalies.md)
- [Chapitre suivant : Courbes de consommation](Chapitre_21_Courbes_Consommation.md)
- [Retour à la table des matières](../../README.md)
