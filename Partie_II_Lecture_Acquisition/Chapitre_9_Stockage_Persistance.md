# ⚙️ Chapitre 9 : Stockage et persistance des données

## 💾 Stratégies de stockage

Le stockage efficace des données du compteur est crucial pour créer un **historique fiable** et permettre les analyses temporelles. Plusieurs stratégies s'offrent à nous selon les besoins.

### Critères de choix du stockage

#### Volume de données

```
Données par jour (estimation) :
• 1 mesure/minute × 1440 = 1440 mesures/jour
• 10 valeurs par mesure × 1440 = 14 400 valeurs/jour
• 1 an = ~5 millions de valeurs
• Taille : ~50-100 MB/an (SQLite)
```

#### Types de données

- **Métriques** : Valeurs numériques (tensions, courants, puissances)
- **Événements** : Logs avec timestamps (coupures, alertes)
- **Métadonnées** : Configuration, calibrage, versions
- **Historique** : Évolution temporelle des mesures

## 🗄️ Base SQLite et SQLAlchemy

### Avantages de SQLite

- ✅ **Zéro configuration** : Fichier unique
- ✅ **Portable** : Même fichier sur tous OS
- ✅ **Robuste** : ACID, transactions
- ✅ **Performant** : Suffisant pour notre usage
- ✅ **Intégré Python** : Pas d'installation supplémentaire

### Modèle de données avec SQLAlchemy

```python
from sqlalchemy import create_engine, Column, Integer, String, Float, DateTime, Text
from sqlalchemy.ext.declarative import declarative_base
from sqlalchemy.orm import sessionmaker
from datetime import datetime
import os

Base = declarative_base()

class MesureEnergie(Base):
    """Modèle pour les mesures énergétiques"""
    __tablename__ = 'mesures_energie'

    id = Column(Integer, primary_key=True, autoincrement=True)
    timestamp = Column(DateTime, default=datetime.utcnow, index=True)

    # Valeurs électriques
    energie_active_import = Column(Float, nullable=True)  # kWh
    energie_active_export = Column(Float, nullable=True)  # kWh
    puissance_active_totale = Column(Float, nullable=True)  # W

    # Tensions par phase
    tension_l1 = Column(Float, nullable=True)  # V
    tension_l2 = Column(Float, nullable=True)  # V
    tension_l3 = Column(Float, nullable=True)  # V

    # Courants par phase
    courant_l1 = Column(Float, nullable=True)  # A
    courant_l2 = Column(Float, nullable=True)  # A
    courant_l3 = Column(Float, nullable=True)  # A

    # Qualité réseau
    frequence = Column(Float, nullable=True)  # Hz
    facteur_puissance = Column(Float, nullable=True)  # 0-1

    # Métadonnées
    source = Column(String(50), default='e450')  # e450, manuel, etc.
    qualite = Column(String(20), default='good')  # good, suspect, bad

    def __repr__(self):
        return f"<MesureEnergie(id={self.id}, ts={self.timestamp}, p={self.puissance_active_totale}W)>"

class EvenementCompteur(Base):
    """Modèle pour les événements du compteur"""
    __tablename__ = 'evenements_compteur'

    id = Column(Integer, primary_key=True, autoincrement=True)
    timestamp = Column(DateTime, default=datetime.utcnow, index=True)

    type_evenement = Column(String(50), nullable=False)  # coupure, surtension, etc.
    severite = Column(String(20), default='info')  # info, warning, error, critical

    description = Column(Text, nullable=True)
    valeur_mesuree = Column(Float, nullable=True)  # Valeur au moment de l'événement
    unite = Column(String(10), nullable=True)  # V, A, W, etc.

    # Contexte
    code_obis = Column(String(20), nullable=True)  # Code OBIS concerné
    seuil_declencheur = Column(Float, nullable=True)  # Seuil dépassé

    def __repr__(self):
        return f"<EvenementCompteur(id={self.id}, type={self.type_evenement}, ts={self.timestamp})>"

class ConfigurationCompteur(Base):
    """Modèle pour la configuration du compteur"""
    __tablename__ = 'configuration_compteur'

    id = Column(Integer, primary_key=True, autoincrement=True)
    timestamp_modification = Column(DateTime, default=datetime.utcnow)

    parametre = Column(String(100), nullable=False, unique=True)
    valeur = Column(String(255), nullable=False)
    type_valeur = Column(String(20), default='string')  # string, int, float, bool

    description = Column(Text, nullable=True)

class BaseDeDonnees:
    """Gestionnaire de base de données"""

    def __init__(self, chemin_db='compteur_e450.db'):
        self.chemin_db = chemin_db
        self.engine = None
        self.SessionLocal = None

    def initialiser(self):
        """Initialisation de la base de données"""
        # Création du répertoire si nécessaire
        os.makedirs(os.path.dirname(self.chemin_db), exist_ok=True)

        # URL de connexion SQLite
        database_url = f'sqlite:///{self.chemin_db}'

        # Création du moteur
        self.engine = create_engine(
            database_url,
            connect_args={'check_same_thread': False},  # Pour SQLite
            echo=False  # True pour debug SQL
        )

        # Création des tables
        Base.metadata.create_all(bind=self.engine)

        # Session factory
        self.SessionLocal = sessionmaker(autocommit=False, autoflush=False, bind=self.engine)

        print(f"Base de données initialisée : {self.chemin_db}")

    def obtenir_session(self):
        """Obtenir une session de base de données"""
        return self.SessionLocal()

    def sauvegarder_mesure(self, mesure_data):
        """Sauvegarde d'une mesure énergétique"""
        session = self.obtenir_session()

        try:
            # Création de l'objet MesureEnergie
            mesure = MesureEnergie(**mesure_data)

            # Ajout à la session
            session.add(mesure)
            session.commit()

            # Refresh pour obtenir l'ID
            session.refresh(mesure)

            print(f"Mesure sauvegardée (ID: {mesure.id})")
            return mesure.id

        except Exception as e:
            session.rollback()
            print(f"Erreur sauvegarde mesure : {e}")
            raise
        finally:
            session.close()

    def sauvegarder_evenement(self, type_evt, description, **kwargs):
        """Sauvegarde d'un événement"""
        session = self.obtenir_session()

        try:
            evenement = EvenementCompteur(
                type_evenement=type_evt,
                description=description,
                **kwargs
            )

            session.add(evenement)
            session.commit()
            session.refresh(evenement)

            print(f"Événement sauvegardé : {type_evt}")
            return evenement.id

        except Exception as e:
            session.rollback()
            print(f"Erreur sauvegarde événement : {e}")
            raise
        finally:
            session.close()

    def recuperer_mesures_periode(self, debut, fin, limit=1000):
        """Récupération des mesures sur une période"""
        session = self.obtenir_session()

        try:
            mesures = session.query(MesureEnergie)\
                .filter(MesureEnergie.timestamp.between(debut, fin))\
                .order_by(MesureEnergie.timestamp)\
                .limit(limit)\
                .all()

            return mesures

        finally:
            session.close()

    def obtenir_statistiques(self):
        """Statistiques générales de la base"""
        session = self.obtenir_session()

        try:
            # Nombre total de mesures
            nb_mesures = session.query(MesureEnergie).count()

            # Période couverte
            premiere = session.query(MesureEnergie).order_by(MesureEnergie.timestamp).first()
            derniere = session.query(MesureEnergie).order_by(MesureEnergie.timestamp.desc()).first()

            # Nombre d'événements
            nb_evenements = session.query(EvenementCompteur).count()

            return {
                'nb_mesures': nb_mesures,
                'date_premiere': premiere.timestamp if premiere else None,
                'date_derniere': derniere.timestamp if derniere else None,
                'nb_evenements': nb_evenements,
                'periode_jours': (derniere.timestamp - premiere.timestamp).days if premiere and derniere else 0
            }

        finally:
            session.close()
```

### Utilisation de la base de données

```python
def exemple_utilisation_base():
    """Exemple complet d'utilisation de la base de données"""

    # Initialisation
    db = BaseDeDonnees('data/compteur_e450.db')
    db.initialiser()

    # Simulation de données du compteur
    donnees_compteur = {
        'energie_active_import': 154.32,
        'puissance_active_totale': 2345.0,
        'tension_l1': 231.5,
        'tension_l2': 232.1,
        'tension_l3': 230.8,
        'courant_l1': 12.5,
        'courant_l2': 11.8,
        'courant_l3': 13.2,
        'frequence': 50.0,
        'facteur_puissance': 0.985
    }

    # Sauvegarde d'une mesure
    id_mesure = db.sauvegarder_mesure(donnees_compteur)
    print(f"Mesure sauvegardée avec ID : {id_mesure}")

    # Sauvegarde d'un événement
    db.sauvegarder_evenement(
        type_evenement='pic_puissance',
        description='Pic de puissance détecté',
        severite='warning',
        valeur_mesuree=4500.0,
        unite='W',
        seuil_declencheur=4000.0
    )

    # Récupération des dernières mesures
    from datetime import datetime, timedelta

    debut = datetime.now() - timedelta(hours=24)
    fin = datetime.now()

    mesures = db.recuperer_mesures_periode(debut, fin)

    print(f"\nMesures des dernières 24h : {len(mesures)}")
    for mesure in mesures[-5:]:  # 5 dernières
        print(f"  {mesure.timestamp}: {mesure.puissance_active_totale}W")

    # Statistiques
    stats = db.obtenir_statistiques()
    print(f"\nStatistiques base de données :")
    print(f"  Mesures totales : {stats['nb_mesures']}")
    print(f"  Événements : {stats['nb_evenements']}")
    print(f"  Période : {stats['periode_jours']} jours")

if __name__ == "__main__":
    exemple_utilisation_base()
```

## 📄 Sérialisation JSON

### Avantages du JSON

- ✅ **Humain lisible** : Facile debug
- ✅ **Flexible** : Structure dynamique
- ✅ **Universel** : Supporté partout
- ✅ **Léger** : Overhead minimal
- ✅ **Versionnable** : Évolution facile

### Structure des fichiers JSON

#### Format par mesure

```json
{
  "metadata": {
    "version": "1.0",
    "compteur_id": "E450001234",
    "creation": "2025-11-07T14:30:00Z"
  },
  "mesures": [
    {
      "timestamp": "2025-11-07T14:30:00Z",
      "valeurs": {
        "energie_active_import": 154.32,
        "puissance_active_totale": 2345.0,
        "tension_l1": 231.5,
        "courant_l1": 12.5
      },
      "qualite": "good",
      "source": "port_optique"
    }
  ]
}
```

#### Format historique consolidé

```json
{
  "compteur": "E450001234",
  "periode": {
    "debut": "2025-01-01T00:00:00Z",
    "fin": "2025-12-31T23:59:59Z"
  },
  "series": {
    "puissance_active": {
      "unite": "W",
      "valeurs": [
        ["2025-01-01T00:00:00Z", 1200],
        ["2025-01-01T00:15:00Z", 1350],
        ["2025-01-01T00:30:00Z", 1180]
      ]
    },
    "tension_l1": {
      "unite": "V",
      "valeurs": [
        ["2025-01-01T00:00:00Z", 231.5],
        ["2025-01-01T00:15:00Z", 232.1],
        ["2025-01-01T00:30:00Z", 231.8]
      ]
    }
  }
}
```

### Classe de gestion JSON

```python
import json
import os
from datetime import datetime, timedelta
from typing import Dict, List, Any
import gzip

class StockageJSON:
    """Gestionnaire de stockage JSON avec compression"""

    def __init__(self, repertoire_base='data_json'):
        self.repertoire_base = repertoire_base
        self.creer_repertoires()

    def creer_repertoires(self):
        """Création de l'arborescence de répertoires"""
        os.makedirs(self.repertoire_base, exist_ok=True)
        os.makedirs(f"{self.repertoire_base}/quotidien", exist_ok=True)
        os.makedirs(f"{self.repertoire_base}/hebdomadaire", exist_ok=True)
        os.makedirs(f"{self.repertoire_base}/mensuel", exist_ok=True)

    def sauvegarder_mesure(self, mesure_data: Dict[str, Any], compresser=True):
        """Sauvegarde d'une mesure au format JSON"""

        # Ajout de métadonnées
        mesure_complete = {
            'timestamp': datetime.utcnow().isoformat() + 'Z',
            'donnees': mesure_data,
            'metadata': {
                'version': '1.0',
                'source': 'compteur_e450'
            }
        }

        # Nom du fichier (par heure)
        maintenant = datetime.utcnow()
        nom_fichier = f"mesure_{maintenant.strftime('%Y%m%d_%H')}.json"

        if compresser:
            nom_fichier += '.gz'

        chemin_fichier = os.path.join(self.repertoire_base, 'quotidien', nom_fichier)

        # Sauvegarde
        self._ecrire_json(chemin_fichier, mesure_complete, compresser)

    def _ecrire_json(self, chemin: str, data: Dict, compresser=True):
        """Écriture du fichier JSON (compressé ou non)"""

        if compresser:
            with gzip.open(chemin, 'wt', encoding='utf-8') as f:
                json.dump(data, f, indent=2, ensure_ascii=False)
        else:
            with open(chemin, 'w', encoding='utf-8') as f:
                json.dump(data, f, indent=2, ensure_ascii=False)

    def consolider_jour(self, date=None):
        """Consolidation des mesures d'une journée"""

        if date is None:
            date = datetime.utcnow().date()

        # Recherche des fichiers horaires
        pattern = f"mesure_{date.strftime('%Y%m%d')}_*.json*"
        fichiers_horaires = glob.glob(os.path.join(self.repertoire_base, 'quotidien', pattern))

        if not fichiers_horaires:
            return False

        # Consolidation
        mesures_consolidees = []
        for fichier in sorted(fichiers_horaires):
            mesures = self._lire_json(fichier)
            if isinstance(mesures, list):
                mesures_consolidees.extend(mesures)
            else:
                mesures_consolidees.append(mesures)

        # Sauvegarde du fichier consolidé
        nom_consolide = f"jour_{date.strftime('%Y%m%d')}.json.gz"
        chemin_consolide = os.path.join(self.repertoire_base, 'quotidien', nom_consolide)

        data_consolidee = {
            'date': date.isoformat(),
            'nb_mesures': len(mesures_consolidees),
            'mesures': mesures_consolidees
        }

        self._ecrire_json(chemin_consolide, data_consolidee, compresser=True)

        # Nettoyage des fichiers horaires (optionnel)
        # for fichier in fichiers_horaires:
        #     os.remove(fichier)

        return True

    def _lire_json(self, chemin: str) -> Dict:
        """Lecture d'un fichier JSON (compressé ou non)"""

        try:
            if chemin.endswith('.gz'):
                with gzip.open(chemin, 'rt', encoding='utf-8') as f:
                    return json.load(f)
            else:
                with open(chemin, 'r', encoding='utf-8') as f:
                    return json.load(f)
        except Exception as e:
            print(f"Erreur lecture {chemin} : {e}")
            return {}

    def rechercher_mesures(self, debut: datetime, fin: datetime) -> List[Dict]:
        """Recherche de mesures dans une période"""

        mesures = []

        # Recherche dans les fichiers consolidés
        date_courante = debut.date()
        while date_courante <= fin.date():
            nom_fichier = f"jour_{date_courante.strftime('%Y%m%d')}.json.gz"
            chemin_fichier = os.path.join(self.repertoire_base, 'quotidien', nom_fichier)

            if os.path.exists(chemin_fichier):
                data_jour = self._lire_json(chemin_fichier)

                if 'mesures' in data_jour:
                    for mesure in data_jour['mesures']:
                        timestamp = datetime.fromisoformat(mesure['timestamp'][:-1])  # Enlever 'Z'

                        if debut <= timestamp <= fin:
                            mesures.append(mesure)

            date_courante += timedelta(days=1)

        return mesures

    def obtenir_statistiques_json(self) -> Dict:
        """Statistiques du stockage JSON"""

        stats = {
            'fichiers_quotidiens': 0,
            'taille_totale_mo': 0,
            'periode_couverte': None,
            'nb_mesures_total': 0
        }

        # Analyse du répertoire quotidien
        repertoire_quotidien = os.path.join(self.repertoire_base, 'quotidien')

        if os.path.exists(repertoire_quotidien):
            total_size = 0
            dates = []

            for fichier in os.listdir(repertoire_quotidien):
                if fichier.startswith('jour_') and fichier.endswith('.json.gz'):
                    chemin = os.path.join(repertoire_quotidien, fichier)
                    total_size += os.path.getsize(chemin)
                    stats['fichiers_quotidiens'] += 1

                    # Extraction de la date
                    date_str = fichier[5:13]  # jour_YYYYMMDD
                    dates.append(datetime.strptime(date_str, '%Y%m%d').date())

            stats['taille_totale_mo'] = round(total_size / (1024*1024), 2)

            if dates:
                stats['periode_couverte'] = {
                    'debut': min(dates).isoformat(),
                    'fin': max(dates).isoformat(),
                    'jours': (max(dates) - min(dates)).days + 1
                }

        return stats
```

## 🌐 API REST /api/data

### Endpoints de l'API

```python
from flask import Flask, jsonify, request
from datetime import datetime, timedelta

app = Flask(__name__)

# Instance de la base de données
db = BaseDeDonnees('compteur_e450.db')
db.initialiser()

@app.route('/api/data')
def get_data():
    """Récupération des données avec filtres"""

    # Paramètres de requête
    limit = int(request.args.get('limit', 100))
    heures = int(request.args.get('hours', 24))
    metriques = request.args.get('metrics', '').split(',') if request.args.get('metrics') else None

    # Calcul de la période
    debut = datetime.utcnow() - timedelta(hours=heures)

    # Récupération des données
    mesures = db.recuperer_mesures_periode(debut, datetime.utcnow(), limit)

    # Formatage de la réponse
    data_response = {
        'periode': {
            'debut': debut.isoformat() + 'Z',
            'fin': datetime.utcnow().isoformat() + 'Z',
            'heures': heures
        },
        'nb_mesures': len(mesures),
        'mesures': []
    }

    for mesure in mesures:
        mesure_dict = {
            'timestamp': mesure.timestamp.isoformat() + 'Z',
            'puissance_active': mesure.puissance_active_totale,
            'tensions': {
                'l1': mesure.tension_l1,
                'l2': mesure.tension_l2,
                'l3': mesure.tension_l3
            },
            'courants': {
                'l1': mesure.courant_l1,
                'l2': mesure.courant_l2,
                'l3': mesure.courant_l3
            },
            'frequence': mesure.frequence,
            'facteur_puissance': mesure.facteur_puissance
        }

        # Filtrage des métriques si demandé
        if metriques:
            mesure_dict = {k: v for k, v in mesure_dict.items() if k in metriques}

        data_response['mesures'].append(mesure_dict)

    return jsonify(data_response)

@app.route('/api/events')
def get_events():
    """Récupération des événements"""

    severite = request.args.get('severity')
    type_evt = request.args.get('type')
    limit = int(request.args.get('limit', 50))

    session = db.obtenir_session()

    try:
        query = session.query(EvenementCompteur).order_by(EvenementCompteur.timestamp.desc())

        if severite:
            query = query.filter(EvenementCompteur.severite == severite)

        if type_evt:
            query = query.filter(EvenementCompteur.type_evenement == type_evt)

        evenements = query.limit(limit).all()

        return jsonify({
            'nb_evenements': len(evenements),
            'evenements': [{
                'id': evt.id,
                'timestamp': evt.timestamp.isoformat() + 'Z',
                'type': evt.type_evenement,
                'severite': evt.severite,
                'description': evt.description,
                'valeur': evt.valeur_mesuree,
                'unite': evt.unite
            } for evt in evenements]
        })

    finally:
        session.close()

@app.route('/api/stats')
def get_stats():
    """Statistiques générales"""

    stats_db = db.obtenir_statistiques()

    # Ajout d'autres statistiques
    stats_db.update({
        'timestamp': datetime.utcnow().isoformat() + 'Z',
        'status': 'online'
    })

    return jsonify(stats_db)

@app.route('/api/data/current')
def get_current_data():
    """Dernière mesure disponible"""

    session = db.obtenir_session()

    try:
        mesure = session.query(MesureEnergie)\
            .order_by(MesureEnergie.timestamp.desc())\
            .first()

        if mesure:
            return jsonify({
                'timestamp': mesure.timestamp.isoformat() + 'Z',
                'puissance_active': mesure.puissance_active_totale,
                'energies': {
                    'active_import': mesure.energie_active_import,
                    'active_export': mesure.energie_active_export
                },
                'tensions': [mesure.tension_l1, mesure.tension_l2, mesure.tension_l3],
                'courants': [mesure.courant_l1, mesure.courant_l2, mesure.courant_l3]
            })
        else:
            return jsonify({'error': 'Aucune donnée disponible'}), 404

    finally:
        session.close()

if __name__ == "__main__":
    app.run(debug=True, host='0.0.0.0', port=5000)
```

### Test de l'API

```bash
# Dernière mesure
curl http://localhost:5000/api/data/current

# Mesures des dernières 6 heures
curl "http://localhost:5000/api/data?hours=6"

# Événements critiques
curl "http://localhost:5000/api/events?severity=critical"

# Statistiques
curl http://localhost:5000/api/stats
```

## 🔄 Stratégies hybrides

### SQLite + JSON

**Avantages** :
- SQLite pour les requêtes complexes
- JSON pour l'archivage et l'échange
- Sauvegarde automatique JSON depuis SQLite

```python
def exporter_vers_json(db, periode_jours=30):
    """Export périodique des données SQLite vers JSON"""

    debut = datetime.utcnow() - timedelta(days=periode_jours)
    mesures = db.recuperer_mesures_periode(debut, datetime.utcnow())

    # Regroupement par jour
    mesures_par_jour = {}
    for mesure in mesures:
        jour = mesure.timestamp.date().isoformat()
        if jour not in mesures_par_jour:
            mesures_par_jour[jour] = []

        mesures_par_jour[jour].append({
            'timestamp': mesure.timestamp.isoformat(),
            'puissance': mesure.puissance_active_totale,
            'tension_l1': mesure.tension_l1,
            'courant_l1': mesure.courant_l1
        })

    # Sauvegarde JSON
    stockage_json = StockageJSON()
    for jour, mesures_jour in mesures_par_jour.items():
        nom_fichier = f"export_{jour}.json.gz"
        chemin = os.path.join(stockage_json.repertoire_base, 'mensuel', nom_fichier)

        data_export = {
            'date': jour,
            'nb_mesures': len(mesures_jour),
            'mesures': mesures_jour
        }

        stockage_json._ecrire_json(chemin, data_export, compresser=True)
```

### Migration et sauvegarde

```python
def strategie_sauvegarde_hybride(donnees_compteur):
    """Stratégie de sauvegarde hybride"""

    # 1. Sauvegarde en base de données (requêtes)
    id_mesure = db.sauvegarder_mesure(donnees_compteur)

    # 2. Sauvegarde JSON compressé (archivage)
    stockage_json = StockageJSON()
    stockage_json.sauvegarder_mesure(donnees_compteur)

    # 3. Consolidation périodique (maintenance)
    if datetime.utcnow().hour == 2:  # À 2h du matin
        stockage_json.consolider_jour()

    return id_mesure
```

> **💡 À retenir** : Le stockage hybride **SQLite + JSON** offre le meilleur compromis entre performance des requêtes et fiabilité de l'archivage.

> **⚠️ Astuce** : Implémentez une **stratégie de rotation** des fichiers pour éviter la saturation du disque (suppression automatique des données anciennes selon politique de rétention).

Félicitations ! La Partie II sur la lecture et l'acquisition des données est maintenant complète. Vous maîtrisez les protocoles de communication, les codes OBIS, et les stratégies de stockage. La Partie III va maintenant vous guider dans le développement de l'application Flask !

---

**Navigation**
- [Chapitre précédent : Structure des données OBIS](Chapitre_8_Structure_OBIS.md)
- [Partie III : Développement de l'application Flask](../Partie_III_Flask_Application/)
- [Retour à la table des matières](../../README.md)
