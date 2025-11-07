# 🔌 Chapitre 17 : Calcul du coût énergétique

## 💰 Intégration du tarif fournisseur

Le calcul précis du coût énergétique nécessite la connaissance des tarifs horaires et des différentes tranches de consommation.

### Structure des tarifs EDF

#### Tarifs réglementés (2025)

```python
# app/services/tarifs_energie.py
from datetime import datetime, time
from typing import Dict, List, Optional
import json

class GestionnaireTarifs:
    """Gestionnaire des tarifs énergétiques"""

    def __init__(self):
        # Tarifs EDF réglementés (exemple 2025)
        self.tarifs_base = {
            'hc_bleu': 0.1332,    # €/kWh Heures creuses
            'hp_bleu': 0.1765,    # €/kWh Heures pleines
            'hc_blanc': 0.1543,   # €/kWh Heures creuses
            'hp_blanc': 0.2001,   # €/kWh Heures pleines
            'hc_rouge': 0.1652,   # €/kWh Heures creuses
            'hp_rouge': 0.2349,   # €/kWh Heures pleines
        }

        # Abonnement mensuel
        self.abonnements = {
            'bleu': 13.48,
            'blanc': 16.34,
            'rouge': 20.15,
        }

        # Taxes et contributions
        self.taxes = {
            'tva': 0.055,        # 5.5% sur abonnement
            'cta': 0.00186,      # Contribution Tarifaire d'Acheminement
            'cspe': 0.02250,     # Contribution au Service Public de l'Électricité
            'tdcfe': 0.00099,    # Taxe Départementale sur la Conso Finale d'Électricité
            'tccfe': 0.00061,    # Taxe Communale sur la Conso Finale d'Électricité
        }

    def obtenir_tarif_actuel(self, dt: datetime = None) -> Dict[str, float]:
        """
        Retourne le tarif applicable à un moment donné

        Args:
            dt: Date/heure (défaut: maintenant)

        Returns:
            Dict avec tarifs HC/HP et abonnement
        """
        if dt is None:
            dt = datetime.now()

        # Détermination du type de jour
        jour_semaine = dt.weekday()  # 0=lundi, 6=dimanche

        # Heures creuses selon le jour
        if jour_semaine < 5:  # Lundi à vendredi
            hc_start = time(22, 0)  # 22h
            hc_end = time(6, 0)     # 6h
        else:  # Samedi et dimanche
            hc_start = time(12, 0)  # Toute la journée
            hc_end = time(12, 0)    # (chevauchement = toute la journée)

        heure_actuelle = dt.time()
        is_heures_creuses = self._est_dans_plage(heure_actuelle, hc_start, hc_end)

        # Option tarifaire (par défaut Tempo bleu)
        option = 'bleu'

        return {
            'tarif_hc': self.tarifs_base[f'hc_{option}'],
            'tarif_hp': self.tarifs_base[f'hp_{option}'],
            'abonnement_mensuel': self.abonnements[option],
            'is_heures_creuses': is_heures_creuses,
            'option': option,
            'taxes': self.taxes.copy()
        }

    def _est_dans_plage(self, heure: time, debut: time, fin: time) -> bool:
        """Vérifie si une heure est dans une plage (gère chevauchement minuit)"""
        if debut <= fin:
            return debut <= heure <= fin
        else:
            # Chevauchement minuit (ex: 22h - 6h)
            return heure >= debut or heure <= fin

    def calculer_cout_consommation(self, puissance_w: float, duree_h: float,
                                  dt: datetime = None) -> Dict[str, float]:
        """
        Calcule le coût d'une consommation donnée

        Args:
            puissance_w: Puissance en watts
            duree_h: Durée en heures
            dt: Moment de la consommation

        Returns:
            Dict avec coût détaillé
        """
        tarifs = self.obtenir_tarif_actuel(dt)

        # Énergie consommée (kWh)
        energie_kwh = (puissance_w * duree_h) / 1000

        # Coût de l'énergie
        if tarifs['is_heures_creuses']:
            cout_energie = energie_kwh * tarifs['tarif_hc']
            tranche = 'hc'
        else:
            cout_energie = energie_kwh * tarifs['tarif_hp']
            tranche = 'hp'

        # Taxes sur l'énergie
        cta = energie_kwh * tarifs['taxes']['cta']
        cspe = energie_kwh * tarifs['taxes']['cspe']
        tdcfe = energie_kwh * tarifs['taxes']['tdcfe']
        tccfe = energie_kwh * tarifs['taxes']['tccfe']

        cout_taxes = cta + cspe + tdcfe + tccfe

        # TVA sur abonnement (calculée séparément)
        abonnement_ht = tarifs['abonnement_mensuel']
        tva_abonnement = abonnement_ht * tarifs['taxes']['tva']

        return {
            'energie_kwh': energie_kwh,
            'cout_energie': cout_energie,
            'cout_taxes': cout_taxes,
            'cout_total': cout_energie + cout_taxes,
            'tranche': tranche,
            'tarif_applique': tarifs['tarif_hc'] if tranche == 'hc' else tarifs['tarif_hp'],
            'abonnement_mensuel': abonnement_ht,
            'tva_abonnement': tva_abonnement,
            'detail_taxes': {
                'cta': cta,
                'cspe': cspe,
                'tdcfe': tdcfe,
                'tccfe': tccfe,
            }
        }

    def estimer_cout_mensuel(self, mesures: List[Dict]) -> Dict[str, float]:
        """
        Estime le coût mensuel basé sur l'historique

        Args:
            mesures: Liste de mesures avec puissance et timestamp

        Returns:
            Estimation du coût mensuel
        """
        if not mesures:
            return {'erreur': 'Aucune mesure disponible'}

        cout_total = 0
        energie_totale = 0
        heures_hc = 0
        heures_hp = 0

        # Tri par timestamp
        mesures.sort(key=lambda x: x['timestamp'])

        for i in range(1, len(mesures)):
            mesure_prec = mesures[i-1]
            mesure_actu = mesures[i]

            # Durée entre mesures (secondes)
            dt_prec = datetime.fromisoformat(mesure_prec['timestamp'].replace('Z', '+00:00'))
            dt_actu = datetime.fromisoformat(mesure_actu['timestamp'].replace('Z', '+00:00'))
            duree_s = (dt_actu - dt_prec).total_seconds()
            duree_h = duree_s / 3600

            # Puissance moyenne
            puissance_moy = (mesure_prec['puissance_active_totale'] + mesure_actu['puissance_active_totale']) / 2

            # Coût pour cette période
            cout_periode = self.calculer_cout_consommation(puissance_moy, duree_h, dt_prec)

            cout_total += cout_periode['cout_total']
            energie_totale += cout_periode['energie_kwh']

            if cout_periode['tranche'] == 'hc':
                heures_hc += duree_h
            else:
                heures_hp += duree_h

        # Extrapolation mensuelle (30 jours)
        jours_dans_donnees = (dt_actu - dt_prec).total_seconds() / 86400
        facteur_mensuel = 30 / jours_dans_donnees if jours_dans_donnees > 0 else 1

        cout_mensuel = cout_total * facteur_mensuel
        abonnement = self.abonnements['bleu']  # Par défaut
        tva_abonnement = abonnement * self.taxes['tva']

        return {
            'cout_energie_mensuel': cout_mensuel,
            'abonnement_mensuel': abonnement,
            'tva_abonnement': tva_abonnement,
            'cout_total_mensuel': cout_mensuel + abonnement + tva_abonnement,
            'energie_estimee_mensuelle': energie_totale * facteur_mensuel,
            'heures_hc_par_jour': heures_hc / jours_dans_donnees if jours_dans_donnees > 0 else 0,
            'heures_hp_par_jour': heures_hp / jours_dans_donnees if jours_dans_donnees > 0 else 0,
            'base_calcul': f"{jours_dans_donnees:.1f} jours de données"
        }
```

### API de calcul des coûts

```python
# app/routes/api_costs.py
from flask import Blueprint, request, jsonify
from app.services.tarifs_energie import GestionnaireTarifs
from app.models import MesureEnergie
from datetime import datetime, timedelta

costs_bp = Blueprint('costs', __name__)
tarifs_manager = GestionnaireTarifs()

@costs_bp.route('/api/costs/estimate')
def estimate_current_cost():
    """Estimation du coût actuel basé sur la dernière mesure"""

    # Dernière mesure
    mesure = MesureEnergie.query.order_by(MesureEnergie.timestamp.desc()).first()

    if not mesure:
        return jsonify({'error': 'Aucune mesure disponible'}), 404

    # Calcul du coût horaire
    cout_horaire = tarifs_manager.calculer_cout_consommation(
        mesure.puissance_active_totale or 0,
        1.0,  # 1 heure
        mesure.timestamp
    )

    return jsonify({
        'cout_horaire': cout_horaire['cout_total'],
        'cout_journalier_estime': cout_horaire['cout_total'] * 24,
        'tranche_actuelle': cout_horaire['tranche'],
        'tarif_applique': cout_horaire['tarif_applique'],
        'puissance_base': mesure.puissance_active_totale,
        'timestamp': mesure.timestamp.isoformat() + 'Z'
    })

@costs_bp.route('/api/costs/monthly')
def estimate_monthly_cost():
    """Estimation du coût mensuel"""

    # Paramètres
    jours = int(request.args.get('days', 7))  # Jours d'historique à utiliser

    # Récupération des mesures
    debut = datetime.utcnow() - timedelta(days=jours)
    mesures = MesureEnergie.query.filter(
        MesureEnergie.timestamp >= debut
    ).order_by(MesureEnergie.timestamp).all()

    if not mesures:
        return jsonify({'error': 'Aucune mesure disponible'}), 404

    # Conversion en format pour l'estimation
    mesures_data = [{
        'timestamp': m.timestamp.isoformat() + 'Z',
        'puissance_active_totale': m.puissance_active_totale or 0
    } for m in mesures]

    # Estimation mensuelle
    estimation = tarifs_manager.estimer_cout_mensuel(mesures_data)

    return jsonify(estimation)

@costs_bp.route('/api/costs/tariffs')
def get_current_tariffs():
    """Retourne les tarifs actuellement applicables"""

    tarifs = tarifs_manager.obtenir_tarif_actuel()

    return jsonify({
        'option_tarifaire': tarifs['option'],
        'is_heures_creuses': tarifs['is_heures_creuses'],
        'tarifs': {
            'hc': tarifs['tarif_hc'],
            'hp': tarifs['tarif_hp']
        },
        'abonnement_mensuel': tarifs['abonnement_mensuel'],
        'taxes': tarifs['taxes']
    })

@costs_bp.route('/api/costs/simulate', methods=['POST'])
def simulate_cost():
    """Simulation de coût pour une consommation donnée"""

    data = request.get_json()

    if not data or 'puissance_w' not in data:
        return jsonify({'error': 'puissance_w requise'}), 400

    puissance = data['puissance_w']
    duree_h = data.get('duree_h', 1.0)
    dt_str = data.get('datetime')

    # Parsing de la date si fournie
    dt = None
    if dt_str:
        try:
            dt = datetime.fromisoformat(dt_str.replace('Z', '+00:00'))
        except:
            return jsonify({'error': 'Format datetime invalide'}), 400

    # Calcul
    cout = tarifs_manager.calculer_cout_consommation(puissance, duree_h, dt)

    return jsonify(cout)
```

## ⚡ Calcul HP/HC, coûts mensuels, moyennes glissantes

### Service de calculs énergétiques avancés

```python
# app/services/calculateur_energie.py
from app.models import MesureEnergie
from app.services.tarifs_energie import GestionnaireTarifs
from datetime import datetime, timedelta
from typing import List, Dict, Tuple
import statistics

class CalculateurEnergie:
    """Calculateur avancé pour analyses énergétiques"""

    def __init__(self):
        self.tarifs = GestionnaireTarifs()

    def analyser_consommation_horaires(self, mesures: List[MesureEnergie]) -> Dict:
        """
        Analyse détaillée HC/HP sur une période
        """
        if not mesures:
            return {'erreur': 'Aucune mesure'}

        # Grouper par heure
        conso_par_heure = {}
        couts_hc = []
        couts_hp = []

        for mesure in mesures:
            heure = mesure.timestamp.replace(minute=0, second=0, microsecond=0)

            if heure not in conso_par_heure:
                conso_par_heure[heure] = []

            conso_par_heure[heure].append(mesure.puissance_active_totale or 0)

        # Calculs par heure
        resultats = []
        total_hc = 0
        total_hp = 0

        for heure, puissances in conso_par_heure.items():
            puissance_moy = statistics.mean(puissances)

            # Déterminer HC/HP
            tarifs_heure = self.tarifs.obtenir_tarif_actuel(heure)
            tranche = 'hc' if tarifs_heure['is_heures_creuses'] else 'hp'

            # Calcul du coût horaire
            cout_heure = self.tarifs.calculer_cout_consommation(puissance_moy, 1.0, heure)

            resultats.append({
                'heure': heure.isoformat(),
                'puissance_moyenne': puissance_moy,
                'tranche': tranche,
                'cout_heure': cout_heure['cout_total']
            })

            if tranche == 'hc':
                total_hc += cout_heure['cout_total']
            else:
                total_hp += cout_heure['cout_total']

        return {
            'analyse_horaire': resultats,
            'total_hc': total_hc,
            'total_hp': total_hp,
            'ratio_hp_hc': total_hp / total_hc if total_hc > 0 else 0,
            'pourcentage_hc': (total_hc / (total_hc + total_hp)) * 100 if (total_hc + total_hp) > 0 else 0
        }

    def calculer_moyennes_glissantes(self, mesures: List[MesureEnergie],
                                   fenetre_minutes: int = 15) -> List[Dict]:
        """
        Calcule les moyennes glissantes pour lisser les données
        """
        if not mesures or len(mesures) < 2:
            return []

        # Tri par timestamp
        mesures.sort(key=lambda x: x.timestamp)

        resultats = []
        fenetre_timedelta = timedelta(minutes=fenetre_minutes)

        for i, mesure_centrale in enumerate(mesures):
            # Fenêtre autour de la mesure centrale
            debut_fenetre = mesure_centrale.timestamp - fenetre_timedelta
            fin_fenetre = mesure_centrale.timestamp + fenetre_timedelta

            # Mesures dans la fenêtre
            mesures_fenetre = [
                m for m in mesures
                if debut_fenetre <= m.timestamp <= fin_fenetre
            ]

            if len(mesures_fenetre) >= 3:  # Minimum pour une moyenne significative
                puissances = [m.puissance_active_totale or 0 for m in mesures_fenetre]
                moyenne_glissante = statistics.mean(puissances)
                ecart_type = statistics.stdev(puissances) if len(puissances) > 1 else 0

                resultats.append({
                    'timestamp': mesure_centrale.timestamp.isoformat(),
                    'puissance_originale': mesure_centrale.puissance_active_totale,
                    'moyenne_glissante': moyenne_glissante,
                    'ecart_type': ecart_type,
                    'nb_mesures_fenetre': len(mesures_fenetre)
                })

        return resultats

    def detecter_pics_consommation(self, mesures: List[MesureEnergie],
                                 seuil_multiplicateur: float = 2.0) -> List[Dict]:
        """
        Détecte les pics de consommation
        """
        if not mesures:
            return []

        # Calculer la moyenne et écart-type
        puissances = [m.puissance_active_totale or 0 for m in mesures]
        moyenne = statistics.mean(puissances)
        ecart_type = statistics.stdev(puissances) if len(puissances) > 1 else 0

        seuil_pic = moyenne + (ecart_type * seuil_multiplicateur)

        pics = []
        for mesure in mesures:
            puissance = mesure.puissance_active_totale or 0
            if puissance > seuil_pic:
                pics.append({
                    'timestamp': mesure.timestamp.isoformat(),
                    'puissance': puissance,
                    'moyenne': moyenne,
                    'ecart_type': ecart_type,
                    'seuil_pic': seuil_pic,
                    'depacement': puissance - seuil_pic
                })

        return sorted(pics, key=lambda x: x['puissance'], reverse=True)

    def analyser_efficacite(self, mesures: List[MesureEnergie]) -> Dict:
        """
        Analyse l'efficacité énergétique (facteur de puissance, etc.)
        """
        if not mesures:
            return {'erreur': 'Aucune mesure'}

        # Statistiques de base
        fact_puiss = [m.facteur_puissance or 1.0 for m in mesures]
        thd_tensions = [m.thd_tension or 0 for m in mesures]
        thd_courants = [m.thd_courant or 0 for m in mesures]

        return {
            'facteur_puissance_moyen': statistics.mean(fact_puiss),
            'facteur_puissance_min': min(fact_puiss),
            'facteur_puissance_max': max(fact_puiss),
            'thd_tension_moyen': statistics.mean(thd_tensions),
            'thd_courant_moyen': statistics.mean(thd_courants),
            'efficacite_globale': self._calculer_score_efficacite(fact_puiss, thd_tensions, thd_courants)
        }

    def _calculer_score_efficacite(self, fact_puiss: List[float],
                                  thd_tensions: List[float],
                                  thd_courants: List[float]) -> float:
        """
        Calcule un score d'efficacité globale (0-100)
        """
        if not fact_puiss:
            return 0

        # Score basé sur facteur de puissance (pondération 50%)
        fp_moyen = statistics.mean(fact_puiss)
        score_fp = min(100, fp_moyen * 100)  # 1.0 = 100 points

        # Score basé sur THD (pondération 30% tension, 20% courant)
        thd_t_moyen = statistics.mean(thd_tensions)
        thd_c_moyen = statistics.mean(thd_courants)

        # THD idéal < 5%, pénalité progressive
        score_thd_t = max(0, 100 - (thd_t_moyen * 5))
        score_thd_c = max(0, 100 - (thd_c_moyen * 5))

        # Score global pondéré
        score_global = (score_fp * 0.5) + (score_thd_t * 0.3) + (score_thd_c * 0.2)

        return round(score_global, 1)

    def prevoir_consommation_jour_suivant(self, mesures: List[MesureEnergie]) -> Dict:
        """
        Prévision simple de la consommation du lendemain
        """
        if len(mesures) < 24:  # Minimum 24h de données
            return {'erreur': 'Données insuffisantes pour prévision'}

        # Grouper par heure de la journée (0-23)
        conso_par_heure = {h: [] for h in range(24)}

        for mesure in mesures:
            heure = mesure.timestamp.hour
            conso_par_heure[heure].append(mesure.puissance_active_totale or 0)

        # Calculer les moyennes par heure
        previsions = {}
        for heure, valeurs in conso_par_heure.items():
            if valeurs:
                previsions[heure] = statistics.mean(valeurs)

        # Estimation consommation totale journalière
        conso_totale_estimee = sum(previsions.values())  # Approximation en Wh

        return {
            'previsions_horaires': previsions,
            'conso_totale_estimee_wh': conso_totale_estimee,
            'base_calcul': f"{len(mesures)} mesures sur {len([h for h, v in conso_par_heure.items() if v])}h"
        }
```

### Tests des calculs

```python
# tests/test_calculs_energie.py
import pytest
from datetime import datetime, timedelta
from app.services.calculateur_energie import CalculateurEnergie
from app.services.tarifs_energie import GestionnaireTarifs

def test_calcul_cout_basique():
    """Test du calcul de coût basique"""
    calc = CalculateurEnergie()

    # Simulation mesure
    from app.models import MesureEnergie
    mesure = MesureEnergie(
        puissance_active_totale=2000.0,
        timestamp=datetime(2025, 1, 15, 14, 30)  # Mardi après-midi = HP
    )

    # Calcul coût pour 1 heure
    cout = calc.tarifs.calculer_cout_consommation(2000.0, 1.0, mesure.timestamp)

    assert cout['energie_kwh'] == 2.0  # 2000W * 1h = 2kWh
    assert cout['tranche'] == 'hp'     # Après-midi = HP
    assert cout['cout_energie'] > 0    # Coût positif
    assert 'tarif_applique' in cout

def test_detection_pics():
    """Test de la détection de pics"""
    calc = CalculateurEnergie()

    # Création de mesures factices
    mesures = []
    base_time = datetime.utcnow()

    for i in range(100):
        puissance = 1000 + (i % 10) * 50  # Variations normales

        # Ajouter quelques pics
        if i in [25, 75]:
            puissance = 3000  # Pic

        mesure = type('Mesure', (), {
            'puissance_active_totale': puissance,
            'timestamp': base_time + timedelta(minutes=i)
        })()
        mesures.append(mesure)

    pics = calc.detecter_pics_consommation(mesures, seuil_multiplicateur=1.5)

    assert len(pics) >= 2  # Au moins les 2 pics ajoutés
    assert all(p['puissance'] > 2000 for p in pics)  # Tous au-dessus du seuil

def test_moyennes_glissantes():
    """Test des moyennes glissantes"""
    calc = CalculateurEnergie()

    # Mesures avec bruit
    mesures = []
    base_time = datetime.utcnow()

    for i in range(20):
        # Signal avec bruit
        signal = 1000 + 200 * (i % 5)  # Périodique
        bruit = (i % 3 - 1) * 100      # ±100W de bruit
        puissance = signal + bruit

        mesure = type('Mesure', (), {
            'puissance_active_totale': puissance,
            'timestamp': base_time + timedelta(minutes=i*5)
        })()
        mesures.append(mesure)

    moyennes = calc.calculer_moyennes_glissantes(mesures, fenetre_minutes=10)

    assert len(moyennes) > 0
    # Les moyennes devraient être plus stables que les valeurs originales
    for m in moyennes:
        assert 'moyenne_glissante' in m
        assert 'ecart_type' in m
```

> **💡 À retenir** : Le calcul précis des coûts nécessite la connaissance des tarifs horaires et des taxes, qui varient selon le fournisseur et la région.

> **⚠️ Astuce** : Implémentez une gestion des tarifs configurable pour supporter différents fournisseurs et options (Tempo, EJP, etc.).

Dans le prochain chapitre, nous implémenterons la **détection automatique d'anomalies** pour alerter sur les consommations suspectes !

---

**Navigation**
- [Chapitre précédent : Webhook vers InfluxDB + Grafana](Chapitre_16_Webhook_InfluxDB_Grafana.md)
- [Chapitre suivant : Détection automatique d'anomalies](Chapitre_18_Detection_Anomalies.md)
- [Retour à la table des matières](../../README.md)
