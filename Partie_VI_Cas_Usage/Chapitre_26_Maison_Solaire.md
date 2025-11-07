# 🌞 Chapitre 26 : Maison solaire

## ☀️ Analyse production/consommation

Dans une maison solaire, le compteur E450 permet d'analyser finement l'équilibre entre production photovoltaïque et consommation domestique, optimisant ainsi l'autoconsommation.

### Architecture solaire domestique

#### Configuration typique

```
Maison solaire ──┬──► Panneaux PV (3-9kWc)
                │
                ├──► Onduleur solaire
                │
                ├──► Compteur E450 (avec injection)
                │
                ├──► Tableau électrique
                │
                └──► Système domotique
```

#### Données disponibles

- **Production solaire** : Via injection réseau (2.8.0)
- **Consommation** : Énergie importée (1.8.0)
- **Autoconsommation** : Calculée (production - injection)
- **Taux autoconsommation** : Ratio production consommée localement

### Analyse de l'autoconsommation

#### Calculateur solaire spécialisé

```python
class AnalyseurSolaire:
    """Analyseur spécialisé pour installations solaires"""

    def __init__(self):
        self.puissance_panneaux = 3000  # Wc crête
        self.rendement_systeme = 0.85   # 85% rendement global

    def analyser_bilan_energetique(self, mesures_compteur, donnees_meteo=None):
        """
        Analyse complète du bilan énergétique solaire

        Args:
            mesures_compteur: Données du compteur E450
            donnees_meteo: Données météo (optionnel)

        Returns:
            Analyse du bilan énergétique
        """
        # Données du compteur
        energie_import = mesures_compteur.get('energie_active_import', 0)
        energie_export = mesures_compteur.get('energie_active_export', 0)
        puissance_actuelle = mesures_compteur.get('puissance_active_totale', 0)

        # Calculs de base
        autoconsommation = max(0, energie_import - energie_export)
        taux_autoconsommation = autoconsommation / energie_import if energie_import > 0 else 0

        # Estimation production solaire
        # Hypothèse: l'excédent exporté = production non consommée
        production_estimee = energie_export + autoconsommation

        # Performance système
        if donnees_meteo and 'ensoleillement' in donnees_meteo:
            production_theorique = self._calculer_production_theorique(donnees_meteo)
            rendement_reel = production_estimee / production_theorique if production_theorique > 0 else 0
        else:
            rendement_reel = None

        # Évaluation économique
        economie = self._calculer_economie(autoconsommation, energie_export)

        return {
            'periode_analyse': 'jour',  # ou heure, semaine selon contexte
            'consommation_totale': energie_import,
            'production_estimee': production_estimee,
            'autoconsommation': autoconsommation,
            'injection_reseau': energie_export,
            'taux_autoconsommation': taux_autoconsommation * 100,  # %
            'rendement_systeme': rendement_reel * 100 if rendement_reel else None,
            'economie_estimee': economie,
            'evaluation': self._evaluer_performance_solaire(taux_autoconsommation, rendement_reel)
        }

    def _calculer_production_theorique(self, meteo):
        """Calcule la production théorique basée sur météo"""
        ensoleillement = meteo.get('ensoleillement', 0)  # kWh/m²/jour
        temperature = meteo.get('temperature', 25)      # °C

        # Facteur de performance (dépend de la température)
        facteur_temp = 1 - 0.004 * (temperature - 25)  # -0.4% par °C au-dessus de 25°C

        # Production théorique
        production_theorique = (
            self.puissance_panneaux / 1000 *      # kWc
            ensoleillement *                     # kWh/m²/jour
            facteur_temp *                       # Correction température
            self.rendement_systeme                # Rendement système
        )

        return production_theorique

    def _calculer_economie(self, autoconsommation, injection):
        """Calcule l'économie réalisée"""
        # Tarifs différenciés
        tarif_autoconsommation = 0.23  # €/kWh (prix de détail)
        tarif_injection = 0.13         # €/kWh (prime injection)

        economie_autoconsommation = autoconsommation * tarif_autoconsommation
        revenu_injection = injection * tarif_injection

        return {
            'economie_autoconsommation': economie_autoconsommation,
            'revenu_injection': revenu_injection,
            'total_economie': economie_autoconsommation + revenu_injection,
            'cout_avec_achat': autoconsommation * 0.15  # Coût si achat réseau
        }

    def _evaluer_performance_solaire(self, taux_auto, rendement):
        """Évalue la performance du système solaire"""
        if taux_auto >= 0.7:
            performance = "excellente"
            note = "A"
        elif taux_auto >= 0.6:
            performance = "très bonne"
            note = "B"
        elif taux_auto >= 0.5:
            performance = "bonne"
            note = "C"
        elif taux_auto >= 0.4:
            performance = "moyenne"
            note = "D"
        else:
            performance = "à améliorer"
            note = "E"

        recommandations = []
        if taux_auto < 0.5:
            recommandations.append("Augmenter la capacité de batteries")
            recommandations.append("Décaler certains usages vers les heures ensoleillées")

        if rendement and rendement < 0.8:
            recommandations.append("Nettoyer les panneaux")
            recommandations.append("Vérifier l'orientation et l'inclinaison")

        return {
            'performance': performance,
            'note': note,
            'recommandations': recommandations
        }

    def optimiser_autoconsommation(self, historique_conso, previsions_meteo):
        """
        Optimisations pour améliorer l'autoconsommation

        Args:
            historique_conso: Historique de consommation
            previsions_meteo: Prévisions météo

        Returns:
            Recommandations d'optimisation
        """
        optimisations = []

        # Analyse des patterns de consommation
        heures_creuses = self._identifier_heures_creuses(historique_conso)

        # Recommandations de programmation
        if heures_creuses:
            optimisations.append({
                'type': 'programmation',
                'description': f'Programmer les appareils sur les heures {heures_creuses}',
                'impact_estime': '+15% autoconsommation'
            })

        # Analyse météo pour prévision production
        if previsions_meteo:
            production_prevue = sum(self._calculer_production_theorique(jour)
                                  for jour in previsions_meteo[:3])  # 3 jours

            optimisations.append({
                'type': 'previsionnel',
                'description': f'Production prévue: {production_prevue:.1f} kWh sur 3 jours',
                'impact_estime': 'Ajustement consommation'
            })

        # Recommandations équipements
        optimisations.extend([
            {
                'type': 'equipement',
                'description': 'Installer un chauffe-eau thermodynamique solaire',
                'impact_estime': '+25% autoconsommation',
                'cout_estime': '3000-5000€'
            },
            {
                'type': 'equipement',
                'description': 'Ajouter des batteries de stockage',
                'impact_estime': '+40% autoconsommation',
                'cout_estime': '5000-10000€'
            }
        ])

        return optimisations

    def _identifier_heures_creuses(self, historique):
        """Identifie les heures de faible consommation"""
        if not historique:
            return None

        # Analyse horaire
        conso_par_heure = {}
        for mesure in historique:
            heure = mesure.timestamp.hour
            puissance = mesure.puissance_active_totale or 0
            if heure not in conso_par_heure:
                conso_par_heure[heure] = []
            conso_par_heure[heure].append(puissance)

        # Moyennes par heure
        moyennes = {h: sum(valeurs)/len(valeurs) for h, valeurs in conso_par_heure.items()}

        # Heures creuses = 20% les plus faibles
        heures_triees = sorted(moyennes.items(), key=lambda x: x[1])
        seuil_creux = heures_triees[int(len(heures_triees) * 0.2)][1]

        heures_creuses = [h for h, m in moyennes.items() if m <= seuil_creux]

        return f"{min(heures_creuses):02d}h-{max(heures_creuses):02d}h" if heures_creuses else None
```

### Dashboard solaire spécialisé

#### Interface utilisateur solaire

```html
<!-- templates/dashboard_solaire.html -->
<div class="solaire-dashboard">
    <!-- Métriques principales solaires -->
    <div class="solaire-metrics">
        <div class="metric-card production">
            <div class="metric-icon">☀️</div>
            <div class="metric-content">
                <div class="metric-value" id="production-aujourdhui">0.0 kWh</div>
                <div class="metric-label">Production aujourd'hui</div>
                <div class="metric-trend" id="trend-production">↗️ +5%</div>
            </div>
        </div>

        <div class="metric-card autoconsommation">
            <div class="metric-icon">🏠</div>
            <div class="metric-content">
                <div class="metric-value" id="autoconsommation">0%</div>
                <div class="metric-label">Taux autoconsommation</div>
                <div class="metric-subtext" id="autoconsommation-abs">0.0 kWh</div>
            </div>
        </div>

        <div class="metric-card economie">
            <div class="metric-icon">💰</div>
            <div class="metric-content">
                <div class="metric-value" id="economie-jour">0.00€</div>
                <div class="metric-label">Économie aujourd'hui</div>
                <div class="metric-subtext" id="economie-cumulee">0.00€ ce mois</div>
            </div>
        </div>
    </div>

    <!-- Graphique production/consommation -->
    <div class="solaire-chart">
        <div class="chart-header">
            <h3>Production vs Consommation</h3>
            <div class="chart-controls">
                <button class="period-btn active" data-period="jour">Aujourd'hui</button>
                <button class="period-btn" data-period="semaine">7 jours</button>
                <button class="period-btn" data-period="mois">30 jours</button>
            </div>
        </div>
        <div class="chart-container">
            <canvas id="solaire-chart"></canvas>
        </div>
    </div>

    <!-- Analyse de performance -->
    <div class="performance-analysis">
        <h3>📊 Performance Solaire</h3>
        <div class="performance-grid">
            <div class="performance-item">
                <div class="perf-label">Rendement système</div>
                <div class="perf-value" id="rendement-systeme">0%</div>
                <div class="perf-gauge">
                    <div class="gauge-fill" id="gauge-rendement"></div>
                </div>
            </div>

            <div class="performance-item">
                <div class="perf-label">Autoconsommation</div>
                <div class="perf-value" id="perf-autoconso">0%</div>
                <div class="perf-gauge">
                    <div class="gauge-fill" id="gauge-autoconso"></div>
                </div>
            </div>

            <div class="performance-item">
                <div class="perf-label">Injection réseau</div>
                <div class="perf-value" id="perf-injection">0 kWh</div>
                <div class="perf-trend" id="trend-injection">Stable</div>
            </div>
        </div>
    </div>

    <!-- Recommandations d'optimisation -->
    <div class="optimisations-panel">
        <h3>💡 Optimisations suggérées</h3>
        <div id="optimisations-list">
            <!-- Chargé dynamiquement -->
        </div>
    </div>
</div>
```

### API solaire spécialisée

```python
# app/routes/api_solaire.py
from flask import Blueprint, request, jsonify
from app.services.analyseur_solaire import AnalyseurSolaire
from app.models import MesureEnergie
from datetime import datetime, timedelta

solaire_bp = Blueprint('solaire', __name__)
analyseur_solaire = AnalyseurSolaire()

@solaire_bp.route('/api/solaire/bilan-energetique')
def get_bilan_energetique():
    """Bilan énergétique solaire du jour"""

    # Mesures du jour
    debut_jour = datetime.utcnow().replace(hour=0, minute=0, second=0, microsecond=0)
    mesures = MesureEnergie.query.filter(
        MesureEnergie.timestamp >= debut_jour
    ).order_by(MesureEnergie.timestamp).all()

    if not mesures:
        return jsonify({'error': 'Aucune donnée solaire disponible'}), 404

    # Simulation de données météo (en production, API météo)
    meteo_simulee = {
        'ensoleillement': 5.2,  # kWh/m² estimé pour la journée
        'temperature': 22
    }

    # Agrégation des données du jour
    energie_import = sum(m.energie_active_import or 0 for m in mesures)
    energie_export = sum(m.energie_active_export or 0 for m in mesures)

    donnees_jour = {
        'energie_active_import': energie_import,
        'energie_active_export': energie_export,
        'puissance_active_totale': mesures[-1].puissance_active_totale if mesures else 0
    }

    # Analyse solaire
    bilan = analyseur_solaire.analyser_bilan_energetique(donnees_jour, meteo_simulee)

    return jsonify({
        'date': debut_jour.date().isoformat(),
        'bilan': bilan,
        'meteo': meteo_simulee
    })

@solaire_bp.route('/api/solaire/optimisations')
def get_optimisations():
    """Recommandations d'optimisation solaire"""

    # Historique récent
    debut = datetime.utcnow() - timedelta(days=7)
    mesures = MesureEnergie.query.filter(
        MesureEnergie.timestamp >= debut
    ).all()

    # Prévisions météo simulées
    previsions_meteo = [
        {'ensoleillement': 6.0, 'temperature': 25},
        {'ensoleillement': 4.5, 'temperature': 20},
        {'ensoleillement': 7.2, 'temperature': 28}
    ]

    optimisations = analyseur_solaire.optimiser_autoconsommation(mesures, previsions_meteo)

    return jsonify({
        'optimisations': optimisations,
        'base_calcul': f"{len(mesures)} mesures sur 7 jours"
    })

@solaire_bp.route('/api/solaire/performance-historique')
def get_performance_historique():
    """Historique des performances solaires"""

    jours = int(request.args.get('days', 30))
    debut = datetime.utcnow() - timedelta(days=jours)

    # Agrégation par jour
    performances = []

    for i in range(jours):
        date_analyse = (debut + timedelta(days=i)).date()

        mesures_jour = MesureEnergie.query.filter(
            db.func.date(MesureEnergie.timestamp) == date_analyse
        ).all()

        if mesures_jour:
            energie_import = sum(m.energie_active_import or 0 for m in mesures_jour)
            energie_export = sum(m.energie_active_export or 0 for m in mesures_jour)

            donnees_jour = {
                'energie_active_import': energie_import,
                'energie_active_export': energie_export
            }

            bilan = analyseur_solaire.analyser_bilan_energetique(donnees_jour)

            performances.append({
                'date': date_analyse.isoformat(),
                'production': bilan['production_estimee'],
                'autoconsommation': bilan['autoconsommation'],
                'taux_autoconsommation': bilan['taux_autoconsommation'],
                'economie': bilan['economie_estimee']['total_economie']
            })

    return jsonify({
        'periode': f"{jours} jours",
        'performances': performances,
        'moyennes': {
            'production_journaliere': sum(p['production'] for p in performances) / len(performances) if performances else 0,
            'autoconsommation_moyenne': sum(p['taux_autoconsommation'] for p in performances) / len(performances) if performances else 0,
            'economie_mensuelle': sum(p['economie'] for p in performances) * 30 / jours if performances else 0
        }
    })

@solaire_bp.route('/api/solaire/prevision-production')
def prevoir_production():
    """Prévision de production solaire"""

    heures = int(request.args.get('hours', 24))

    # Simulation de prévision basée sur historique
    # En production: utiliser modèle météo + historique
    prevision = []

    base_time = datetime.utcnow()
    production_base = 3.5  # kW de puissance installée

    for h in range(heures):
        heure_prevue = base_time + timedelta(hours=h)

        # Simulation simplifiée (sinus pour variation journalière)
        heure_jour = heure_prevue.hour
        facteur_solaire = max(0, (heure_jour - 6) * (18 - heure_jour) / 36)  # Pic à midi

        production_prevue = production_base * facteur_solaire * 0.8  # 80% rendement

        prevision.append({
            'timestamp': heure_prevue.isoformat(),
            'production_prevue_kw': round(production_prevue, 2),
            'facteur_ensoleillement': round(facteur_solaire, 2)
        })

    return jsonify({
        'prevision': prevision,
        'total_prevu_kwh': round(sum(p['production_prevue_kw'] for p in prevision), 1),
        'methode': 'Simulation basée sur historique',
        'precision_estimee': '70%'
    })
```

### Intégration Home Assistant solaire

#### Configuration HA pour solaire

```yaml
# configuration.yaml
homeassistant:
  # Configuration de base

# Capteurs solaires
sensor:
  - platform: mqtt
    name: "Production Solaire"
    state_topic: "homeassistant/sensor/compteur_e450/production"
    value_template: "{{ value_json.production_kw }}"
    unit_of_measurement: "kW"
    device_class: power

  - platform: mqtt
    name: "Autoconsommation"
    state_topic: "homeassistant/sensor/compteur_e450/autoconso"
    value_template: "{{ value_json.taux_autoconsommation }}"
    unit_of_measurement: "%"
    icon: "mdi:solar-power"

  - platform: mqtt
    name: "Économie Solaire"
    state_topic: "homeassistant/sensor/compteur_e450/economie"
    value_template: "{{ value_json.economie_euro }}"
    unit_of_measurement: "€"
    icon: "mdi:cash"

# Automatisations solaires
automation:
  - alias: "Surplus solaire - Chauffe eau"
    trigger:
      platform: numeric_state
      entity_id: sensor.production_solaire
      above: 2.5
      for:
        minutes: 10
    condition:
      condition: time
      after: '09:00:00'
      before: '17:00:00'
    action:
      - service: switch.turn_on
        entity_id: switch.chauffe_eau

  - alias: "Notification économie solaire"
    trigger:
      platform: time
      at: "20:00:00"
    action:
      - service: notify.mobile_app
        data:
          message: >
            🌞 Économie solaire aujourd'hui:
            Production: {{ states('sensor.production_solaire') }} kWh
            Autoconsommation: {{ states('sensor.autoconsommation') }}%
            Économies: {{ states('sensor.economie_solaire') }}€
```

> **💡 À retenir** : Dans une maison solaire, le compteur E450 révèle l'efficacité réelle de l'installation et guide les optimisations pour maximiser l'autoconsommation.

> **⚠️ Astuce** : Utilisez les données du compteur combinées à des prévisions météo pour anticiper la production solaire et optimiser votre consommation domestique.

Dans le dernier chapitre de cette partie, nous explorerons l'immeuble collectif avec centralisation M-Bus !

---

**Navigation**
- [Chapitre précédent : Atelier industriel](Chapitre_25_Atelier_Industriel.md)
- [Chapitre suivant : Immeuble collectif](Chapitre_27_Immeuble_Collectif.md)
- [Retour à la table des matières](../../README.md)
