# 🔌 Chapitre 16 : Webhook vers InfluxDB + Grafana

## 📊 Installation rapide d'InfluxDB et Grafana

InfluxDB et Grafana forment la stack TICK (Telegraf, InfluxDB, Chronograf, Kapacitor) parfaite pour le stockage et la visualisation de séries temporelles énergétiques.

### Installation d'InfluxDB

#### Sur Debian/Ubuntu

```bash
# Ajout du repository InfluxData
wget -qO- https://repos.influxdata.com/influxdb.key | sudo apt-key add -
echo "deb https://repos.influxdata.com/debian $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/influxdb.list

# Installation
sudo apt update
sudo apt install influxdb

# Démarrage du service
sudo systemctl start influxdb
sudo systemctl enable influxdb

# Vérification
curl -sL -I http://localhost:8086/ping
```

#### Configuration initiale

```bash
# Connexion à l'interface CLI
influx

# Création de la base de données
CREATE DATABASE compteur_e450;

# Création d'un utilisateur
CREATE USER compteur WITH PASSWORD 'mot_de_passe_compteur';

# Attribution des droits
GRANT ALL ON compteur_e450 TO compteur;

# Création d'un token d'API (InfluxDB 2.x)
# Depuis l'interface web http://localhost:8086
```

#### Avec Docker

```yaml
# docker-compose.yml
version: '3.8'
services:
  influxdb:
    image: influxdb:2.7
    ports:
      - "8086:8086"
    volumes:
      - influxdb_data:/var/lib/influxdb2
      - influxdb_config:/etc/influxdb2
    environment:
      - DOCKER_INFLUXDB_INIT_MODE=setup
      - DOCKER_INFLUXDB_INIT_USERNAME=admin
      - DOCKER_INFLUXDB_INIT_PASSWORD=admin123
      - DOCKER_INFLUXDB_INIT_ORG=compteur_e450
      - DOCKER_INFLUXDB_INIT_BUCKET=energie
      - DOCKER_INFLUXDB_INIT_ADMIN_TOKEN=mon_token_admin
    restart: unless-stopped

volumes:
  influxdb_data:
  influxdb_config:
```

### Installation de Grafana

#### Sur Debian/Ubuntu

```bash
# Ajout du repository Grafana
wget -q -O - https://packages.grafana.com/gpg.key | sudo apt-key add -
echo "deb https://packages.grafana.com/oss/deb stable main" | sudo tee /etc/apt/sources.list.d/grafana.list

# Installation
sudo apt update
sudo apt install grafana

# Démarrage
sudo systemctl start grafana-server
sudo systemctl enable grafana-server

# Interface web : http://localhost:3000 (admin/admin)
```

#### Avec Docker

```yaml
# docker-compose.yml (extension)
services:
  grafana:
    image: grafana/grafana:latest
    ports:
      - "3000:3000"
    volumes:
      - grafana_data:/var/lib/grafana
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin
      - GF_USERS_ALLOW_SIGN_UP=false
    depends_on:
      - influxdb
    restart: unless-stopped

volumes:
  grafana_data:
```

### Configuration de la source de données

#### Connexion Grafana → InfluxDB

1. **Accès à Grafana** : http://localhost:3000
2. **Login** : admin / admin (changer le mot de passe)
3. **Menu** : Configuration → Data Sources → Add data source
4. **Type** : InfluxDB
5. **Configuration** :
   - **URL** : http://influxdb:8086 (ou localhost:8086)
   - **Database** : compteur_e450
   - **User** : compteur
   - **Password** : mot_de_passe_compteur
6. **Save & Test**

## 🌐 Webhook Flask pour stockage temps réel

### Création du webhook

```python
# app/routes/webhooks.py
from flask import Blueprint, request, jsonify
from app.models import MesureEnergie
from app.extensions import db
from app.services import detecteur_anomalies
import logging

webhooks_bp = Blueprint('webhooks', __name__)
logger = logging.getLogger(__name__)

@webhooks_bp.route('/webhook/influxdb', methods=['POST'])
def webhook_influxdb():
    """
    Webhook pour réception des données depuis InfluxDB
    Utile pour les requêtes en temps réel ou les alertes
    """
    try:
        data = request.get_json()

        if not data:
            return jsonify({'error': 'No data provided'}), 400

        # Traitement des données InfluxDB
        # Format typique des webhooks InfluxDB
        measurements = data.get('measurements', [])

        processed_count = 0
        for measurement in measurements:
            # Conversion du format InfluxDB vers notre format
            mesure_data = convertir_mesure_influxdb(measurement)

            if mesure_data:
                # Sauvegarde en base
                mesure = MesureEnergie(**mesure_data)
                db.session.add(mesure)
                processed_count += 1

        db.session.commit()

        logger.info(f"Webhook InfluxDB: {processed_count} mesures traitées")
        return jsonify({
            'status': 'success',
            'processed': processed_count
        })

    except Exception as e:
        logger.error(f"Erreur webhook InfluxDB: {e}")
        db.session.rollback()
        return jsonify({'error': str(e)}), 500

@webhooks_bp.route('/webhook/energie', methods=['POST'])
def webhook_energie():
    """
    Webhook générique pour données énergétiques
    Accepte différents formats (JSON, etc.)
    """
    try:
        # Support de différents content-types
        if request.is_json:
            data = request.get_json()
        else:
            # Tentative de parsing manuel
            data = request.get_data(as_text=True)
            try:
                import json
                data = json.loads(data)
            except:
                return jsonify({'error': 'Format non supporté'}), 400

        # Validation des données
        validation_errors = valider_donnees_energie(data)
        if validation_errors:
            return jsonify({
                'error': 'Données invalides',
                'details': validation_errors
            }), 400

        # Conversion et sauvegarde
        mesure_data = convertir_donnees_energie(data)
        mesure = MesureEnergie(**mesure_data)

        # Détection d'anomalies
        anomalies = detecteur_anomalies.detecter_anomalie_mesure(mesure)
        if anomalies:
            mesure.qualite = 'suspect'  # Marquer comme suspect

        db.session.add(mesure)
        db.session.commit()

        # Réponse avec informations supplémentaires
        response = {
            'status': 'success',
            'id': mesure.id,
            'anomalies_detectees': len(anomalies) if anomalies else 0
        }

        if anomalies:
            response['anomalies'] = anomalies

        logger.info(f"Webhook énergie: mesure {mesure.id} sauvegardée")
        return jsonify(response)

    except Exception as e:
        logger.error(f"Erreur webhook énergie: {e}")
        db.session.rollback()
        return jsonify({'error': str(e)}), 500

def convertir_mesure_influxdb(measurement):
    """Conversion d'une mesure InfluxDB vers notre format"""
    try:
        # Structure typique d'une mesure InfluxDB
        fields = measurement.get('fields', {})
        tags = measurement.get('tags', {})

        mesure_data = {
            'timestamp': measurement.get('timestamp'),
            'source': 'influxdb_webhook',
            'qualite': 'good',
            'compteur_id': tags.get('compteur_id', 'E450001234'),
        }

        # Mapping des champs
        field_mapping = {
            'puissance_active': 'puissance_active_totale',
            'tension_l1': 'tension_l1',
            'tension_l2': 'tension_l2',
            'tension_l3': 'tension_l3',
            'courant_l1': 'courant_l1',
            'courant_l2': 'courant_l2',
            'courant_l3': 'courant_l3',
            'frequence': 'frequence',
            'facteur_puissance': 'facteur_puissance',
            'energie_import': 'energie_active_import',
            'energie_export': 'energie_active_export',
        }

        for influx_field, db_field in field_mapping.items():
            if influx_field in fields:
                mesure_data[db_field] = fields[influx_field]

        return mesure_data

    except Exception as e:
        logger.error(f"Erreur conversion InfluxDB: {e}")
        return None

def convertir_donnees_energie(data):
    """Conversion générique des données énergétiques"""
    from datetime import datetime

    mesure_data = {
        'source': data.get('source', 'webhook'),
        'qualite': data.get('qualite', 'good'),
        'compteur_id': data.get('compteur_id', 'E450001234'),
        'timestamp': data.get('timestamp', datetime.utcnow()),
    }

    # Mapping direct des champs connus
    direct_fields = [
        'puissance_active_totale', 'tension_l1', 'tension_l2', 'tension_l3',
        'courant_l1', 'courant_l2', 'courant_l3', 'courant_neutre',
        'frequence', 'facteur_puissance', 'thd_tension', 'thd_courant',
        'energie_active_import', 'energie_active_export',
        'energie_reactive_import', 'energie_reactive_export'
    ]

    for field in direct_fields:
        if field in data:
            mesure_data[field] = data[field]

    return mesure_data

def valider_donnees_energie(data):
    """Validation des données énergétiques reçues"""
    errors = []

    # Champs obligatoires
    required_fields = ['puissance_active_totale']

    for field in required_fields:
        if field not in data:
            errors.append(f"Champ obligatoire manquant: {field}")

    # Validation des types et plages
    validations = {
        'puissance_active_totale': {'type': (int, float), 'min': 0, 'max': 50000},
        'tension_l1': {'type': (int, float), 'min': 180, 'max': 280},
        'tension_l2': {'type': (int, float), 'min': 180, 'max': 280},
        'tension_l3': {'type': (int, float), 'min': 180, 'max': 280},
        'courant_l1': {'type': (int, float), 'min': 0, 'max': 100},
        'courant_l2': {'type': (int, float), 'min': 0, 'max': 100},
        'courant_l3': {'type': (int, float), 'min': 0, 'max': 100},
        'frequence': {'type': (int, float), 'min': 45, 'max': 55},
        'facteur_puissance': {'type': (int, float), 'min': 0, 'max': 1},
    }

    for field, rules in validations.items():
        if field in data:
            value = data[field]

            # Type
            if not isinstance(value, rules['type']):
                errors.append(f"{field}: type invalide (attendu {rules['type']})")

            # Plage
            if 'min' in rules and value < rules['min']:
                errors.append(f"{field}: valeur trop faible (min {rules['min']})")
            if 'max' in rules and value > rules['max']:
                errors.append(f"{field}: valeur trop élevée (max {rules['max']})")

    return errors
```

### Enregistrement du blueprint

```python
# app/routes/__init__.py
from .webhooks import webhooks_bp

def register_blueprints(app):
    # ... autres blueprints ...
    app.register_blueprint(webhooks_bp, url_prefix='/api')
```

## 📈 Création de dashboards visuels interactifs

### Dashboard énergétique principal

#### Création du dashboard

1. **Accès Grafana** : http://localhost:3000
2. **Menu** : + → Dashboard → Add new panel
3. **Configuration des panels** :

#### Panel 1 : Puissance actuelle

```json
{
  "title": "Puissance Active (Temps Réel)",
  "type": "stat",
  "targets": [
    {
      "query": "SELECT last(\"puissance_active_totale\") FROM \"mesures_energie\" WHERE time >= now() - 1h",
      "rawQuery": true
    }
  ],
  "fieldConfig": {
    "defaults": {
      "unit": "watt",
      "color": {
        "mode": "thresholds"
      },
      "thresholds": {
        "mode": "absolute",
        "steps": [
          { "color": "green", "value": null },
          { "color": "orange", "value": 3000 },
          { "color": "red", "value": 4500 }
        ]
      }
    }
  }
}
```

#### Panel 2 : Graphique de tendance

```json
{
  "title": "Évolution de la Puissance",
  "type": "graph",
  "targets": [
    {
      "query": "SELECT mean(\"puissance_active_totale\") FROM \"mesures_energie\" WHERE time >= now() - 24h GROUP BY time(15m)",
      "rawQuery": true
    }
  ],
  "xaxis": {
    "mode": "time"
  },
  "yaxes": [
    {
      "unit": "watt",
      "label": "Puissance"
    }
  ]
}
```

#### Panel 3 : Tensions par phase

```json
{
  "title": "Tensions par Phase",
  "type": "graph",
  "targets": [
    {
      "query": "SELECT mean(\"tension_l1\") AS \"L1\", mean(\"tension_l2\") AS \"L2\", mean(\"tension_l3\") AS \"L3\" FROM \"mesures_energie\" WHERE time >= now() - 1h GROUP BY time(5m)",
      "rawQuery": true
    }
  ],
  "seriesOverrides": [
    { "alias": "L1", "color": "#56B4E9" },
    { "alias": "L2", "color": "#009E73" },
    { "alias": "L3", "color": "#E69F00" }
  ]
}
```

#### Panel 4 : Consommation journalière

```json
{
  "title": "Consommation Journalière",
  "type": "bargauge",
  "targets": [
    {
      "query": "SELECT integral(\"puissance_active_totale\") / 3600000 AS \"kWh\" FROM \"mesures_energie\" WHERE time >= now() - 24h",
      "rawQuery": true
    }
  ],
  "fieldConfig": {
    "defaults": {
      "unit": "kwh",
      "min": 0,
      "max": 50
    }
  }
}
```

### Dashboard d'analyse avancée

#### Panel : Analyse spectrale (harmoniques)

```json
{
  "title": "Analyse des Harmoniques",
  "type": "table",
  "targets": [
    {
      "query": "SELECT mean(\"thd_tension\") AS \"THD Tension\", mean(\"thd_courant\") AS \"THD Courant\" FROM \"mesures_energie\" WHERE time >= now() - 1h",
      "rawQuery": true
    }
  ],
  "fieldConfig": {
    "defaults": {
      "unit": "percent",
      "thresholds": {
        "mode": "absolute",
        "steps": [
          { "color": "green", "value": null },
          { "color": "orange", "value": 5 },
          { "color": "red", "value": 8 }
        ]
      }
    }
  }
}
```

#### Panel : Efficacité énergétique

```sql
-- Requête pour calculer l'efficacité
SELECT 
  (integral("energie_active_import") / (integral("energie_active_import") + integral("energie_reactive_import"))) * 100 AS "Efficacité (%)"
FROM "mesures_energie" 
WHERE time >= now() - 24h
```

### Alertes Grafana

#### Configuration d'alertes

1. **Panel** : Clic droit → Edit → Alert
2. **Conditions** :
   - **Query A** : `SELECT last("puissance_active_totale") FROM "mesures_energie"`
   - **Condition** : `WHEN last() ABOVE 4500`
   - **For** : `5m`
3. **Notifications** :
   - **Type** : Webhook
   - **URL** : `http://votre-app:5000/api/webhook/alerte`
   - **Body** : `{"alerte": "puissance_elevée", "valeur": {{.Value}}}`

### Template de dashboard JSON

```json
{
  "dashboard": {
    "title": "Compteur E450 - Monitoring Énergétique",
    "tags": ["energie", "compteur", "e450"],
    "timezone": "browser",
    "panels": [
      // Panels définis ci-dessus
    ],
    "time": {
      "from": "now-24h",
      "to": "now"
    },
    "refresh": "30s"
  }
}
```

## 📤 Service d'export vers InfluxDB

### Client InfluxDB pour Python

```python
# app/services/influxdb_service.py
from influxdb_client import InfluxDBClient, Point, WritePrecision
from influxdb_client.client.write_api import SYNCHRONOUS
import logging
from datetime import datetime

logger = logging.getLogger(__name__)

class ServiceInfluxDB:
    """Service d'export vers InfluxDB"""

    def __init__(self, config):
        self.url = config.get('url', 'http://localhost:8086')
        self.token = config.get('token')
        self.org = config.get('org', 'compteur_e450')
        self.bucket = config.get('bucket', 'energie')

        self.client = None
        self.write_api = None
        self.query_api = None

    def connecter(self):
        """Connexion à InfluxDB"""
        try:
            self.client = InfluxDBClient(
                url=self.url,
                token=self.token,
                org=self.org
            )

            # APIs
            self.write_api = self.client.write_api(write_options=SYNCHRONOUS)
            self.query_api = self.client.query_api()

            # Test de connexion
            health = self.client.health()
            if health.status == 'pass':
                logger.info(f"Connecté à InfluxDB: {self.url}")
                return True
            else:
                logger.error(f"InfluxDB health check failed: {health}")
                return False

        except Exception as e:
            logger.error(f"Erreur connexion InfluxDB: {e}")
            return False

    def deconnecter(self):
        """Déconnexion"""
        if self.client:
            self.client.close()
            logger.info("Déconnecté d'InfluxDB")

    def exporter_mesure(self, mesure_data):
        """
        Export d'une mesure vers InfluxDB

        Args:
            mesure_data: Dict des données de mesure
        """
        if not self.write_api:
            logger.error("Write API non initialisée")
            return False

        try:
            # Création du point de données
            point = Point("mesures_energie") \
                .tag("compteur_id", mesure_data.get('compteur_id', 'E450001234')) \
                .tag("source", mesure_data.get('source', 'app')) \
                .time(mesure_data.get('timestamp', datetime.utcnow()), WritePrecision.NS)

            # Ajout des champs
            field_mapping = {
                'puissance_active_totale': 'puissance_active_totale',
                'tension_l1': 'tension_l1',
                'tension_l2': 'tension_l2',
                'tension_l3': 'tension_l3',
                'courant_l1': 'courant_l1',
                'courant_l2': 'courant_l2',
                'courant_l3': 'courant_l3',
                'frequence': 'frequence',
                'facteur_puissance': 'facteur_puissance',
                'energie_active_import': 'energie_active_import',
                'energie_active_export': 'energie_active_export',
            }

            for field_app, field_influx in field_mapping.items():
                valeur = mesure_data.get(field_app)
                if valeur is not None:
                    point.field(field_influx, float(valeur))

            # Écriture
            self.write_api.write(self.bucket, self.org, point)

            logger.debug(f"Mesure exportée vers InfluxDB: {len(field_mapping)} champs")
            return True

        except Exception as e:
            logger.error(f"Erreur export InfluxDB: {e}")
            return False

    def requeter_donnees(self, query, params=None):
        """Exécution d'une requête Flux"""
        if not self.query_api:
            return None

        try:
            # Exécution de la requête
            result = self.query_api.query(query, params=params)

            # Conversion en liste de dicts
            records = []
            for table in result:
                for record in table.records:
                    records.append({
                        'timestamp': record.get_time(),
                        'measurement': record.get_measurement(),
                        'fields': record.values,
                        'tags': record.get_tags()
                    })

            return records

        except Exception as e:
            logger.error(f"Erreur requête InfluxDB: {e}")
            return None

    def obtenir_statistiques(self):
        """Statistiques du bucket"""
        query = f'''
        from(bucket: "{self.bucket}")
        |> range(start: -30d)
        |> count()
        |> group()
        '''

        result = self.requeter_donnees(query)

        if result and len(result) > 0:
            total_points = sum(record['fields'].get('_value', 0) for record in result)
            return {'total_points': total_points}
        else:
            return {'total_points': 0}
```

### Intégration dans le service de lecture

```python
# app/services/lecture_compteur.py (extension)
def _sauvegarder_mesure(self, donnees):
    # ... sauvegarde SQLite ...

    # Export vers InfluxDB
    try:
        from app.services import influxdb_service
        if influxdb_service:
            influxdb_service.exporter_mesure(donnees)
    except Exception as e:
        logger.error(f"Erreur export InfluxDB: {e}")

    # ... reste du code ...
```

### Test de l'intégration

```python
# scripts/test_influxdb_integration.py
#!/usr/bin/env python3
"""
Test de l'intégration InfluxDB + Grafana
"""

import requests
import json
from datetime import datetime, timedelta
import time

def tester_influxdb(url="http://localhost:8086", token=None, org="compteur_e450", bucket="energie"):
    """Test de la connexion InfluxDB"""

    print("🔍 Test InfluxDB")
    print("=" * 30)

    # Test de santé
    try:
        response = requests.get(f"{url}/health")
        if response.status_code == 200:
            print("✅ InfluxDB accessible")
        else:
            print(f"❌ InfluxDB non accessible (HTTP {response.status_code})")
            return False
    except Exception as e:
        print(f"❌ Erreur connexion InfluxDB: {e}")
        return False

    # Test d'écriture (si token fourni)
    if token:
        try:
            headers = {
                'Authorization': f'Token {token}',
                'Content-Type': 'text/plain'
            }

            # Données de test
            data = f'mesures_energie,compteur_id=E450_TEST puissance_active_totale=1234.5 {int(time.time() * 1e9)}'

            response = requests.post(
                f"{url}/api/v2/write?org={org}&bucket={bucket}",
                headers=headers,
                data=data
            )

            if response.status_code == 204:
                print("✅ Écriture InfluxDB réussie")
            else:
                print(f"❌ Échec écriture InfluxDB (HTTP {response.status_code})")
                print(f"   Réponse: {response.text}")

        except Exception as e:
            print(f"❌ Erreur écriture InfluxDB: {e}")

    # Test de lecture
    try:
        headers = {
            'Authorization': f'Token {token}',
            'Accept': 'application/csv'
        }

        query = f'''
        from(bucket: "{bucket}")
        |> range(start: -1h)
        |> filter(fn: (r) => r["_measurement"] == "mesures_energie")
        |> limit(n: 1)
        '''

        response = requests.post(
            f"{url}/api/v2/query?org={org}",
            headers=headers,
            data=query
        )

        if response.status_code == 200:
            print("✅ Lecture InfluxDB réussie")
            return True
        else:
            print(f"❌ Échec lecture InfluxDB (HTTP {response.status_code})")
            return False

    except Exception as e:
        print(f"❌ Erreur lecture InfluxDB: {e}")
        return False

def tester_grafana(url="http://localhost:3000", api_key=None):
    """Test de la connexion Grafana"""

    print("\n🔍 Test Grafana")
    print("=" * 30)

    headers = {}
    if api_key:
        headers['Authorization'] = f'Bearer {api_key}'

    # Test d'accès
    try:
        response = requests.get(f"{url}/api/health", headers=headers)
        if response.status_code == 200:
            health = response.json()
            print("✅ Grafana accessible")
            print(f"   Version: {health.get('version', 'N/A')}")
            print(f"   Database: {health.get('database', 'N/A')}")
        else:
            print(f"❌ Grafana non accessible (HTTP {response.status_code})")
            return False
    except Exception as e:
        print(f"❌ Erreur connexion Grafana: {e}")
        return False

    # Test des data sources
    try:
        response = requests.get(f"{url}/api/datasources", headers=headers)
        if response.status_code == 200:
            datasources = response.json()

            influx_found = any(ds['type'] == 'influxdb' for ds in datasources)
            if influx_found:
                print("✅ Data source InfluxDB configurée")
            else:
                print("⚠️  Aucune data source InfluxDB trouvée")
                print("   Configurez InfluxDB dans Grafana")

        return True

    except Exception as e:
        print(f"❌ Erreur test data sources: {e}")
        return False

def tester_webhook_flask(url="http://localhost:5000"):
    """Test du webhook Flask"""

    print("\n🔍 Test Webhook Flask")
    print("=" * 30)

    # Données de test
    test_data = {
        "timestamp": datetime.utcnow().isoformat() + 'Z',
        "source": "test_integration",
        "puissance_active_totale": 2500.0,
        "tension_l1": 232.1,
        "courant_l1": 10.8,
        "frequence": 50.02,
        "facteur_puissance": 0.98
    }

    try:
        response = requests.post(
            f"{url}/api/webhook/energie",
            json=test_data,
            timeout=10
        )

        if response.status_code == 200:
            result = response.json()
            if result.get('status') == 'success':
                print("✅ Webhook Flask fonctionnel")
                print(f"   Mesure créée: ID {result.get('id')}")
                return True
            else:
                print(f"❌ Webhook rejeté: {result}")
                return False
        else:
            print(f"❌ Webhook erreur HTTP {response.status_code}")
            print(f"   Réponse: {response.text}")
            return False

    except Exception as e:
        print(f"❌ Erreur test webhook: {e}")
        return False

def main():
    """Fonction principale"""
    import argparse

    parser = argparse.ArgumentParser(description='Test intégration InfluxDB + Grafana')
    parser.add_argument('--influx-url', default='http://localhost:8086', help='URL InfluxDB')
    parser.add_argument('--influx-token', help='Token InfluxDB')
    parser.add_argument('--influx-org', default='compteur_e450', help='Org InfluxDB')
    parser.add_argument('--influx-bucket', default='energie', help='Bucket InfluxDB')
    parser.add_argument('--grafana-url', default='http://localhost:3000', help='URL Grafana')
    parser.add_argument('--grafana-key', help='Clé API Grafana')
    parser.add_argument('--flask-url', default='http://localhost:5000', help='URL Flask')

    args = parser.parse_args()

    success_count = 0
    total_tests = 3

    # Test InfluxDB
    if tester_influxdb(args.influx_url, args.influx_token, args.influx_org, args.influx_bucket):
        success_count += 1

    # Test Grafana
    if tester_grafana(args.grafana_url, args.grafana_key):
        success_count += 1

    # Test Webhook
    if tester_webhook_flask(args.flask_url):
        success_count += 1

    print(f"\n📊 RÉSULTATS: {success_count}/{total_tests} tests réussis")

    if success_count == total_tests:
        print("🎉 Intégration InfluxDB + Grafana complète!")
        print("Votre stack de monitoring énergétique est opérationnelle.")
    else:
        print("⚠️  Certains composants nécessitent configuration.")
        print("Vérifiez les logs et la documentation pour résoudre les problèmes.")

    return success_count == total_tests

if __name__ == "__main__":
    success = main()
    exit(0 if success else 1)
```

> **💡 À retenir** : InfluxDB + Grafana offre une solution professionnelle de stockage et visualisation de séries temporelles, parfaite pour l'analyse énergétique.

> **⚠️ Astuce** : Commencez par tester chaque composant séparément avant de les connecter ensemble, et utilisez les APIs REST pour déboguer les intégrations.

Dans le prochain chapitre, nous aborderons le calcul automatique du coût énergétique et les intégrations tarifaires !

---

**Navigation**
- [Chapitre précédent : Envoi vers Home Assistant (MQTT)](Chapitre_15_Envoi_Home_Assistant_MQTT.md)
- [Chapitre suivant : Calcul du coût énergétique](Chapitre_17_Calcul_Cout_Energetique.md)
- [Retour à la table des matières](../../README.md)
