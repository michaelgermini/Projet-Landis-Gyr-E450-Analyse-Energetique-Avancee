# 🏭 Chapitre 25 : Atelier industriel

## ⚡ Monitoring triphasé avancé

Dans un environnement industriel, le compteur E450 permet un monitoring énergétique précis avec analyse triphasée, détection d'anomalies et optimisation des processus de production.

### Architecture de monitoring industriel

#### Configuration matérielle

```
Atelier de production ──┬──► Compteur E450 (triphasé)
                       │
                       ├──► Automate industriel (PLC)
                       │
                       ├──► Serveur SCADA
                       │
                       └──► Système de supervision
```

#### Interfaces utilisées

- **Port optique** : Lecture périodique des données
- **Bus M-Bus** : Intégration dans le réseau d'automatisation
- **API REST** : Exposition des données pour SCADA
- **MQTT** : Publication temps réel vers système de supervision

### Analyse triphasée spécialisée

#### Équilibrage des phases

```python
class AnalyseurTriphase:
    """Analyseur spécialisé pour équilibre triphasé"""

    def __init__(self):
        self.seuil_desiquilibre = 0.1  # 10% de déséquilibre max

    def analyser_equilibre_phases(self, mesures):
        """
        Analyse l'équilibre des phases

        Args:
            mesures: Dict avec tensions et courants des 3 phases

        Returns:
            Rapport d'équilibre
        """
        # Tensions par phase
        tensions = [
            mesures.get('tension_l1', 0),
            mesures.get('tension_l2', 0),
            mesures.get('tension_l3', 0)
        ]

        # Courants par phase
        courants = [
            mesures.get('courant_l1', 0),
            mesures.get('courant_l2', 0),
            mesures.get('courant_l3', 0)
        ]

        # Calculs d'équilibre
        tension_moyenne = sum(tensions) / len(tensions)
        courant_moyen = sum(courants) / len(courants)

        # Écarts par rapport à la moyenne
        ecarts_tension = [(t - tension_moyenne) / tension_moyenne for t in tensions]
        ecarts_courant = [(c - courant_moyen) / courant_moyen for c in courants]

        # Score d'équilibre (0 = parfait, 1 = très déséquilibré)
        score_equilibre = max(abs(max(ecarts_tension)), abs(max(ecarts_courant)))

        return {
            'equilibre_global': score_equilibre < self.seuil_desiquilibre,
            'score_equilibre': score_equilibre,
            'phase_plus_chargee_tension': tensions.index(max(tensions)) + 1,
            'phase_plus_chargee_courant': courants.index(max(courants)) + 1,
            'recommandations': self._generer_recommandations_equilibre(score_equilibre)
        }

    def _generer_recommandations_equilibre(self, score):
        """Génère des recommandations d'équilibrage"""
        if score < 0.05:
            return "Équilibre excellent - Pas d'action nécessaire"
        elif score < 0.1:
            return "Équilibre acceptable - Surveillance recommandée"
        elif score < 0.2:
            return "Déséquilibre modéré - Redistribution de charge conseillée"
        else:
            return "Déséquilibre important - Rééquilibrage urgent nécessaire"
```

#### Analyse des puissances réactives

```python
class GestionnairePuissanceReactive:
    """Gestionnaire de puissance réactive industrielle"""

    def __init__(self):
        self.cible_facteur_puissance = 0.95  # Objectif 0.95
        self.penalites_fp = {
            0.90: 0,      # Pas de pénalité > 0.90
            0.85: 0.02,   # 2% de pénalité entre 0.85-0.90
            0.80: 0.05,   # 5% de pénalité < 0.80
        }

    def analyser_facteur_puissance(self, mesures):
        """
        Analyse complète du facteur de puissance

        Args:
            mesures: Données de puissance active/réactive

        Returns:
            Analyse du facteur de puissance
        """
        p_active = mesures.get('puissance_active_totale', 0)
        p_reactive = mesures.get('puissance_reactive_totale', 0)

        # Calcul du facteur de puissance
        s_apparente = (p_active**2 + p_reactive**2)**0.5
        fp = p_active / s_apparente if s_apparente != 0 else 1.0

        # Évaluation
        if fp >= self.cible_facteur_puissance:
            evaluation = "excellent"
            penalite = 0
        elif fp >= 0.90:
            evaluation = "bon"
            penalite = 0
        elif fp >= 0.85:
            evaluation = "acceptable"
            penalite = self.penalites_fp[0.85]
        elif fp >= 0.80:
            evaluation = "médiocre"
            penalite = self.penalites_fp[0.80]
        else:
            evaluation = "critique"
            penalite = self.penalites_fp[0.80]  # Pénalité maximale

        # Calcul des économies potentielles avec compensation
        if fp < self.cible_facteur_puissance:
            fp_ameliore = min(fp + 0.1, 1.0)  # Amélioration de 0.1
            p_active_amelioree = s_apparente * fp_ameliore
            economies_potentiel = p_active - p_active_amelioree
        else:
            economies_potentiel = 0

        return {
            'facteur_puissance': fp,
            'evaluation': evaluation,
            'penalite_applicable': penalite,
            'economies_compensation': economies_potentiel,
            'compensation_recommandee': fp < 0.90,
            'capacite_batteries_necesaire': self._calculer_capacite_batteries(p_reactive)
        }

    def _calculer_capacite_batteries(self, p_reactive):
        """Calcule la capacité de batteries de compensation nécessaire"""
        # Règle empirique: 1 kVAr ≈ 100-150 kVAr de batteries
        capacite_min = abs(p_reactive) * 100 / 1000  # kVAr
        capacite_max = abs(p_reactive) * 150 / 1000  # kVAr

        return {
            'capacite_min_kvar': capacite_min,
            'capacite_max_kvar': capacite_max,
            'capacite_recommandee_kvar': (capacite_min + capacite_max) / 2
        }
```

### Intégration SCADA industrielle

#### API spécialisée industrie

```python
# app/routes/api_industrie.py
from flask import Blueprint, request, jsonify
from app.services.analyse_triphase import AnalyseurTriphase
from app.services.gestion_puissance_reactive import GestionnairePuissanceReactive
from app.models import MesureEnergie

industrie_bp = Blueprint('industrie', __name__)

@industrie_bp.route('/api/industrie/equilibre-phases')
def get_equilibre_phases():
    """État d'équilibre des phases"""

    # Récupération dernière mesure
    mesure = MesureEnergie.query.order_by(MesureEnergie.timestamp.desc()).first()

    if not mesure:
        return jsonify({'error': 'Aucune mesure disponible'}), 404

    # Analyse d'équilibre
    analyseur = AnalyseurTriphase()
    mesures_data = {
        'tension_l1': mesure.tension_l1,
        'tension_l2': mesure.tension_l2,
        'tension_l3': mesure.tension_l3,
        'courant_l1': mesure.courant_l1,
        'courant_l2': mesure.courant_l2,
        'courant_l3': mesure.courant_l3,
    }

    equilibre = analyseur.analyser_equilibre_phases(mesures_data)

    return jsonify({
        'timestamp': mesure.timestamp.isoformat(),
        'equilibre': equilibre
    })

@industrie_bp.route('/api/industrie/facteur-puissance')
def get_facteur_puissance():
    """Analyse du facteur de puissance"""

    mesure = MesureEnergie.query.order_by(MesureEnergie.timestamp.desc()).first()

    if not mesure:
        return jsonify({'error': 'Aucune mesure disponible'}), 404

    gestionnaire = GestionnairePuissanceReactive()
    mesures_data = {
        'puissance_active_totale': mesure.puissance_active_totale,
        'puissance_reactive_totale': mesure.puissance_reactive_totale,
    }

    analyse_fp = gestionnaire.analyser_facteur_puissance(mesures_data)

    return jsonify({
        'timestamp': mesure.timestamp.isoformat(),
        'facteur_puissance': analyse_fp
    })

@industrie_bp.route('/api/industrie/charges-par-machine')
def get_charges_machine():
    """Répartition des charges par machine/équipement"""

    # Simulation de répartition (en production, utiliser des capteurs individuels)
    charges_estimees = {
        'ligne_production_1': 15.5,  # kW
        'ligne_production_2': 12.8,
        'ventilation': 8.2,
        'refroidissement': 6.1,
        'eclairage': 3.4,
        'divers': 4.0
    }

    total = sum(charges_estimees.values())

    return jsonify({
        'timestamp': datetime.utcnow().isoformat(),
        'charges_par_equipement': charges_estimees,
        'total_consomme': total,
        'pourcentages': {k: (v/total)*100 for k, v in charges_estimees.items()}
    })

@industrie_bp.route('/api/industrie/kpi-energetiques')
def get_kpi_energetiques():
    """KPI énergétiques pour tableau de bord industriel"""

    # Période d'analyse (dernière heure)
    debut = datetime.utcnow() - timedelta(hours=1)

    mesures = MesureEnergie.query.filter(
        MesureEnergie.timestamp >= debut
    ).order_by(MesureEnergie.timestamp).all()

    if not mesures:
        return jsonify({'error': 'Données insuffisantes'}), 404

    # Calculs des KPI
    puissances = [m.puissance_active_totale for m in mesures if m.puissance_active_totale]

    kpis = {
        'puissance_moyenne_kw': sum(puissances) / len(puissances) / 1000,
        'puissance_max_kw': max(puissances) / 1000,
        'consommation_horaire_kwh': sum(puissances) / len(puissances) / 1000,
        'efficacite_globale': self._calculer_efficacite_industrielle(mesures),
        'heures_analyse': 1,
        'qualite_donnees': len([m for m in mesures if m.qualite == 'good']) / len(mesures) * 100
    }

    return jsonify({
        'periode': f"{debut.strftime('%H:%M')} - {datetime.utcnow().strftime('%H:%M')}",
        'kpis': kpis
    })

def _calculer_efficacite_industrielle(self, mesures):
    """Calcule un indice d'efficacité énergétique industrielle"""
    if not mesures:
        return 0

    # Facteur simplifié: rapport puissance utile / puissance totale
    # En réalité, utiliser des données de production
    facteurs_puissance = [m.facteur_puissance or 0.9 for m in mesures]
    fp_moyen = sum(facteurs_puissance) / len(facteurs_puissance)

    # Pénalité pour les harmoniques
    thd_moyen = sum([m.thd_tension or 3.0 for m in mesures]) / len(mesures)
    penalite_harmoniques = min(thd_moyen / 5, 0.2)  # Max 20% de pénalité

    efficacite = fp_moyen * (1 - penalite_harmoniques)

    return round(efficacite * 100, 1)  # Pourcentage
```

### Alertes et automatisations industrielles

#### Système d'alertes critiques

```python
class SystemeAlertesIndustrielles:
    """Système d'alertes spécialisé pour environnement industriel"""

    def __init__(self):
        self.seuils_critiques = {
            'puissance_max': 50,      # kW - Sécurité installations
            'desequilibre_phases': 0.15,  # 15% - Équilibre réseau
            'facteur_puissance_min': 0.85,  # Qualité réseau
            'tension_min': 380,      # V - Alimentation triphasée
            'tension_max': 420,      # V
            'thd_max': 8.0,          # % - Qualité d'onde
        }

    def evaluer_risques_operationnels(self, mesures):
        """
        Évaluation des risques opérationnels

        Args:
            mesures: Données du compteur

        Returns:
            Niveau de risque et recommandations
        """
        risques = []

        # Risque surcharge
        if mesures.get('puissance_active_totale', 0) > self.seuils_critiques['puissance_max'] * 1000:
            risques.append({
                'type': 'surcharge',
                'severite': 'critique',
                'description': 'Risque de déclenchement protection',
                'action': 'Réduire immédiatement la charge'
            })

        # Risque qualité réseau
        fp = mesures.get('facteur_puissance', 1.0)
        if fp < self.seuils_critiques['facteur_puissance_min']:
            risques.append({
                'type': 'qualite_reseau',
                'severite': 'majeur',
                'description': f'Facteur de puissance faible: {fp:.3f}',
                'action': 'Vérifier compensation réactive'
            })

        # Risque tension
        tensions = [mesures.get(f'tension_l{i}', 400) for i in range(1, 4)]
        tension_min = min(tensions)
        tension_max = max(tensions)

        if tension_min < self.seuils_critiques['tension_min']:
            risques.append({
                'type': 'tension_basse',
                'severite': 'majeur',
                'description': f'Tension minimale: {tension_min}V',
                'action': 'Vérifier alimentation secteur'
            })

        # Classification globale du risque
        if any(r['severite'] == 'critique' for r in risques):
            niveau_global = 'critique'
        elif any(r['severite'] == 'majeur' for r in risques):
            niveau_global = 'majeur'
        elif risques:
            niveau_global = 'mineur'
        else:
            niveau_global = 'normal'

        return {
            'niveau_risque': niveau_global,
            'risques_detectes': risques,
            'recommandations': self._generer_recommandations_securite(risques)
        }

    def _generer_recommandations_securite(self, risques):
        """Génère des recommandations de sécurité"""
        recommandations = []

        types_risques = [r['type'] for r in risques]

        if 'surcharge' in types_risques:
            recommandations.extend([
                "Arrêter les équipements non essentiels",
                "Vérifier la répartition des charges",
                "Contacter le service maintenance électrique"
            ])

        if 'qualite_reseau' in types_risques:
            recommandations.extend([
                "Vérifier les batteries de compensation",
                "Programmer une maintenance préventive",
                "Surveiller la consommation réactive"
            ])

        if 'tension_basse' in types_risques:
            recommandations.extend([
                "Vérifier les connexions électriques",
                "Contrôler les transformateurs",
                "Contacter le fournisseur d'énergie"
            ])

        return recommandations
```

### Dashboard industriel Grafana

#### Panels spécialisés industrie

```json
// Panel: Équilibre triphasé
{
  "title": "Équilibre des Phases",
  "type": "gauge",
  "targets": [{
    "query": "SELECT last(\"score_equilibre\") FROM \"analyse_triphase\"",
    "rawQuery": true
  }],
  "fieldConfig": {
    "defaults": {
      "unit": "percent",
      "min": 0,
      "max": 20,
      "thresholds": {
        "steps": [
          { "color": "green", "value": 0 },
          { "color": "orange", "value": 5 },
          { "color": "red", "value": 10 }
        ]
      }
    }
  }
}

// Panel: Puissance réactive
{
  "title": "Puissance Réactive & Facteur de Puissance",
  "type": "stat",
  "targets": [{
    "query": "SELECT last(\"facteur_puissance\") FROM \"analyse_fp\" WHERE time >= now() - 1h",
    "rawQuery": true
  }],
  "fieldConfig": {
    "defaults": {
      "unit": "none",
      "min": 0.8,
      "max": 1.0,
      "thresholds": {
        "steps": [
          { "color": "red", "value": 0.8 },
          { "color": "orange", "value": 0.9 },
          { "color": "green", "value": 0.95 }
        ]
      }
    }
  }
}
```

> **💡 À retenir** : Dans un environnement industriel, le E450 permet non seulement le suivi énergétique mais aussi l'optimisation des processus de production et la prévention des pannes.

> **⚠️ Astuce** : Intégrez le compteur dans votre système SCADA existant pour une supervision énergétique complète de vos installations industrielles.

Dans le prochain chapitre, nous explorerons la maison solaire avec analyse production/consommation !

---

**Navigation**
- [Chapitre précédent : Appartement individuel](Chapitre_24_Appartement_Individuel.md)
- [Chapitre suivant : Maison solaire](Chapitre_26_Maison_Solaire.md)
- [Retour à la table des matières](../../README.md)
