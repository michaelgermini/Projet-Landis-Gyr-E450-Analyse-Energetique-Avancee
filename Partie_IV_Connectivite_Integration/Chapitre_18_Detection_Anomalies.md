# 🔌 Chapitre 18 : Détection automatique d'anomalies

## 🔍 Algorithmes de moyenne mobile et seuils

La détection d'anomalies est cruciale pour identifier les problèmes électriques ou les changements de comportement de consommation.

### Service de détection d'anomalies

```python
# app/services/detecteur_anomalies.py
from app.models import MesureEnergie, EvenementCompteur
from app.extensions import db
from datetime import datetime, timedelta
from typing import List, Dict, Optional, Tuple
import statistics
import numpy as np
from enum import Enum

class TypeAnomalie(Enum):
    """Types d'anomalies détectables"""
    PIC_CONSOMMATION = "pic_consommation"
    CHUTE_CONSOMMATION = "chute_consommation"
    VARIATION_BRUTALE = "variation_brutale"
    TENSION_ANORMALE = "tension_anormale"
    COURANT_DISPARITE = "courant_disparite"
    FREQUENCE_INSTABLE = "frequence_instable"
    FACTEUR_PUISSANCE_FAIBLE = "facteur_puissance_faible"
    HARMONIQUES_ELEVES = "harmoniques_eleves"

class SeveriteAnomalie(Enum):
    """Niveaux de sévérité"""
    INFO = "info"
    WARNING = "warning"
    ERROR = "error"
    CRITICAL = "critical"

class DetecteurAnomalies:
    """Service de détection d'anomalies énergétiques"""

    def __init__(self):
        # Seuils configurables
        self.seuils = {
            'pic_consommation_multiplicateur': 2.5,  # x fois la moyenne
            'chute_consommation_ratio': 0.1,          # % de la moyenne normale
            'variation_brutale_seuil': 1000,          # W par minute
            'tension_min': 200,                       # V
            'tension_max': 250,                       # V
            'frequence_min': 49.5,                    # Hz
            'frequence_max': 50.5,                    # Hz
            'facteur_puissance_min': 0.8,             # -
            'thd_max': 8.0,                          # %
            'disparite_phases_max': 20,              # % d'écart max entre phases
        }

        # Historique pour calculs statistiques
        self.cache_stats = {}
        self.cache_validite = datetime.utcnow()

    def detecter_anomalie_mesure(self, mesure: MesureEnergie) -> List[Dict]:
        """
        Détecte les anomalies sur une mesure individuelle

        Args:
            mesure: MesureEnergie à analyser

        Returns:
            Liste des anomalies détectées
        """
        anomalies = []

        # Récupération du contexte statistique
        stats = self._calculer_stats_contexte(mesure.timestamp)

        # 1. Détection de pics de consommation
        if mesure.puissance_active_totale and stats['moyenne_puissance']:
            seuil_pic = stats['moyenne_puissance'] * self.seuils['pic_consommation_multiplicateur']
            if mesure.puissance_active_totale > seuil_pic:
                anomalies.append({
                    'type': TypeAnomalie.PIC_CONSOMMATION.value,
                    'severite': SeveriteAnomalie.WARNING.value,
                    'titre': 'Pic de consommation détecté',
                    'description': f'Puissance {mesure.puissance_active_totale:.0f}W, seuil {seuil_pic:.0f}W',
                    'valeur': mesure.puissance_active_totale,
                    'seuil': seuil_pic,
                    'code_obis': '16.7.0'
                })

        # 2. Anomalies de tension
        tensions = [mesure.tension_l1, mesure.tension_l2, mesure.tension_l3]
        tensions_valides = [t for t in tensions if t is not None]

        if tensions_valides:
            tension_moy = statistics.mean(tensions_valides)

            if tension_moy < self.seuils['tension_min']:
                anomalies.append({
                    'type': TypeAnomalie.TENSION_ANORMALE.value,
                    'severite': SeveriteAnomalie.ERROR.value,
                    'titre': 'Tension trop basse',
                    'description': f'Tension moyenne {tension_moy:.1f}V (min {self.seuils["tension_min"]}V)',
                    'valeur': tension_moy,
                    'seuil': self.seuils['tension_min'],
                    'code_obis': '32.7.0'
                })
            elif tension_moy > self.seuils['tension_max']:
                anomalies.append({
                    'type': TypeAnomalie.TENSION_ANORMALE.value,
                    'severite': SeveriteAnomalie.ERROR.value,
                    'titre': 'Tension trop élevée',
                    'description': f'Tension moyenne {tension_moy:.1f}V (max {self.seuils["tension_max"]}V)',
                    'valeur': tension_moy,
                    'seuil': self.seuils['tension_max'],
                    'code_obis': '32.7.0'
                })

        # 3. Fréquence instable
        if mesure.frequence:
            if not (self.seuils['frequence_min'] <= mesure.frequence <= self.seuils['frequence_max']):
                anomalies.append({
                    'type': TypeAnomalie.FREQUENCE_INSTABLE.value,
                    'severite': SeveriteAnomalie.WARNING.value,
                    'titre': 'Fréquence réseau instable',
                    'description': f'Fréquence {mesure.frequence:.2f}Hz (normal: 50.00Hz)',
                    'valeur': mesure.frequence,
                    'seuil': f"{self.seuils['frequence_min']}-{self.seuils['frequence_max']}",
                    'code_obis': '14.7.0'
                })

        # 4. Facteur de puissance faible
        if mesure.facteur_puissance and mesure.facteur_puissance < self.seuils['facteur_puissance_min']:
            anomalies.append({
                'type': TypeAnomalie.FACTEUR_PUISSANCE_FAIBLE.value,
                'severite': SeveriteAnomalie.INFO.value,
                'titre': 'Facteur de puissance faible',
                'description': f'FP = {mesure.facteur_puissance:.3f} (min recommandé {self.seuils["facteur_puissance_min"]})',
                'valeur': mesure.facteur_puissance,
                'seuil': self.seuils['facteur_puissance_min'],
                'code_obis': '13.7.0'
            })

        # 5. Harmoniques élevés
        if mesure.thd_tension and mesure.thd_tension > self.seuils['thd_max']:
            anomalies.append({
                'type': TypeAnomalie.HARMONIQUES_ELEVES.value,
                'severite': SeveriteAnomalie.WARNING.value,
                'titre': 'Distorsion harmonique élevée',
                'description': f'THD tension = {mesure.thd_tension:.1f}% (max {self.seuils["thd_max"]}%)',
                'valeur': mesure.thd_tension,
                'seuil': self.seuils['thd_max'],
                'code_obis': '7.7.0'
            })

        # 6. Disparité entre phases
        disparite = self._calculer_disparite_phases(mesure)
        if disparite and disparite > self.seuils['disparite_phases_max']:
            anomalies.append({
                'type': TypeAnomalie.COURANT_DISPARITE.value,
                'severite': SeveriteAnomalie.WARNING.value,
                'titre': 'Déséquilibre des phases',
                'description': f'Écart entre phases: {disparite:.1f}% (max {self.seuils["disparite_phases_max"]}%)',
                'valeur': disparite,
                'seuil': self.seuils['disparite_phases_max'],
                'code_obis': '31.7.0'
            })

        return anomalies

    def detecter_anomalies_tendances(self, mesures: List[MesureEnergie]) -> List[Dict]:
        """
        Détecte les anomalies sur des tendances (séries temporelles)

        Args:
            mesures: Liste de mesures triées par timestamp

        Returns:
            Liste des anomalies de tendance
        """
        if len(mesures) < 10:
            return []  # Pas assez de données

        anomalies = []

        # 1. Variations brutales
        variations = self._calculer_variations_brutales(mesures)
        seuil_variation = self.seuils['variation_brutale_seuil']

        for var in variations:
            if var['variation_w_par_min'] > seuil_variation:
                anomalies.append({
                    'type': TypeAnomalie.VARIATION_BRUTALE.value,
                    'severite': SeveriteAnomalie.WARNING.value,
                    'titre': 'Variation brutale de puissance',
                    'description': f'Variation de {var["variation_w_par_min"]:.0f}W/min à {var["timestamp"]}',
                    'valeur': var['variation_w_par_min'],
                    'seuil': seuil_variation,
                    'timestamp': var['timestamp']
                })

        # 2. Chutes de consommation prolongées
        chutes = self._detecter_chutes_consommation(mesures)
        seuil_chute = self.seuils['chute_consommation_ratio']

        for chute in chutes:
            if chute['ratio_moyenne'] < seuil_chute:
                anomalies.append({
                    'type': TypeAnomalie.CHUTE_CONSOMMATION.value,
                    'severite': SeveriteAnomalie.ERROR.value,
                    'titre': 'Chute de consommation détectée',
                    'description': f'Consommation {chute["ratio_moyenne"]:.1%} de la moyenne normale',
                    'valeur': chute['ratio_moyenne'],
                    'seuil': seuil_chute,
                    'timestamp': chute['periode']
                })

        return anomalies

    def _calculer_stats_contexte(self, timestamp: datetime) -> Dict:
        """
        Calcule les statistiques de contexte pour une période donnée
        """
        # Cache pour éviter les recalculs
        cache_key = timestamp.date()
        if cache_key in self.cache_stats and (datetime.utcnow() - self.cache_validite).seconds < 3600:
            return self.cache_stats[cache_key]

        # Période de référence (derniers 7 jours, même heure)
        debut_ref = timestamp - timedelta(days=7)
        fin_ref = timestamp - timedelta(days=1)

        mesures_ref = MesureEnergie.query.filter(
            MesureEnergie.timestamp.between(debut_ref, fin_ref),
            db.extract('hour', MesureEnergie.timestamp) == timestamp.hour
        ).all()

        if mesures_ref:
            puissances = [m.puissance_active_totale for m in mesures_ref if m.puissance_active_totale]
            stats = {
                'moyenne_puissance': statistics.mean(puissances) if puissances else None,
                'ecart_type_puissance': statistics.stdev(puissances) if len(puissances) > 1 else 0,
                'min_puissance': min(puissances) if puissances else None,
                'max_puissance': max(puissances) if puissances else None,
                'nb_mesures': len(puissances)
            }
        else:
            stats = {
                'moyenne_puissance': None,
                'ecart_type_puissance': 0,
                'min_puissance': None,
                'max_puissance': None,
                'nb_mesures': 0
            }

        # Mise en cache
        self.cache_stats[cache_key] = stats
        self.cache_validite = datetime.utcnow()

        return stats

    def _calculer_disparite_phases(self, mesure: MesureEnergie) -> Optional[float]:
        """Calcule le déséquilibre entre phases"""
        courants = [mesure.courant_l1, mesure.courant_l2, mesure.courant_l3]
        courants_valides = [c for c in courants if c is not None]

        if len(courants_valides) < 2:
            return None

        courant_moyen = statistics.mean(courants_valides)
        if courant_moyen == 0:
            return 0

        # Écart maximum par rapport à la moyenne
        ecarts = [abs(c - courant_moyen) / courant_moyen * 100 for c in courants_valides]
        return max(ecarts)

    def _calculer_variations_brutales(self, mesures: List[MesureEnergie]) -> List[Dict]:
        """Détecte les variations brutales de puissance"""
        variations = []

        for i in range(1, len(mesures)):
            mesure_prec = mesures[i-1]
            mesure_actu = mesures[i]

            if not (mesure_prec.puissance_active_totale and mesure_actu.puissance_active_totale):
                continue

            # Durée entre mesures (minutes)
            duree_min = (mesure_actu.timestamp - mesure_prec.timestamp).total_seconds() / 60

            if duree_min > 0:
                variation_w = mesure_actu.puissance_active_totale - mesure_prec.puissance_active_totale
                variation_w_par_min = abs(variation_w) / duree_min

                variations.append({
                    'timestamp': mesure_actu.timestamp.isoformat(),
                    'variation_w': variation_w,
                    'variation_w_par_min': variation_w_par_min,
                    'duree_min': duree_min
                })

        return variations

    def _detecter_chutes_consommation(self, mesures: List[MesureEnergie]) -> List[Dict]:
        """Détecte les périodes de chute de consommation"""
        if len(mesures) < 20:
            return []

        # Calculer la moyenne générale
        puissances = [m.puissance_active_totale for m in mesures if m.puissance_active_totale]
        if not puissances:
            return []

        moyenne_generale = statistics.mean(puissances)

        # Détecter les périodes basses
        chutes = []
        i = 0

        while i < len(mesures) - 5:  # Minimum 5 mesures pour une période
            # Fenêtre glissante de 5 mesures
            fenetre = mesures[i:i+5]
            puissances_fenetre = [m.puissance_active_totale for m in fenetre if m.puissance_active_totale]

            if len(puissances_fenetre) >= 3:
                moyenne_fenetre = statistics.mean(puissances_fenetre)
                ratio = moyenne_fenetre / moyenne_generale if moyenne_generale > 0 else 0

                if ratio < 0.3:  # Chute significative
                    chutes.append({
                        'periode': f"{fenetre[0].timestamp.isoformat()} à {fenetre[-1].timestamp.isoformat()}",
                        'moyenne_fenetre': moyenne_fenetre,
                        'ratio_moyenne': ratio,
                        'moyenne_generale': moyenne_generale
                    })

                    i += 5  # Sauter la période détectée
                    continue

            i += 1

        return chutes

    def analyser_anomalies_periode(self, debut: datetime, fin: datetime) -> Dict:
        """
        Analyse complète des anomalies sur une période

        Args:
            debut, fin: Période d'analyse

        Returns:
            Rapport d'analyse des anomalies
        """
        mesures = MesureEnergie.query.filter(
            MesureEnergie.timestamp.between(debut, fin)
        ).order_by(MesureEnergie.timestamp).all()

        if not mesures:
            return {'erreur': 'Aucune mesure dans la période'}

        # Analyse individuelle
        anomalies_individuelles = []
        for mesure in mesures:
            anomalies = self.detecter_anomalie_mesure(mesure)
            anomalies_individuelles.extend(anomalies)

        # Analyse de tendances
        anomalies_tendances = self.detecter_anomalies_tendances(mesures)

        # Statistiques
        stats_anomalies = {}
        for anomalie in anomalies_individuelles + anomalies_tendances:
            type_anomalie = anomalie['type']
            if type_anomalie not in stats_anomalies:
                stats_anomalies[type_anomalie] = 0
            stats_anomalies[type_anomalie] += 1

        # Anomalies les plus critiques
        anomalies_critiques = [
            a for a in anomalies_individuelles + anomalies_tendances
            if a['severite'] in ['error', 'critical']
        ]

        return {
            'periode': {
                'debut': debut.isoformat(),
                'fin': fin.isoformat(),
                'duree_heures': (fin - debut).total_seconds() / 3600
            },
            'nb_mesures_analysees': len(mesures),
            'anomalies_total': len(anomalies_individuelles) + len(anomalies_tendances),
            'anomalies_par_type': stats_anomalies,
            'anomalies_critiques': len(anomalies_critiques),
            'anomalies_recentes': sorted(
                anomalies_individuelles + anomalies_tendances,
                key=lambda x: x.get('timestamp', datetime.utcnow().isoformat()),
                reverse=True
            )[:10]  # Top 10 récentes
        }

    def get_seuils_config(self) -> Dict:
        """Retourne la configuration des seuils"""
        return self.seuils.copy()

    def update_seuil(self, nom_seuil: str, valeur: float):
        """Met à jour un seuil de détection"""
        if nom_seuil in self.seuils:
            self.seuils[nom_seuil] = valeur
            # Invalider le cache
            self.cache_validite = datetime.utcnow() - timedelta(hours=2)
            return True
        return False
```

### API pour les anomalies

```python
# app/routes/api_anomalies.py
from flask import Blueprint, request, jsonify
from app.services.detecteur_anomalies import DetecteurAnomalies
from app.models import EvenementCompteur
from datetime import datetime, timedelta

anomalies_bp = Blueprint('anomalies', __name__)
detecteur = DetecteurAnomalies()

@anomalies_bp.route('/api/anomalies/analyse')
def analyser_periode():
    """Analyse des anomalies sur une période"""

    # Paramètres
    heures = int(request.args.get('hours', 24))

    debut = datetime.utcnow() - timedelta(hours=heures)
    fin = datetime.utcnow()

    # Analyse
    rapport = detecteur.analyser_anomalies_periode(debut, fin)

    return jsonify(rapport)

@anomalies_bp.route('/api/anomalies/seuils')
def get_seuils():
    """Récupération des seuils de détection"""
    return jsonify(detecteur.get_seuils_config())

@anomalies_bp.route('/api/anomalies/seuils', methods=['POST'])
def update_seuil():
    """Mise à jour d'un seuil"""

    data = request.get_json()
    if not data or 'nom' not in data or 'valeur' not in data:
        return jsonify({'error': 'nom et valeur requis'}), 400

    success = detecteur.update_seuil(data['nom'], data['valeur'])

    if success:
        return jsonify({'message': f'Seuil {data["nom"]} mis à jour'})
    else:
        return jsonify({'error': f'Seuil {data["nom"]} introuvable'}), 404

@anomalies_bp.route('/api/anomalies/recentes')
def anomalies_recentes():
    """Anomalies récentes"""

    limit = int(request.args.get('limit', 20))

    # Récupération des événements d'anomalie
    evenements = EvenementCompteur.query.filter(
        EvenementCompteur.type_evenement.in_([
            'pic_consommation', 'chute_consommation', 'variation_brutale',
            'tension_anormale', 'courant_disparite', 'frequence_instable'
        ])
    ).order_by(EvenementCompteur.timestamp.desc()).limit(limit).all()

    return jsonify({
        'count': len(evenements),
        'anomalies': [evt.to_dict() for evt in evenements]
    })
```

### Alerte via MQTT ou notification Flask

```python
# app/services/notifications.py (extension)
from app.services.detecteur_anomalies import SeveriteAnomalie

def notifier_anomalie(anomalie: Dict, mesure_data: Dict):
    """Notification d'anomalie selon sa sévérité"""

    severite = anomalie.get('severite', 'info')

    # Notification MQTT pour Home Assistant
    if severite in ['warning', 'error', 'critical']:
        publier_alerte_mqtt(anomalie, mesure_data)

    # Notification email pour anomalies critiques
    if severite == 'critical':
        envoyer_alerte_email(anomalie, mesure_data)

    # Notification WebSocket pour interface temps réel
    notifier_websocket(anomalie, mesure_data)

def publier_alerte_mqtt(anomalie: Dict, mesure_data: Dict):
    """Publication d'alerte via MQTT"""

    try:
        from app.services import mqtt_service

        if mqtt_service:
            payload = {
                'timestamp': datetime.utcnow().isoformat(),
                'type': 'anomalie',
                'anomalie': anomalie,
                'mesure': mesure_data
            }

            mqtt_service.publier_donnees(payload)

    except Exception as e:
        logger.error(f"Erreur notification MQTT anomalie: {e}")

def envoyer_alerte_email(anomalie: Dict, mesure_data: Dict):
    """Envoi d'alerte par email"""

    try:
        from app.services import notification_service

        sujet = f"🚨 Alerte critique compteur E450: {anomalie['titre']}"

        # Destinataires configurés
        destinataires = get_destinataires_alertes()

        notification_service.send_email(
            subject=sujet,
            recipients=destinataires,
            template='alerte_anomalie',
            anomalie=anomalie,
            mesure=mesure_data,
            timestamp=datetime.utcnow()
        )

    except Exception as e:
        logger.error(f"Erreur envoi email alerte: {e}")

def notifier_websocket(anomalie: Dict, mesure_data: Dict):
    """Notification via WebSocket"""

    try:
        from app.services import websocket_service

        websocket_service.notify_anomaly(anomalie, mesure_data)

    except Exception as e:
        logger.error(f"Erreur notification WebSocket: {e}")

def get_destinataires_alertes():
    """Récupération des destinataires d'alertes"""
    from flask import current_app

    # Configuration dans app.config ou base de données
    return current_app.config.get('ALERT_EMAIL_RECIPIENTS', [])
```

### Tests de détection d'anomalies

```python
# tests/test_anomalies.py
import pytest
from datetime import datetime, timedelta
from app.services.detecteur_anomalies import DetecteurAnomalies, TypeAnomalie, SeveriteAnomalie

def create_test_mesure(puissance=1000, tension=230, frequence=50.0, timestamp=None):
    """Helper pour créer une mesure de test"""
    from app.models import MesureEnergie

    if timestamp is None:
        timestamp = datetime.utcnow()

    return MesureEnergie(
        puissance_active_totale=puissance,
        tension_l1=tension,
        tension_l2=tension,
        tension_l3=tension,
        frequence=frequence,
        facteur_puissance=0.95,
        timestamp=timestamp
    )

def test_detection_pic_consommation():
    """Test détection pic de consommation"""
    detecteur = DetecteurAnomalies()

    # Mesure normale
    mesure_normale = create_test_mesure(1000)

    # Simuler historique normal
    detecteur._calculer_stats_contexte = lambda ts: {
        'moyenne_puissance': 1000,
        'ecart_type_puissance': 100
    }

    anomalies = detecteur.detecter_anomalie_mesure(mesure_normale)
    assert len(anomalies) == 0  # Pas d'anomalie

    # Mesure avec pic
    mesure_pic = create_test_mesure(3000)  # 3x la moyenne
    anomalies = detecteur.detecter_anomalie_mesure(mesure_pic)

    assert len(anomalies) == 1
    assert anomalies[0]['type'] == TypeAnomalie.PIC_CONSOMMATION.value
    assert anomalies[0]['severite'] == SeveriteAnomalie.WARNING.value

def test_detection_tension_anormale():
    """Test détection tension anormale"""
    detecteur = DetecteurAnomalies()

    # Tension trop basse
    mesure_basse = create_test_mesure(tension=180)
    anomalies = detecteur.detecter_anomalie_mesure(mesure_basse)

    assert len(anomalies) == 1
    assert anomalies[0]['type'] == TypeAnomalie.TENSION_ANORMALE.value
    assert 'trop basse' in anomalies[0]['description']

    # Tension trop élevée
    mesure_haute = create_test_mesure(tension=260)
    anomalies = detecteur.detecter_anomalie_mesure(mesure_haute)

    assert len(anomalies) == 1
    assert 'trop élevée' in anomalies[0]['description']

def test_detection_frequence_instable():
    """Test détection fréquence instable"""
    detecteur = DetecteurAnomalies()

    # Fréquence normale
    mesure_normale = create_test_mesure(frequence=50.0)
    anomalies = detecteur.detecter_anomalie_mesure(mesure_normale)
    assert len([a for a in anomalies if a['type'] == TypeAnomalie.FREQUENCE_INSTABLE.value]) == 0

    # Fréquence instable
    mesure_instable = create_test_mesure(frequence=49.0)
    anomalies = detecteur.detecter_anomalie_mesure(mesure_instable)

    freq_anomalies = [a for a in anomalies if a['type'] == TypeAnomalie.FREQUENCE_INSTABLE.value]
    assert len(freq_anomalies) == 1
    assert '49.00Hz' in freq_anomalies[0]['description']

def test_analyse_periode():
    """Test analyse complète d'une période"""
    detecteur = DetecteurAnomalies()

    # Créer des mesures avec anomalies
    mesures = []
    base_time = datetime.utcnow() - timedelta(hours=1)

    for i in range(60):  # 1 heure de données
        puissance = 1000

        # Ajouter un pic à la 30ème mesure
        if i == 30:
            puissance = 4000  # Pic

        # Tension basse à la 45ème mesure
        tension = 230 if i != 45 else 190

        mesure = create_test_mesure(
            puissance=puissance,
            tension=tension,
            timestamp=base_time + timedelta(minutes=i)
        )
        mesures.append(mesure)

    # Analyse de la période
    rapport = detecteur.analyser_anomalies_periode(
        base_time,
        base_time + timedelta(hours=1)
    )

    assert rapport['nb_mesures_analysees'] == 60
    assert rapport['anomalies_total'] >= 2  # Au moins pic + tension
    assert 'anomalies_par_type' in rapport
    assert 'anomalies_critiques' in rapport
```

> **💡 À retenir** : La détection d'anomalies combine seuils statiques et analyse statistique pour identifier les problèmes électriques en temps réel.

> **⚠️ Astuce** : Ajustez les seuils de détection selon votre installation - un seuil trop sensible génère des fausses alertes, trop laxiste rate les vrais problèmes.

Dans le dernier chapitre de cette partie, nous verrons comment **stocker et gérer l'historique** des anomalies détectées !

---

**Navigation**
- [Chapitre précédent : Calcul du coût énergétique](Chapitre_17_Calcul_Cout_Energetique.md)
- [Chapitre suivant : Historique des anomalies détectées](Chapitre_19_Historique_Anomalies.md)
- [Retour à la table des matières](../../README.md)
