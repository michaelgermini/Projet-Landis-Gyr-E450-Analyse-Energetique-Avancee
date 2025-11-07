# 📊 Chapitre 23 : Prévision énergétique avec Prophet

## 🔮 Prédiction de la consommation future

Prophet est une bibliothèque de prévision développée par Facebook (Meta) qui excelle dans la prédiction de séries temporelles avec saisonnalité.

### Installation et configuration de Prophet

```bash
# Installation de Prophet
pip install prophet

# Pour les calculs statistiques avancés
pip install scikit-learn numpy pandas matplotlib
```

### Service de prévision avec Prophet

```python
# app/services/prediction_service.py
from app.models import MesureEnergie
from prophet import Prophet
import pandas as pd
from datetime import datetime, timedelta
from typing import Dict, List, Optional, Tuple
import logging
import numpy as np

logger = logging.getLogger(__name__)

class ServicePrediction:
    """Service de prévision énergétique avec Prophet"""

    def __init__(self):
        self.min_donnees_prediction = 168  # Minimum 7 jours de données (168h)
        self.periode_prediction_max = 168  # Maximum 7 jours de prédiction

    def prevoir_consommation(self, heures_historique: int = 720,
                           heures_prediction: int = 24) -> Dict:
        """
        Prévision de la consommation future

        Args:
            heures_historique: Nombre d'heures d'historique à utiliser
            heures_prediction: Nombre d'heures à prévoir

        Returns:
            Dictionnaire avec prévisions et métriques
        """
        # Récupération des données historiques
        debut = datetime.utcnow() - timedelta(hours=heures_historique)
        mesures = MesureEnergie.query.filter(
            MesureEnergie.timestamp >= debut
        ).order_by(MesureEnergie.timestamp).all()

        if len(mesures) < self.min_donnees_prediction:
            return {
                'erreur': f'Données insuffisantes. Minimum {self.min_donnees_prediction}h requises, {len(mesures)}h disponibles'
            }

        # Préparation des données pour Prophet
        df = self._preparer_donnees_prophet(mesures)

        if df.empty:
            return {'erreur': 'Aucune donnée valide pour la prévision'}

        # Entraînement du modèle
        modele = self._entrainer_modele_prophet(df)

        # Génération des prédictions
        predictions = self._generer_predictions(modele, heures_prediction)

        # Calcul des métriques de performance
        metriques = self._calculer_metriques_performance(df, modele)

        # Analyse des composantes
        composantes = self._analyser_composantes(modele, predictions)

        return {
            'predictions': predictions,
            'metriques': metriques,
            'composantes': composantes,
            'parametres': {
                'heures_historique': heures_historique,
                'heures_prediction': heures_prediction,
                'nb_mesures_entrainement': len(df)
            }
        }

    def _preparer_donnees_prophet(self, mesures: List[MesureEnergie]) -> pd.DataFrame:
        """Préparation des données pour Prophet"""

        donnees = []
        for mesure in mesures:
            if mesure.puissance_active_totale is not None and mesure.puissance_active_totale > 0:
                donnees.append({
                    'ds': mesure.timestamp,  # DateStamp pour Prophet
                    'y': mesure.puissance_active_totale  # Valeur à prédire
                })

        df = pd.DataFrame(donnees)

        # Agrégation par heure si nécessaire (pour réduire le bruit)
        df['heure'] = df['ds'].dt.floor('H')
        df_agg = df.groupby('heure').agg({
            'y': 'mean',
            'ds': 'first'  # Garde le premier timestamp de l'heure
        }).reset_index(drop=True)

        return df_agg[['ds', 'y']]

    def _entrainer_modele_prophet(self, df: pd.DataFrame) -> Prophet:
        """Entraînement du modèle Prophet"""

        # Configuration du modèle
        modele = Prophet(
            yearly_seasonality=False,  # Pas de saisonnalité annuelle (trop court)
            weekly_seasonality=True,   # Saisonnalité hebdomadaire
            daily_seasonality=True,    # Saisonnalité journalière
            changepoint_prior_scale=0.05,  # Sensibilité aux changements
            seasonality_prior_scale=10,    # Force des saisonnalités
            interval_width=0.8             # Intervalle de confiance 80%
        )

        # Ajout de saisonnalités personnalisées
        # Saisonnalité horaire plus fine
        modele.add_seasonality(
            name='horaire_detaillee',
            period=24,
            fourier_order=8  # Plus de détails pour les patterns horaires
        )

        # Entraînement
        modele.fit(df)

        return modele

    def _generer_predictions(self, modele: Prophet, heures_prediction: int) -> Dict:
        """Génération des prédictions"""

        # Création du DataFrame futur
        future = modele.make_future_dataframe(
            periods=heures_prediction,
            freq='H',  # Prédictions horaires
            include_history=False  # Uniquement les prédictions
        )

        # Prédiction
        forecast = modele.predict(future)

        # Formatage des résultats
        predictions = []
        for _, row in forecast.iterrows():
            predictions.append({
                'timestamp': row['ds'].isoformat(),
                'prediction': round(row['yhat'], 1),
                'borne_inf': round(row['yhat_lower'], 1),
                'borne_sup': round(row['yhat_upper'], 1),
                'trend': round(row['trend'], 1) if 'trend' in row else None,
                'seasonalite_quotidienne': round(row['daily'], 1) if 'daily' in row else None,
                'seasonalite_hebdomadaire': round(row['weekly'], 1) if 'weekly' in row else None
            })

        return {
            'predictions': predictions,
            'periode_prediction': {
                'debut': predictions[0]['timestamp'] if predictions else None,
                'fin': predictions[-1]['timestamp'] if predictions else None,
                'heures': len(predictions)
            },
            'stats_predictions': {
                'moyenne': round(np.mean([p['prediction'] for p in predictions]), 1),
                'ecart_type': round(np.std([p['prediction'] for p in predictions]), 1),
                'minimum': round(min([p['prediction'] for p in predictions]), 1),
                'maximum': round(max([p['prediction'] for p in predictions]), 1)
            }
        }

    def _calculer_metriques_performance(self, df_historique: pd.DataFrame,
                                       modele: Prophet) -> Dict:
        """Calcul des métriques de performance du modèle"""

        # Validation croisée
        from prophet.diagnostics import cross_validation, performance_metrics

        try:
            # Validation croisée (si assez de données)
            if len(df_historique) > 168:  # Plus de 7 jours
                df_cv = cross_validation(
                    modele,
                    initial='168 hours',  # Période initiale d'entraînement
                    period='24 hours',    # Période entre cutoffs
                    horizon='24 hours'    # Horizon de prédiction
                )

                df_metrics = performance_metrics(df_cv)

                metriques = {
                    'mape': round(df_metrics['mape'].mean(), 4),  # Mean Absolute Percentage Error
                    'mae': round(df_metrics['mae'].mean(), 1),    # Mean Absolute Error
                    'rmse': round(df_metrics['rmse'].mean(), 1),   # Root Mean Square Error
                    'validation_effectuee': True
                }
            else:
                # Métriques simples sur les données d'entraînement
                forecast_train = modele.predict(df_historique)
                y_true = df_historique['y'].values
                y_pred = forecast_train['yhat'].values

                mape = np.mean(np.abs((y_true - y_pred) / y_true)) * 100
                mae = np.mean(np.abs(y_true - y_pred))
                rmse = np.sqrt(np.mean((y_true - y_pred)**2))

                metriques = {
                    'mape': round(mape, 2),
                    'mae': round(mae, 1),
                    'rmse': round(rmse, 1),
                    'validation_effectuee': False
                }

        except Exception as e:
            logger.warning(f"Erreur calcul métriques: {e}")
            metriques = {
                'mape': None,
                'mae': None,
                'rmse': None,
                'validation_effectuee': False,
                'erreur': str(e)
            }

        return metriques

    def _analyser_composantes(self, modele: Prophet, predictions: Dict) -> Dict:
        """Analyse des composantes de la prédiction"""

        try:
            # Récupération des composantes du modèle
            future = modele.make_future_dataframe(
                periods=24,  # Analyse sur 24h
                freq='H'
            )
            forecast = modele.predict(future)

            # Force du trend
            trend_strength = abs(forecast['trend'].iloc[-1] - forecast['trend'].iloc[0])

            # Analyse des saisonnalités
            daily_amplitude = forecast['daily'].max() - forecast['daily'].min()
            weekly_amplitude = forecast['weekly'].max() - forecast['weekly'].min() if 'weekly' in forecast.columns else 0

            return {
                'force_trend': round(trend_strength, 1),
                'amplitude_saisonnalite_quotidienne': round(daily_amplitude, 1),
                'amplitude_saisonnalite_hebdomadaire': round(weekly_amplitude, 1),
                'interpretation': self._interpreter_composantes(trend_strength, daily_amplitude, weekly_amplitude)
            }

        except Exception as e:
            logger.error(f"Erreur analyse composantes: {e}")
            return {'erreur': str(e)}

    def _interpreter_composantes(self, trend: float, daily_amp: float, weekly_amp: float) -> str:
        """Interprétation des composantes"""

        interpretations = []

        # Trend
        if trend < 10:
            interpretations.append("consommation stable")
        elif trend < 50:
            interpretations.append("légère évolution de la consommation")
        else:
            interpretations.append("forte évolution de la consommation")

        # Saisonnalité quotidienne
        if daily_amp > 100:
            interpretations.append("forts variations quotidiens")
        elif daily_amp > 50:
            interpretations.append("variations quotidiennes modérées")
        else:
            interpretations.append("variations quotidiennes faibles")

        # Saisonnalité hebdomadaire
        if weekly_amp > 50:
            interpretations.append("différences marquées entre jours de semaine/week-end")

        if not interpretations:
            return "Consommation régulière sans patterns marqués"

        return "Consommation avec " + ", ".join(interpretations)

    def prevoir_couts_futurs(self, heures_prediction: int = 24) -> Dict:
        """
        Prévision des coûts futurs basée sur les prédictions de consommation

        Args:
            heures_prediction: Nombre d'heures à prévoir

        Returns:
            Prévision des coûts
        """
        from app.services.tarifs_energie import GestionnaireTarifs

        tarifs = GestionnaireTarifs()

        # Récupération des prévisions de consommation
        previsions = self.prevoir_consommation(heures_prediction=heures_prediction)

        if 'erreur' in previsions:
            return {'erreur': previsions['erreur']}

        couts_predits = []
        cout_total = 0

        for prediction in previsions['predictions']['predictions']:
            puissance = prediction['prediction']
            timestamp_str = prediction['timestamp']
            timestamp = datetime.fromisoformat(timestamp_str.replace('Z', '+00:00'))

            # Calcul du coût pour cette heure
            cout_heure = tarifs.calculer_cout_consommation(puissance, 1.0, timestamp)

            couts_predits.append({
                'timestamp': timestamp_str,
                'puissance_prevue': puissance,
                'cout_heure': cout_heure['cout_total'],
                'tranche': cout_heure['tranche']
            })

            cout_total += cout_heure['cout_total']

        # Statistiques des coûts
        couts_valeurs = [c['cout_heure'] for c in couts_predits]

        return {
            'periode_prediction': f"{heures_prediction} heures",
            'cout_total_prevu': round(cout_total, 2),
            'cout_moyen_horaire': round(cout_total / heures_prediction, 3),
            'cout_max_horaire': round(max(couts_valeurs), 3),
            'cout_min_horaire': round(min(couts_valeurs), 3),
            'evolution_couts': couts_predits,
            'abonnement_mensuel': tarifs.abonnements['bleu'],
            'estimation_mensuelle': round(cout_total * (720 / heures_prediction), 2)  # Approximation mensuelle
        }

    def optimiser_parametres_modele(self, df: pd.DataFrame) -> Dict:
        """
        Optimisation des paramètres du modèle Prophet

        Args:
            df: DataFrame des données d'entraînement

        Returns:
            Meilleurs paramètres trouvés
        """
        from sklearn.model_selection import ParameterGrid

        # Grille de paramètres à tester
        param_grid = {
            'changepoint_prior_scale': [0.01, 0.05, 0.1, 0.5],
            'seasonality_prior_scale': [0.1, 1.0, 10.0],
            'changepoint_range': [0.8, 0.9, 0.95]
        }

        meilleures_metriques = {'mae': float('inf')}
        meilleurs_params = None

        grid = ParameterGrid(param_grid)

        for params in grid:
            try:
                # Entraînement avec ces paramètres
                modele = Prophet(
                    yearly_seasonality=False,
                    weekly_seasonality=True,
                    daily_seasonality=True,
                    **params
                )
                modele.fit(df)

                # Validation simple
                forecast = modele.predict(df)
                mae = np.mean(np.abs(df['y'].values - forecast['yhat'].values))

                if mae < meilleures_metriques['mae']:
                    meilleures_metriques = {
                        'mae': mae,
                        'mape': np.mean(np.abs((df['y'].values - forecast['yhat'].values) / df['y'].values)) * 100
                    }
                    meilleurs_params = params

            except Exception as e:
                logger.debug(f"Erreur avec params {params}: {e}")
                continue

        return {
            'meilleurs_parametres': meilleurs_params,
            'metriques': meilleures_metriques
        }
```

### API de prévision

```python
# app/routes/api_prediction.py
from flask import Blueprint, request, jsonify
from app.services.prediction_service import ServicePrediction
from datetime import datetime, timedelta

prediction_bp = Blueprint('prediction', __name__)
prediction_service = ServicePrediction()

@prediction_bp.route('/api/predict/consumption')
def predict_consumption():
    """Prédiction de la consommation future"""

    # Paramètres
    heures_historique = int(request.args.get('history_hours', 720))  # 30 jours par défaut
    heures_prediction = int(request.args.get('predict_hours', 24))   # 24h par défaut

    # Validation
    if heures_prediction > prediction_service.periode_prediction_max:
        return jsonify({
            'error': f'Période de prédiction maximale: {prediction_service.periode_prediction_max}h'
        }), 400

    # Prédiction
    result = prediction_service.prevoir_consommation(heures_historique, heures_prediction)

    if 'erreur' in result:
        return jsonify({'error': result['erreur']}), 400

    return jsonify(result)

@prediction_bp.route('/api/predict/costs')
def predict_costs():
    """Prédiction des coûts futurs"""

    heures_prediction = int(request.args.get('hours', 168))  # 7 jours par défaut

    result = prediction_service.prevoir_couts_futurs(heures_prediction)

    if 'erreur' in result:
        return jsonify({'error': result['erreur']}), 400

    return jsonify(result)

@prediction_bp.route('/api/predict/optimize')
def optimize_model():
    """Optimisation des paramètres du modèle"""

    heures_historique = int(request.args.get('history_hours', 720))

    # Récupération des données
    debut = datetime.utcnow() - timedelta(hours=heures_historique)
    mesures = MesureEnergie.query.filter(
        MesureEnergie.timestamp >= debut
    ).order_by(MesureEnergie.timestamp).all()

    if len(mesures) < 168:
        return jsonify({
            'error': f'Données insuffisantes. {len(mesures)} mesures, minimum 168 requises'
        }), 400

    # Préparation des données
    df = prediction_service._preparer_donnees_prophet(mesures)

    # Optimisation
    result = prediction_service.optimiser_parametres_modele(df)

    return jsonify(result)

@prediction_bp.route('/api/predict/info')
def prediction_info():
    """Informations sur le service de prédiction"""

    return jsonify({
        'service': 'Prophet',
        'version': '1.1',
        'min_donnees_requises': prediction_service.min_donnees_prediction,
        'max_prediction_heures': prediction_service.periode_prediction_max,
        'capacites': [
            'Prévision de consommation horaire',
            'Décomposition tendance/saisonnalité',
            'Prédiction de coûts',
            'Optimisation automatique des paramètres',
            'Validation croisée'
        ],
        'metriques_performance': [
            'MAPE (Mean Absolute Percentage Error)',
            'MAE (Mean Absolute Error)',
            'RMSE (Root Mean Square Error)'
        ]
    })
```

### Interface de prévision

```html
<!-- templates/prediction.html -->
{% extends "base.html" %}

{% block title %}Prévisions Énergétiques - Compteur E450{% endblock %}

{% block extra_head %}
<link rel="stylesheet" href="{{ url_for('static', filename='css/prediction.css') }}">
<script src="https://cdn.jsdelivr.net/npm/chart.js"></script>
<script src="https://cdn.jsdelivr.net/npm/chartjs-adapter-date-fns"></script>
{% endblock %}

{% block content %}
<div class="prediction-container">
    <div class="prediction-header">
        <h1>🔮 Prévisions Énergétiques</h1>
        <p>Prévision de votre consommation future avec Prophet</p>
    </div>

    <div class="prediction-controls">
        <div class="control-group">
            <label>Historique utilisé:</label>
            <select id="history-select">
                <option value="168">7 jours</option>
                <option value="336">14 jours</option>
                <option value="720" selected>30 jours</option>
            </select>
        </div>

        <div class="control-group">
            <label>Prévision sur:</label>
            <select id="prediction-select">
                <option value="24" selected>24 heures</option>
                <option value="72">3 jours</option>
                <option value="168">7 jours</option>
            </select>
        </div>

        <button id="predict-btn" class="btn btn-primary">🔮 Générer Prévision</button>
    </div>

    <!-- Résultats de prévision -->
    <div class="prediction-results" id="prediction-results" style="display: none;">
        <div class="prediction-summary">
            <div class="summary-card">
                <h3>📊 Statistiques</h3>
                <div id="prediction-stats">
                    <!-- Chargé dynamiquement -->
                </div>
            </div>

            <div class="summary-card">
                <h3>💰 Coûts Prévisionnels</h3>
                <div id="prediction-costs">
                    <!-- Chargé dynamiquement -->
                </div>
            </div>

            <div class="summary-card">
                <h3>📈 Performance Modèle</h3>
                <div id="prediction-metrics">
                    <!-- Chargé dynamiquement -->
                </div>
            </div>
        </div>

        <div class="prediction-chart">
            <canvas id="prediction-chart"></canvas>
        </div>

        <div class="prediction-details">
            <h3>🔍 Détails des Prévisions</h3>
            <div id="prediction-details">
                <!-- Table détaillée -->
            </div>
        </div>
    </div>

    <!-- Historique des prévisions -->
    <div class="prediction-history">
        <h3>📚 Historique des Prévisions</h3>
        <div id="prediction-history">
            <!-- Chargé dynamiquement -->
        </div>
    </div>
</div>
{% endblock %}

{% block extra_scripts %}
<script src="{{ url_for('static', filename='js/prediction.js') }}"></script>
{% endblock %}
```

> **💡 À retenir** : Prophet excelle dans la prévision de séries temporelles avec saisonnalité, offrant des prédictions fiables pour optimiser votre consommation énergétique.

> **⚠️ Astuce** : Commencez avec des horizons de prédiction courts (24h) pour valider la qualité du modèle avant d'étendre aux prévisions plus longues.

Félicitations ! La Partie V sur l'analyse et la visualisation avancée est maintenant complète. Vous maîtrisez l'analyse statistique, les visualisations, l'export de données et la prévision avec Prophet. La Partie VI va explorer les cas d'usage pratiques !

---

**Navigation**
- [Chapitre précédent : Export CSV/Excel](Chapitre_22_Export_CSV_Excel.md)
- [Partie VI : Cas d'usage et exemples pratiques](../Partie_VI_Cas_Usage/)
- [Retour à la table des matières](../../README.md)
