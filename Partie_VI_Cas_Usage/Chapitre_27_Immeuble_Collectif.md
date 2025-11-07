# 🏢 Chapitre 27 : Immeuble collectif

## 🏠 Centralisation M-Bus et gestion multi-locataires

Dans un immeuble collectif, le compteur E450 permet la centralisation des données énergétiques de tous les logements via le bus M-Bus, facilitant la gestion technique et la répartition équitable des coûts.

### Architecture M-Bus collective

#### Configuration réseau

```
Immeuble ──┬──► Armoire électrique collective
          │
          ├──► Compteurs E450 (locataires) - Bus M-Bus
          │     • Appartement 1: Adresse 1
          │     • Appartement 2: Adresse 2
          │     • ...
          │     • Appartement 20: Adresse 20
          │
          ├──► Compteur général (consommation totale)
          │
          ├──► Serveur de supervision
          │
          └──► Interface web syndic
```

#### Avantages de la centralisation

- ✅ **Répartition équitable** : Consommation par appartement
- ✅ **Détection fuites** : Comparaison avec compteur général
- ✅ **Maintenance préventive** : Surveillance continue
- ✅ **Optimisation globale** : Gestion chauffage/climatisation
- ✅ **Reporting syndic** : Tableaux de bord détaillés

### Système de supervision collective

#### Gestionnaire multi-compteurs

```python
# app/services/gestion_immeuble.py
from app.models import MesureEnergie, CompteurLocataire, ConsommationGlobale
from app.extensions import db
from datetime import datetime, timedelta
from typing import List, Dict, Optional
import logging

logger = logging.getLogger(__name__)

class GestionnaireImmeuble:
    """Gestionnaire d'immeuble avec multi-compteurs M-Bus"""

    def __init__(self, config_immeuble):
        self.config = config_immeuble
        self.nb_locataires = config_immeuble.get('nb_locataires', 20)
        self.puissance_souscrite = config_immeuble.get('puissance_souscrite', 36)  # kVA

        # Seuils d'alerte
        self.seuils = {
            'consommation_max_locataire': 15,  # kWh/jour
            'depasseement_global': 0.9,        # 90% de la puissance souscrite
            'fuite_detectee': 0.95,            # 95% correlation min
        }

    def analyser_consommation_globale(self, date_analyse=None):
        """
        Analyse de la consommation globale de l'immeuble

        Args:
            date_analyse: Date d'analyse (défaut: aujourd'hui)

        Returns:
            Analyse globale détaillée
        """
        if date_analyse is None:
            date_analyse = datetime.utcnow().date()

        # Consommations individuelles
        consommations_locataires = self._recuperer_consommations_locataires(date_analyse)

        # Consommation globale (compteur principal)
        conso_globale = self._recuperer_consommation_globale(date_analyse)

        # Analyses
        repartition = self._analyser_repartition(consommations_locataires)
        anomalies = self._detecter_anomalies_collectives(consommations_locataires, conso_globale)
        efficacite = self._calculer_efficacite_globale(consommations_locataires)

        return {
            'date': date_analyse.isoformat(),
            'consommation_globale': conso_globale,
            'consommations_locataires': consommations_locataires,
            'repartition': repartition,
            'anomalies': anomalies,
            'efficacite': efficacite,
            'recommandations': self._generer_recommandations_syndic(anomalies, repartition)
        }

    def _recuperer_consommations_locataires(self, date):
        """Récupère les consommations individuelles"""
        debut_jour = datetime.combine(date, datetime.min.time())
        fin_jour = datetime.combine(date, datetime.max.time())

        consommations = {}

        # Simulation de récupération M-Bus
        # En production: interrogation de chaque compteur
        for locataire_id in range(1, self.nb_locataires + 1):
            # Simulation de données (remplacer par vraie interrogation M-Bus)
            mesures_locataire = MesureEnergie.query.filter(
                MesureEnergie.timestamp.between(debut_jour, fin_jour),
                MesureEnergie.compteur_id == f'LOC{locataire_id:02d}'
            ).all()

            if mesures_locataire:
                energie_debut = mesures_locataire[0].energie_active_import or 0
                energie_fin = mesures_locataire[-1].energie_active_import or 0
                conso_jour = energie_fin - energie_debut

                consommations[f'locataire_{locataire_id}'] = {
                    'energie_consommee': conso_jour,
                    'puissance_max': max((m.puissance_active_totale or 0) for m in mesures_locataire),
                    'nb_mesures': len(mesures_locataire),
                    'statut': 'actif'
                }
            else:
                consommations[f'locataire_{locataire_id}'] = {
                    'energie_consommee': 0,
                    'puissance_max': 0,
                    'nb_mesures': 0,
                    'statut': 'inactif'
                }

        return consommations

    def _recuperer_consommation_globale(self, date):
        """Récupère la consommation globale"""
        debut_jour = datetime.combine(date, datetime.min.time())
        fin_jour = datetime.combine(date, datetime.max.time())

        mesures_globales = MesureEnergie.query.filter(
            MesureEnergie.timestamp.between(debut_jour, fin_jour),
            MesureEnergie.compteur_id == 'GLOBAL'
        ).all()

        if mesures_globales:
            energie_debut = mesures_globales[0].energie_active_import or 0
            energie_fin = mesures_globales[-1].energie_active_import or 0
            conso_globale = energie_fin - energie_debut

            return {
                'energie_consommee': conso_globale,
                'puissance_max': max((m.puissance_active_totale or 0) for m in mesures_globales),
                'puissance_moyenne': sum((m.puissance_active_totale or 0) for m in mesures_globales) / len(mesures_globales),
                'nb_mesures': len(mesures_globales)
            }

        return {
            'energie_consommee': 0,
            'puissance_max': 0,
            'puissance_moyenne': 0,
            'nb_mesures': 0
        }

    def _analyser_repartition(self, consommations_locataires):
        """Analyse la répartition des consommations"""
        energies = [c['energie_consommee'] for c in consommations_locataires.values()]

        if not energies:
            return {'erreur': 'Aucune donnée'}

        repartition = {
            'total_locataires': sum(energies),
            'moyenne_par_locataire': sum(energies) / len(energies),
            'mediane': sorted(energies)[len(energies)//2],
            'minimum': min(energies),
            'maximum': max(energies),
            'ecart_type': statistics.stdev(energies) if len(energies) > 1 else 0,
            'coefficient_variation': statistics.stdev(energies) / (sum(energies)/len(energies)) if energies else 0
        }

        # Classes de consommation
        seuils = [5, 10, 15, 20]  # kWh/jour
        classes = {'faible': 0, 'moyen': 0, 'eleve': 0, 'tres_eleve': 0}

        for energie in energies:
            if energie <= seuils[0]:
                classes['faible'] += 1
            elif energie <= seuils[1]:
                classes['moyen'] += 1
            elif energie <= seuils[2]:
                classes['eleve'] += 1
            else:
                classes['tres_eleve'] += 1

        repartition['classes_consommation'] = classes

        return repartition

    def _detecter_anomalies_collectives(self, consommations_locataires, conso_globale):
        """Détecte les anomalies au niveau collectif"""
        anomalies = []

        # 1. Comparaison somme individuelle vs globale
        somme_individuelle = sum(c['energie_consommee'] for c in consommations_locataires.values())
        globale = conso_globale['energie_consommee']

        if globale > 0:
            correlation = somme_individuelle / globale
            if correlation < self.seuils['fuite_detectee']:
                anomalies.append({
                    'type': 'fuite_energetique',
                    'severite': 'majeur',
                    'description': f'Fuite détectée: {correlation:.1%} de corrélation',
                    'valeur': correlation,
                    'impact_estime': globale - somme_individuelle
                })

        # 2. Dépassement de puissance souscrite
        puissance_max_globale = conso_globale['puissance_max']
        seuil_global = self.puissance_souscrite * 1000 * self.seuils['depasseement_global']

        if puissance_max_globale > seuil_global:
            anomalies.append({
                'type': 'depasseement_puissance',
                'severite': 'critique',
                'description': f'Puissance max: {puissance_max_globale:.0f}W > seuil {seuil_global:.0f}W',
                'valeur': puissance_max_globale,
                'seuil': seuil_global
            })

        # 3. Locataires à consommation nulle (défaillance compteur)
        locataires_inactifs = [
            loc_id for loc_id, data in consommations_locataires.items()
            if data['energie_consommee'] == 0 and data['statut'] == 'actif'
        ]

        if locataires_inactifs:
            anomalies.append({
                'type': 'compteurs_defaillants',
                'severite': 'mineur',
                'description': f'{len(locataires_inactifs)} compteurs inactifs: {", ".join(locataires_inactifs)}',
                'locataires_concernes': locataires_inactifs
            })

        return anomalies

    def _calculer_efficacite_globale(self, consommations_locataires):
        """Calcule l'efficacité énergétique globale"""
        energies = [c['energie_consommee'] for c in consommations_locataires.values() if c['energie_consommee'] > 0]

        if not energies:
            return {'erreur': 'Données insuffisantes'}

        # Indicateurs d'efficacité
        moyenne = sum(energies) / len(energies)
        mediane = sorted(energies)[len(energies)//2]

        # Score d'homogénéité (plus c'est proche de 1, mieux c'est)
        coefficient_variation = statistics.stdev(energies) / moyenne if len(energies) > 1 else 0
        score_homogeneite = max(0, 1 - coefficient_variation)

        return {
            'moyenne_journaliere': moyenne,
            'mediane': mediane,
            'score_homogeneite': score_homogeneite * 100,  # %
            'coefficient_variation': coefficient_variation * 100,  # %
            'interpretation': self._interpreter_efficacite(score_homogeneite)
        }

    def _interpreter_efficacite(self, score_homogeneite):
        """Interprète le score d'efficacité"""
        if score_homogeneite > 0.8:
            return "Excellente homogénéité - Gestion énergétique équilibrée"
        elif score_homogeneite > 0.6:
            return "Bonne homogénéité - Quelques écarts à surveiller"
        elif score_homogeneite > 0.4:
            return "Homogénéité moyenne - Optimisations possibles"
        else:
            return "Faible homogénéité - Actions correctives nécessaires"

    def _generer_recommandations_syndic(self, anomalies, repartition):
        """Génère des recommandations pour le syndic"""
        recommandations = []

        # Recommandations basées sur anomalies
        for anomalie in anomalies:
            if anomalie['type'] == 'fuite_energetique':
                recommandations.append({
                    'priorite': 'haute',
                    'domaine': 'technique',
                    'action': 'Inspection des installations communes',
                    'description': f'Investiguer la fuite énergétique de {anomalie["impact_estime"]:.1f} kWh/jour'
                })
            elif anomalie['type'] == 'depasseement_puissance':
                recommandations.append({
                    'priorite': 'critique',
                    'domaine': 'electricite',
                    'action': 'Augmenter puissance souscrite',
                    'description': f'Puissance actuelle insuffisante ({self.puissance_souscrite} kVA)'
                })

        # Recommandations basées sur répartition
        if repartition.get('coefficient_variation', 0) > 30:
            recommandations.append({
                'priorite': 'moyenne',
                'domaine': 'comportement',
                'action': 'Campagne de sensibilisation',
                'description': 'Écarts importants de consommation entre locataires'
            })

        # Recommandations générales
        recommandations.extend([
            {
                'priorite': 'basse',
                'domaine': 'maintenance',
                'action': 'Maintenance préventive compteurs',
                'description': 'Vérification annuelle des compteurs individuels'
            },
            {
                'priorite': 'basse',
                'domaine': 'reporting',
                'action': 'Rapport énergétique annuel',
                'description': 'Établir bilan énergétique détaillé'
            }
        ])

        return recommandations

    def calculer_repartition_charges(self, periode_facturation='mois'):
        """
        Calcule la répartition des charges énergétiques

        Args:
            periode_facturation: 'mois', 'trimestre', 'annee'

        Returns:
            Répartition détaillée par locataire
        """
        # Période d'analyse
        if periode_facturation == 'mois':
            debut = (datetime.utcnow() - timedelta(days=30)).replace(day=1)
            fin = datetime.utcnow()
        elif periode_facturation == 'trimestre':
            debut = (datetime.utcnow() - timedelta(days=90)).replace(day=1)
            fin = datetime.utcnow()
        else:  # année
            debut = datetime(datetime.utcnow().year, 1, 1)
            fin = datetime.utcnow()

        # Récupération des données
        repartition = {}

        for locataire_id in range(1, self.nb_locataires + 1):
            mesures = MesureEnergie.query.filter(
                MesureEnergie.timestamp.between(debut, fin),
                MesureEnergie.compteur_id == f'LOC{locataire_id:02d}'
            ).order_by(MesureEnergie.timestamp).all()

            if mesures:
                # Calcul de la consommation sur la période
                energie_debut = mesures[0].energie_active_import or 0
                energie_fin = mesures[-1].energie_active_import or 0
                conso_periode = energie_fin - energie_debut

                repartition[f'locataire_{locataire_id}'] = {
                    'consommation_kwh': conso_periode,
                    'jours_factures': (fin - debut).days,
                    'statut': 'actif'
                }
            else:
                repartition[f'locataire_{locataire_id}'] = {
                    'consommation_kwh': 0,
                    'jours_factures': (fin - debut).days,
                    'statut': 'inactif'
                }

        # Calcul des parts
        total_conso = sum(r['consommation_kwh'] for r in repartition.values())
        total_locataires_actifs = sum(1 for r in repartition.values() if r['statut'] == 'actif')

        for locataire, data in repartition.items():
            if total_conso > 0:
                data['part_consommation'] = data['consommation_kwh'] / total_conso * 100
                data['part_egalitaire'] = 100 / total_locataires_actifs
                data['ecart_vs_moyenne'] = data['part_consommation'] - data['part_egalitaire']
            else:
                data['part_consommation'] = 0
                data['part_egalitaire'] = 100 / total_locataires_actifs
                data['ecart_vs_moyenne'] = -data['part_egalitaire']

        return {
            'periode': {
                'debut': debut.isoformat(),
                'fin': fin.isoformat(),
                'type': periode_facturation
            },
            'repartition': repartition,
            'total_general': {
                'consommation_totale': total_conso,
                'locataires_actifs': total_locataires_actifs,
                'consommation_moyenne': total_conso / total_locataires_actifs if total_locataires_actifs > 0 else 0
            }
        }
```

### API pour gestion d'immeuble

```python
# app/routes/api_immeuble.py
from flask import Blueprint, request, jsonify
from app.services.gestion_immeuble import GestionnaireImmeuble
from datetime import datetime, timedelta

immeuble_bp = Blueprint('immeuble', __name__)

# Configuration de l'immeuble (en production: base de données)
CONFIG_IMMEUBLE = {
    'nb_locataires': 20,
    'puissance_souscrite': 36,  # kVA
    'adresse': '123 Rue de l\'Immeuble',
    'syndic': 'M. Dupont'
}

gestionnaire = GestionnaireImmeuble(CONFIG_IMMEUBLE)

@immeuble_bp.route('/api/immeuble/analyse-quotidienne')
def analyse_quotidienne():
    """Analyse quotidienne de l'immeuble"""

    jours = int(request.args.get('days', 1))
    analyses = []

    for i in range(jours):
        date_analyse = (datetime.utcnow() - timedelta(days=i)).date()
        analyse = gestionnaire.analyser_consommation_globale(date_analyse)
        analyses.append(analyse)

    return jsonify({
        'analyses': analyses,
        'periode': f"{jours} jour(s)",
        'config_immeuble': CONFIG_IMMEUBLE
    })

@immeuble_bp.route('/api/immeuble/repartition-charges')
def repartition_charges():
    """Répartition des charges énergétiques"""

    periode = request.args.get('periode', 'mois')

    repartition = gestionnaire.calculer_repartition_charges(periode)

    return jsonify(repartition)

@immeuble_bp.route('/api/immeuble/tableau-bord-syndic')
def tableau_bord_syndic():
    """Tableau de bord complet pour le syndic"""

    # Analyse du mois en cours
    analyse_mois = gestionnaire.analyser_consommation_globale()

    # Répartition des charges
    repartition = gestionnaire.calculer_repartition_charges('mois')

    # Tendances sur 30 jours
    tendances = []
    for i in range(30):
        date_analyse = (datetime.utcnow() - timedelta(days=i)).date()
        analyse_jour = gestionnaire.analyser_consommation_globale(date_analyse)
        tendances.append({
            'date': date_analyse.isoformat(),
            'conso_totale': analyse_jour.get('consommation_globale', {}).get('energie_consommee', 0),
            'nb_anomalies': len(analyse_jour.get('anomalies', []))
        })

    return jsonify({
        'config_immeuble': CONFIG_IMMEUBLE,
        'analyse_actuelle': analyse_mois,
        'repartition_charges': repartition,
        'tendances_30j': tendances,
        'kpis_synthese': {
            'conso_moyenne_journaliere': sum(t['conso_totale'] for t in tendances) / len(tendances),
            'anomalies_moyennes': sum(t['nb_anomalies'] for t in tendances) / len(tendances),
            'locataires_suivis': CONFIG_IMMEUBLE['nb_locataires']
        }
    })

@immeuble_bp.route('/api/immeuble/rapports-locataires')
def rapports_locataires():
    """Rapports individuels pour les locataires"""

    locataire_id = request.args.get('locataire')
    jours = int(request.args.get('days', 30))

    if not locataire_id:
        return jsonify({'error': 'ID locataire requis'}), 400

    debut = datetime.utcnow() - timedelta(days=jours)

    # Mesures du locataire
    mesures = MesureEnergie.query.filter(
        MesureEnergie.timestamp >= debut,
        MesureEnergie.compteur_id == f'LOC{int(locataire_id):02d}'
    ).order_by(MesureEnergie.timestamp).all()

    if not mesures:
        return jsonify({
            'locataire': locataire_id,
            'message': 'Aucune donnée disponible',
            'periode': f"{jours} jours"
        })

    # Analyse individuelle
    analyse = {
        'locataire': locataire_id,
        'periode': {
            'debut': debut.isoformat(),
            'fin': datetime.utcnow().isoformat(),
            'jours': jours
        },
        'statistiques': {
            'nb_mesures': len(mesures),
            'puissance_max': max((m.puissance_active_totale or 0) for m in mesures),
            'puissance_moyenne': sum((m.puissance_active_totale or 0) for m in mesures) / len(mesures),
            'energie_totale': sum((m.energie_active_import or 0) for m in mesures[-1:]) - sum((m.energie_active_import or 0) for m in mesures[:1]) if mesures else 0
        },
        'consommation_quotidienne': [],  # À calculer par jour
        'alertes_personnelles': []  # Alertes spécifiques au locataire
    }

    return jsonify(analyse)

@immeuble_bp.route('/api/immeuble/maintenance')
def maintenance_immeuble():
    """Rapport de maintenance des compteurs"""

    # Vérification de tous les compteurs
    rapport_maintenance = {
        'date_verification': datetime.utcnow().isoformat(),
        'compteurs_verifies': CONFIG_IMMEUBLE['nb_locataires'],
        'statuts': {
            'actifs': 0,
            'inactifs': 0,
            'anomalies': 0
        },
        'recommandations': [
            'Vérification semestrielle des compteurs',
            'Calibration annuelle',
            'Nettoyage des optiques',
            'Test de communication M-Bus'
        ]
    }

    # Simulation de vérification (en production: vraie interrogation)
    rapport_maintenance['statuts']['actifs'] = CONFIG_IMMEUBLE['nb_locataires'] - 2
    rapport_maintenance['statuts']['inactifs'] = 2

    return jsonify(rapport_maintenance)
```

> **💡 À retenir** : Dans un immeuble collectif, le compteur E450 avec M-Bus permet une gestion énergétique équitable et transparente, facilitant la maintenance et l'optimisation des consommations.

> **⚠️ Astuce** : Implémentez un système de notification automatique pour alerter les locataires en cas de consommation anormalement élevée ou de problèmes techniques.

Félicitations ! La Partie VI sur les cas d'usage pratiques est maintenant complète. Vous maîtrisez maintenant les applications concrètes du compteur E450 dans différents environnements : appartement, atelier industriel, maison solaire et immeuble collectif. La Partie VII va explorer la pédagogie et les références !

---

**Navigation**
- [Chapitre précédent : Maison solaire](Chapitre_26_Maison_Solaire.md)
- [Partie VII : Pédagogie & références](../Partie_VII_Pedagogie_References/)
- [Retour à la table des matières](../../README.md)
