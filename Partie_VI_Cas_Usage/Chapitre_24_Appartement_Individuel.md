# 🏠 Chapitre 24 : Appartement individuel

## 📊 Suivi quotidien de la consommation

Dans un appartement individuel, le compteur E450 permet un suivi précis de la consommation énergétique pour optimiser les habitudes et réduire les coûts.

### Configuration typique

#### Matériel installé
- **Compteur E450** triphasé en tableau électrique
- **Adaptateur USB/IR** connecté au port optique
- **Raspberry Pi** ou ordinateur dédié dans un placard
- **Alimentation stabilisée** pour le bus M-Bus (si extension future)

#### Logiciel déployé
- **Application Flask** en service systemd
- **Base SQLite** pour l'historique local
- **Interface web** accessible depuis le réseau domestique
- **Notifications** par email pour les anomalies

### Cas d'usage quotidiens

#### 1. Suivi des appareils énergivores

```python
# Exemple de détection d'appareils
def detecter_appareil_energivore(mesure_actuelle, mesure_precedente, seuil_watt=500):
    """
    Détection d'un appareil énergivore basé sur la variation de puissance
    """
    if not mesure_precedente:
        return None

    variation = mesure_actuelle.puissance_active_totale - mesure_precedente.puissance_active_totale

    if variation > seuil_watt:
        return {
            'type': 'demarrage_appareil',
            'puissance_supplementaire': variation,
            'heure': mesure_actuelle.timestamp.strftime('%H:%M'),
            'appareil_estime': estimer_appareil(variation)
        }

    return None

def estimer_appareil(puissance_watt):
    """Estimation de l'appareil basé sur la puissance"""
    appareils_connus = {
        (2000, 2500): "Plaque de cuisson",
        (1500, 2000): "Lave-linge",
        (1000, 1500): "Sèche-linge",
        (800, 1200): "Four électrique",
        (600, 900): "Aspirateur",
        (400, 700): "Lave-vaisselle"
    }

    for (min_p, max_p), appareil in appareils_connus.items():
        if min_p <= puissance_watt <= max_p:
            return appareil

    return f"Appareil {puissance_watt}W (inconnu)"
```

#### 2. Alertes intelligentes

```python
# Configuration des alertes personnalisées
ALERTES_PERSONNALISEES = {
    'chauffe_eau': {
        'heure_debut': '06:00',
        'heure_fin': '09:00',
        'puissance_max': 2000,
        'message': "Chauffe-eau détecté en dehors des heures creuses"
    },
    'soiree': {
        'heure_debut': '18:00',
        'heure_fin': '23:00',
        'puissance_max': 3000,
        'message': "Consommation élevée en soirée"
    },
    'nuit': {
        'heure_debut': '23:00',
        'heure_fin': '06:00',
        'puissance_max': 100,
        'message': "Consommation nocturne détectée"
    }
}

def verifier_alertes_personnalisees(mesure):
    """Vérification des alertes personnalisées"""
    heure_actuelle = mesure.timestamp.strftime('%H:%M')

    for nom_alerte, config in ALERTES_PERSONNALISEES.items():
        if (config['heure_debut'] <= heure_actuelle <= config['heure_fin'] and
            mesure.puissance_active_totale > config['puissance_max']):

            return {
                'type': 'alerte_personnalisee',
                'alerte': nom_alerte,
                'message': config['message'],
                'puissance': mesure.puissance_active_totale,
                'heure': heure_actuelle
            }

    return None
```

#### 3. Budget énergétique

```python
class GestionnaireBudget:
    """Gestionnaire de budget énergétique"""

    def __init__(self, budget_mensuel_euros=80):
        self.budget_mensuel = budget_mensuel_euros
        self.tarifs = GestionnaireTarifs()

    def calculer_budget_restant(self, date=None):
        """Calcul du budget restant pour le mois"""
        if date is None:
            date = datetime.utcnow()

        debut_mois = date.replace(day=1, hour=0, minute=0, second=0, microsecond=0)

        # Consommation du mois en cours
        mesures = MesureEnergie.query.filter(
            MesureEnergie.timestamp >= debut_mois
        ).all()

        cout_total = 0
        for mesure in mesures:
            cout_heure = self.tarifs.calculer_cout_consommation(
                mesure.puissance_active_totale or 0, 1.0, mesure.timestamp
            )
            cout_total += cout_heure['cout_total']

        jours_ecoules = date.day
        jours_total = (date.replace(month=date.month+1, day=1) - date.replace(day=1)).days

        # Estimation du budget restant
        budget_utilise = cout_total
        budget_restant = self.budget_mensuel - budget_utilise

        # Projection fin de mois
        cout_journalier_moyen = budget_utilise / jours_ecoules if jours_ecoules > 0 else 0
        projection_fin_mois = cout_journalier_moyen * jours_total

        return {
            'budget_mensuel': self.budget_mensuel,
            'utilise_ce_mois': round(budget_utilise, 2),
            'restant_ce_mois': round(budget_restant, 2),
            'projection_fin_mois': round(projection_fin_mois, 2),
            'depense_supplementaire': round(projection_fin_mois - self.budget_mensuel, 2),
            'pourcentage_budget': round((budget_utilise / self.budget_mensuel) * 100, 1),
            'jours_restants': jours_total - jours_ecoules
        }

    def generer_alertes_budget(self):
        """Génération d'alertes budgétaires"""
        budget = self.calculer_budget_restant()

        alertes = []

        # Alerte 80% du budget
        if budget['pourcentage_budget'] > 80:
            alertes.append({
                'niveau': 'warning',
                'message': f"Budget à {budget['pourcentage_budget']:.1f}% - {budget['restant_ce_mois']:.2f}€ restants"
            })

        # Alerte dépassement prévu
        if budget['projection_fin_mois'] > self.budget_mensuel:
            depassement = budget['depense_supplementaire']
            alertes.append({
                'niveau': 'error',
                'message': f"Dépassement budgétaire prévu: +{depassement:.2f}€ fin de mois"
            })

        return alertes
```

### Interface utilisateur adaptée

#### Dashboard simplifié pour appartement

```html
<!-- templates/dashboard_appartement.html -->
<div class="appartement-dashboard">
    <div class="budget-section">
        <h3>💰 Budget Énergétique</h3>
        <div class="budget-gauge">
            <div class="gauge-container">
                <svg class="gauge" viewBox="0 0 120 120">
                    <!-- Cercle de fond -->
                    <circle cx="60" cy="60" r="50" fill="none" stroke="#e0e0e0" stroke-width="10"/>

                    <!-- Cercle de progression -->
                    <circle cx="60" cy="60" r="50" fill="none" stroke="#00d4ff" stroke-width="10"
                            stroke-dasharray="314" stroke-dashoffset="157"
                            transform="rotate(-90 60 60)"/>
                </svg>
                <div class="gauge-text">
                    <div class="percentage">65%</div>
                    <div class="budget-info">45€ / 70€</div>
                </div>
            </div>
        </div>
        <div class="budget-details">
            <div class="detail-item">
                <span>Restant ce mois:</span>
                <span class="highlight">25€</span>
            </div>
            <div class="detail-item">
                <span>Jours restants:</span>
                <span>12</span>
            </div>
        </div>
    </div>

    <div class="consommation-quotidienne">
        <h3>📊 Aujourd'hui</h3>
        <div class="daily-stats">
            <div class="stat-card">
                <div class="stat-icon">⚡</div>
                <div class="stat-value">12.5 kWh</div>
                <div class="stat-label">Consommé</div>
            </div>
            <div class="stat-card">
                <div class="stat-icon">💰</div>
                <div class="stat-value">3.45€</div>
                <div class="stat-label">Coût</div>
            </div>
            <div class="stat-card">
                <div class="stat-icon">🔌</div>
                <div class="stat-value">1450W</div>
                <div class="stat-label">Pic</div>
            </div>
        </div>
    </div>

    <div class="alertes-recommandations">
        <h3>🔔 Alertes & Conseils</h3>
        <div class="alertes-list">
            <div class="alerte-item warning">
                <div class="alerte-icon">⚠️</div>
                <div class="alerte-content">
                    <div class="alerte-title">Chauffe-eau à 14h</div>
                    <div class="alerte-desc">Déplacement vers HC recommandé</div>
                </div>
            </div>
            <div class="alerte-item info">
                <div class="alerte-icon">💡</div>
                <div class="alerte-content">
                    <div class="alerte-title">Économie possible</div>
                    <div class="alerte-desc">2.3€ par semaine en optimisant</div>
                </div>
            </div>
        </div>
    </div>
</div>
```

### Automatisations Home Assistant

#### Intégration complète

```yaml
# configuration.yaml pour appartement
homeassistant:
  # Configuration de base
  name: Appartement E450
  latitude: 48.8566
  longitude: 2.3522

# Intégration MQTT
mqtt:
  broker: localhost
  discovery: true

# Capteurs énergie
sensor:
  - platform: mqtt
    name: "Consommation E450"
    state_topic: "homeassistant/sensor/compteur_e450/state"
    value_template: "{{ value_json.puissance_active_w }}"
    unit_of_measurement: "W"
    device_class: power

  - platform: mqtt
    name: "Coût Horaire"
    state_topic: "homeassistant/sensor/compteur_e450/state"
    value_template: "{{ value_json.cout_horaire }}"
    unit_of_measurement: "€"
    icon: "mdi:currency-eur"

# Automatisations
automation:
  - alias: "Alerte Budget Élevé"
    trigger:
      platform: numeric_state
      entity_id: sensor.consommation_e450
      above: 3000
    action:
      - service: notify.mobile_app
        data:
          message: "Consommation élevée détectée: {{ states('sensor.consommation_e450') }}W"

  - alias: "Rapport Quotidien"
    trigger:
      platform: time
      at: "20:00:00"
    action:
      - service: notify.mobile_app
        data:
          message: >
            📊 Rapport quotidien:
            Consommation: {{ states('sensor.energie_quotidienne') }} kWh
            Coût: {{ states('sensor.cout_journalier') }} €
            Budget restant: {{ states('sensor.budget_restant') }} €

# Tableaux de bord
lovelace:
  mode: yaml
  dashboards:
    energie:
      mode: yaml
      filename: dashboards/energie.yaml
```

### Optimisations spécifiques appartement

#### Programmation des appareils

```python
class OptimisateurConsommation:
    """Optimiseur de consommation pour appartement"""

    def __init__(self):
        self.tarifs = GestionnaireTarifs()
        self.appareils = {
            'chauffe_eau': {'puissance': 2000, 'duree_quotidienne': 2},
            'lave_linge': {'puissance': 2000, 'duree_cycle': 2.5},
            'lave_vaisselle': {'puissance': 1800, 'duree_cycle': 3},
            'seche_linge': {'puissance': 2500, 'duree_cycle': 2}
        }

    def calculer_economies_potentieles(self):
        """Calcul des économies réalisables"""
        economies = {}

        # Chauffe-eau
        cout_hc = self.tarifs.calculer_cout_consommation(2000, 2)['cout_total']
        cout_hp = self.tarifs.calculer_cout_consommation(2000, 2, datetime(2025, 1, 15, 14, 0))['cout_total']
        economies['chauffe_eau'] = {
            'economies_quotidiennes': cout_hp - cout_hc,
            'economies_mensuelles': (cout_hp - cout_hc) * 30
        }

        # Lave-linge (3 cycles/semaine)
        cout_cycle = self.tarifs.calculer_cout_consommation(2000, 2.5)['cout_total']
        economies['lave_linge'] = {
            'cout_par_cycle': cout_cycle,
            'economies_mensuelles': cout_cycle * 1.5  # 3 cycles/semaine = 12/mois
        }

        return economies

    def suggerer_programmation(self):
        """Suggestions de programmation optimale"""
        return {
            'chauffe_eau': 'Programmer entre 22h-6h (HC)',
            'lave_linge': 'Lancer en HC (22h-6h)',
            'lave_vaisselle': 'Cycle nocturne en HC',
            'seche_linge': 'Après 18h pour bénéficier des dernières HC'
        }

    def estimer_impact_mensuel(self, changements):
        """Estimation de l'impact des changements"""
        cout_actuel = 80  # Budget mensuel actuel

        # Simulation avec changements
        cout_nouveau = cout_actuel * 0.85  # -15% estimation

        return {
            'cout_actuel': cout_actuel,
            'cout_nouveau': cout_nouveau,
            'economies_mensuelles': cout_actuel - cout_nouveau,
            'economies_annuelles': (cout_actuel - cout_nouveau) * 12
        }
```

> **💡 À retenir** : Dans un appartement, le compteur E450 permet un suivi personnalisé pour optimiser les habitudes quotidiennes et respecter le budget énergétique.

> **⚠️ Astuce** : Commencez par identifier vos appareils principaux et leurs horaires d'utilisation pour créer des alertes personnalisées efficaces.

Dans le prochain chapitre, nous explorerons l'atelier industriel avec monitoring triphasé avancé !

---

**Navigation**
- [Chapitre précédent : Prévision énergétique avec Prophet](../Partie_V_Analyse_Visualisation/Chapitre_23_Preuve_Energetique_Prophet.md)
- [Chapitre suivant : Atelier industriel](Chapitre_25_Atelier_Industriel.md)
- [Retour à la table des matières](../../README.md)
