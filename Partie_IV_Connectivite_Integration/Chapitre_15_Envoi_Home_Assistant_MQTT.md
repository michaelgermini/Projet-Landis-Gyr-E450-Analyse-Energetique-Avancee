# 🔌 Chapitre 15 : Envoi vers Home Assistant (MQTT)

## 📡 Installation et configuration du broker MQTT

Home Assistant utilise MQTT comme protocole principal pour la communication avec les appareils IoT. Nous allons intégrer notre compteur E450 dans cet écosystème.

### Installation de Mosquitto (broker MQTT)

#### Sur Raspberry Pi / Debian

```bash
# Mise à jour du système
sudo apt update && sudo apt upgrade -y

# Installation de Mosquitto
sudo apt install mosquitto mosquitto-clients

# Installation des clients Python
pip install paho-mqtt

# Démarrage du service
sudo systemctl start mosquitto
sudo systemctl enable mosquitto

# Vérification
sudo systemctl status mosquitto
```

#### Sur Docker

```yaml
# docker-compose.yml pour MQTT
version: '3.8'
services:
  mqtt:
    image: eclipse-mosquitto:2.0
    ports:
      - "1883:1883"
      - "9001:9001"
    volumes:
      - ./mosquitto/config:/mosquitto/config
      - ./mosquitto/data:/mosquitto/data
      - ./mosquitto/log:/mosquitto/log
    restart: unless-stopped
```

```bash
# mosquitto/config/mosquitto.conf
listener 1883
protocol mqtt

# Authentification
allow_anonymous false
password_file /mosquitto/config/passwordfile

# Logging
log_dest file /mosquitto/log/mosquitto.log
log_type all

# Persistence
persistence true
persistence_location /mosquitto/data/
```

#### Configuration de l'authentification

```bash
# Création du fichier de mots de passe
sudo mosquitto_passwd -c /etc/mosquitto/passwordfile compteur_e450
# Saisir le mot de passe quand demandé

# Redémarrage du service
sudo systemctl restart mosquitto
```

### Test du broker MQTT

```python
# scripts/test_mqtt.py
#!/usr/bin/env python3
"""
Test du broker MQTT et connexion
"""

import paho.mqtt.client as mqtt
import time
import json

class TestMQTT:
    """Classe de test MQTT"""

    def __init__(self, broker="localhost", port=1883, username=None, password=None):
        self.broker = broker
        self.port = port
        self.username = username
        self.password = password

        # Client MQTT
        self.client = mqtt.Client()
        self.client.on_connect = self.on_connect
        self.client.on_message = self.on_message
        self.client.on_disconnect = self.on_disconnect

        # Authentification
        if self.username and self.password:
            self.client.username_pw_set(self.username, self.password)

        # Statistiques
        self.connected = False
        self.messages_received = []

    def on_connect(self, client, userdata, flags, rc):
        """Callback de connexion"""
        if rc == 0:
            print(f"✅ Connecté au broker {self.broker}:{self.port}")
            self.connected = True
        else:
            print(f"❌ Échec de connexion, code: {rc}")
            print("Codes d'erreur:")
            print("  1: Connexion refusée - protocole incorrect")
            print("  2: Connexion refusée - identifiant rejeté")
            print("  3: Connexion refusée - serveur indisponible")
            print("  4: Connexion refusée - identifiant ou mot de passe incorrect")
            print("  5: Connexion refusée - non autorisé")

    def on_message(self, client, userdata, message):
        """Callback de réception de message"""
        try:
            payload = message.payload.decode('utf-8')
            print(f"📨 Message reçu sur {message.topic}: {payload}")

            # Tentative de parsing JSON
            try:
                data = json.loads(payload)
                self.messages_received.append({
                    'topic': message.topic,
                    'data': data,
                    'timestamp': time.time()
                })
            except json.JSONDecodeError:
                self.messages_received.append({
                    'topic': message.topic,
                    'data': payload,
                    'timestamp': time.time()
                })

        except Exception as e:
            print(f"❌ Erreur traitement message: {e}")

    def on_disconnect(self, client, userdata, rc):
        """Callback de déconnexion"""
        print(f"📴 Déconnecté du broker, code: {rc}")
        self.connected = False

    def test_connexion(self):
        """Test de connexion simple"""
        print(f"🔍 Test de connexion à {self.broker}:{self.port}")

        try:
            self.client.connect(self.broker, self.port, 60)
            self.client.loop_start()

            # Attente de connexion
            timeout = 5
            start_time = time.time()

            while not self.connected and (time.time() - start_time) < timeout:
                time.sleep(0.1)

            if self.connected:
                print("✅ Test de connexion réussi")
                return True
            else:
                print("❌ Timeout de connexion")
                return False

        except Exception as e:
            print(f"❌ Erreur de connexion: {e}")
            return False

        finally:
            self.client.loop_stop()
            if self.connected:
                self.client.disconnect()

    def test_publication(self, topic="test/compteur", message="Hello MQTT!"):
        """Test de publication de message"""
        print(f"📤 Test de publication sur {topic}")

        if not self.connected:
            if not self.test_connexion():
                return False

        try:
            result = self.client.publish(topic, message, qos=1)
            result.wait_for_publish(timeout=5)

            if result.rc == 0:
                print(f"✅ Message publié avec succès: {message}")
                return True
            else:
                print(f"❌ Échec de publication, code: {result.rc}")
                return False

        except Exception as e:
            print(f"❌ Erreur de publication: {e}")
            return False

    def test_abonnement(self, topic="test/#", timeout=10):
        """Test d'abonnement à un topic"""
        print(f"👂 Test d'abonnement à {topic}")

        if not self.connected:
            if not self.test_connexion():
                return False

        try:
            self.client.subscribe(topic, qos=1)

            print(f"⏳ Écoute pendant {timeout} secondes...")
            start_time = time.time()

            while (time.time() - start_time) < timeout:
                self.client.loop(0.1)

                if self.messages_received:
                    print(f"📨 {len(self.messages_received)} message(s) reçu(s)")
                    for msg in self.messages_received:
                        print(f"  Topic: {msg['topic']}")
                        print(f"  Data: {msg['data']}")
                    break

            if not self.messages_received:
                print("⚠️  Aucun message reçu pendant le timeout")
                print("Conseils:")
                print("- Vérifiez que quelqu'un publie sur ce topic")
                print("- Testez avec un client MQTT externe (MQTT Explorer)")
                print("- Vérifiez les droits d'accès")

            return len(self.messages_received) > 0

        except Exception as e:
            print(f"❌ Erreur d'abonnement: {e}")
            return False

def main():
    """Fonction principale de test"""
    import argparse

    parser = argparse.ArgumentParser(description='Test MQTT pour compteur E450')
    parser.add_argument('--broker', default='localhost', help='Adresse du broker MQTT')
    parser.add_argument('--port', type=int, default=1883, help='Port du broker MQTT')
    parser.add_argument('--username', help='Nom d\'utilisateur MQTT')
    parser.add_argument('--password', help='Mot de passe MQTT')
    parser.add_argument('--test', choices=['connexion', 'publication', 'abonnement', 'all'],
                       default='all', help='Type de test à effectuer')

    args = parser.parse_args()

    # Instance de test
    tester = TestMQTT(args.broker, args.port, args.username, args.password)

    success_count = 0

    print("=" * 60)
    print("🧪 TESTS MQTT POUR COMPTEUR E450")
    print("=" * 60)

    # Test de connexion
    if args.test in ['connexion', 'all']:
        print("\n" + "="*30 + " TEST CONNEXION " + "="*30)
        if tester.test_connexion():
            success_count += 1

    # Test de publication
    if args.test in ['publication', 'all']:
        print("\n" + "="*30 + " TEST PUBLICATION " + "="*30)
        if tester.test_publication("test/compteur/e450", json.dumps({
            "puissance": 2345,
            "tension": 231.5,
            "timestamp": time.time()
        })):
            success_count += 1

    # Test d'abonnement
    if args.test in ['abonnement', 'all']:
        print("\n" + "="*30 + " TEST ABONNEMENT " + "="*30)
        if tester.test_abonnement("test/compteur/#", timeout=5):
            success_count += 1

    print("\n" + "=" * 60)
    print(f"📊 RÉSULTATS: {success_count}/3 tests réussis")

    if success_count == 3:
        print("🎉 Tous les tests MQTT sont passés!")
        print("Votre configuration MQTT est prête pour le compteur E450.")
    else:
        print("⚠️  Certains tests ont échoué.")
        print("Vérifiez votre configuration MQTT avant de continuer.")

    return success_count == 3

if __name__ == "__main__":
    success = main()
    exit(0 if success else 1)
```

#### Tests du broker

```bash
# Test basique
python scripts/test_mqtt.py --test connexion

# Test avec authentification
python scripts/test_mqtt.py --username compteur_e450 --password mon_mot_de_passe --test all

# Test avec broker distant
python scripts/test_mqtt.py --broker 192.168.1.100 --port 1883 --test connexion
```

## 📨 Envoi JSON périodique (paho-mqtt)

### Service MQTT pour le compteur

```python
# app/services/mqtt_service.py
import paho.mqtt.client as mqtt
import json
import logging
from datetime import datetime
from threading import Thread, Event
import time

logger = logging.getLogger(__name__)

class ServiceMQTT:
    """Service MQTT pour publication des données du compteur"""

    def __init__(self, broker_config):
        self.broker = broker_config.get('broker', 'localhost')
        self.port = broker_config.get('port', 1883)
        self.username = broker_config.get('username')
        self.password = broker_config.get('password')
        self.client_id = broker_config.get('client_id', 'compteur_e450')

        # Topics
        self.base_topic = broker_config.get('base_topic', 'homeassistant/sensor/compteur_e450')
        self.state_topic = f"{self.base_topic}/state"
        self.config_topic = f"{self.base_topic}/config"

        # Client MQTT
        self.client = None
        self.connected = Event()
        self.is_running = False
        self.thread_mqtt = None

        # Statistiques
        self.stats = {
            'messages_sent': 0,
            'connection_attempts': 0,
            'last_sent': None,
            'errors': 0
        }

    def demarrer(self):
        """Démarrage du service MQTT"""
        if self.is_running:
            logger.warning("Service MQTT déjà démarré")
            return

        logger.info(f"Démarrage du service MQTT vers {self.broker}:{self.port}")
        self.is_running = True

        # Création du client
        self.client = mqtt.Client(client_id=self.client_id)
        self.client.on_connect = self._on_connect
        self.client.on_disconnect = self._on_disconnect
        self.client.on_publish = self._on_publish

        # Authentification
        if self.username and self.password:
            self.client.username_pw_set(self.username, self.password)

        # Démarrage du thread
        self.thread_mqtt = Thread(target=self._boucle_mqtt, daemon=True)
        self.thread_mqtt.start()

    def arreter(self):
        """Arrêt du service MQTT"""
        logger.info("Arrêt du service MQTT")
        self.is_running = False

        if self.client:
            self.client.disconnect()

        if self.thread_mqtt and self.thread_mqtt.is_alive():
            self.thread_mqtt.join(timeout=5)

    def _boucle_mqtt(self):
        """Boucle principale MQTT"""
        while self.is_running:
            try:
                logger.info(f"Connexion au broker MQTT {self.broker}:{self.port}")
                self.stats['connection_attempts'] += 1

                self.client.connect(self.broker, self.port, keepalive=60)
                self.client.loop_forever()

            except Exception as e:
                logger.error(f"Erreur boucle MQTT: {e}")
                self.stats['errors'] += 1

                if self.is_running:
                    logger.info("Tentative de reconnexion dans 30 secondes...")
                    time.sleep(30)

    def _on_connect(self, client, userdata, flags, rc):
        """Callback de connexion"""
        if rc == 0:
            logger.info("Connecté au broker MQTT")
            self.connected.set()

            # Publication du message de naissance (LWT)
            self.client.publish(f"{self.base_topic}/status", "online", retain=True)

        else:
            logger.error(f"Échec de connexion MQTT, code: {rc}")
            self.connected.clear()

    def _on_disconnect(self, client, userdata, rc):
        """Callback de déconnexion"""
        logger.warning(f"Déconnecté du broker MQTT, code: {rc}")
        self.connected.clear()

    def _on_publish(self, client, userdata, mid):
        """Callback de publication"""
        logger.debug(f"Message MQTT publié, ID: {mid}")

    def publier_donnees(self, donnees_compteur):
        """
        Publication des données du compteur vers MQTT

        Args:
            donnees_compteur: Dict des données du compteur
        """
        if not self.connected.is_set():
            logger.warning("Client MQTT non connecté, publication ignorée")
            return False

        try:
            # Formatage des données pour Home Assistant
            payload = self._formatter_payload_ha(donnees_compteur)

            # Publication
            result = self.client.publish(
                self.state_topic,
                json.dumps(payload),
                qos=1,
                retain=True
            )

            # Attente de confirmation (optionnel)
            # result.wait_for_publish(timeout=5)

            self.stats['messages_sent'] += 1
            self.stats['last_sent'] = datetime.utcnow()

            logger.debug(f"Données publiées sur MQTT: {len(payload)} valeurs")
            return True

        except Exception as e:
            logger.error(f"Erreur publication MQTT: {e}")
            self.stats['errors'] += 1
            return False

    def _formatter_payload_ha(self, donnees):
        """Formatage des données pour Home Assistant"""
        payload = {
            "timestamp": datetime.utcnow().isoformat() + 'Z',
            "source": donnees.get('source', 'compteur_e450'),
        }

        # Mapping des champs
        mapping = {
            'puissance_active_totale': 'puissance_active_w',
            'tension_l1': 'tension_l1_v',
            'tension_l2': 'tension_l2_v',
            'tension_l3': 'tension_l3_v',
            'courant_l1': 'courant_l1_a',
            'courant_l2': 'courant_l2_a',
            'courant_l3': 'courant_l3_a',
            'frequence': 'frequence_hz',
            'facteur_puissance': 'facteur_puissance',
            'energie_active_import': 'energie_import_wh',
            'energie_active_export': 'energie_export_wh',
        }

        for champ_compteur, champ_ha in mapping.items():
            valeur = donnees.get(champ_compteur)
            if valeur is not None:
                payload[champ_ha] = valeur

        return payload

    def publier_configuration_ha(self):
        """Publication de la configuration Home Assistant (discovery)"""
        if not self.connected.is_set():
            return False

        try:
            # Configuration pour chaque capteur
            capteurs = [
                {
                    'name': 'Puissance Active',
                    'device_class': 'power',
                    'unit_of_measurement': 'W',
                    'value_field': 'puissance_active_w',
                    'unique_id': 'compteur_e450_puissance_active',
                    'icon': 'mdi:flash'
                },
                {
                    'name': 'Tension L1',
                    'device_class': 'voltage',
                    'unit_of_measurement': 'V',
                    'value_field': 'tension_l1_v',
                    'unique_id': 'compteur_e450_tension_l1',
                    'icon': 'mdi:current-ac'
                },
                {
                    'name': 'Tension L2',
                    'device_class': 'voltage',
                    'unit_of_measurement': 'V',
                    'value_field': 'tension_l2_v',
                    'unique_id': 'compteur_e450_tension_l2',
                    'icon': 'mdi:current-ac'
                },
                {
                    'name': 'Tension L3',
                    'device_class': 'voltage',
                    'unit_of_measurement': 'V',
                    'value_field': 'tension_l3_v',
                    'unique_id': 'compteur_e450_tension_l3',
                    'icon': 'mdi:current-ac'
                },
                {
                    'name': 'Courant L1',
                    'device_class': 'current',
                    'unit_of_measurement': 'A',
                    'value_field': 'courant_l1_a',
                    'unique_id': 'compteur_e450_courant_l1',
                    'icon': 'mdi:current-ac'
                },
                {
                    'name': 'Énergie Importée',
                    'device_class': 'energy',
                    'unit_of_measurement': 'Wh',
                    'value_field': 'energie_import_wh',
                    'unique_id': 'compteur_e450_energie_import',
                    'icon': 'mdi:meter-electric'
                },
                {
                    'name': 'Fréquence',
                    'device_class': None,
                    'unit_of_measurement': 'Hz',
                    'value_field': 'frequence_hz',
                    'unique_id': 'compteur_e450_frequence',
                    'icon': 'mdi:sine-wave'
                }
            ]

            for capteur in capteurs:
                config = {
                    "name": capteur['name'],
                    "state_topic": self.state_topic,
                    "value_template": f"{{{{ value_json.{capteur['value_field']} }}}}",
                    "unique_id": capteur['unique_id'],
                    "device": {
                        "identifiers": ["compteur_e450"],
                        "name": "Compteur E450",
                        "model": "Landis+Gyr E450",
                        "manufacturer": "Landis+Gyr"
                    },
                    "availability_topic": f"{self.base_topic}/status",
                    "payload_available": "online",
                    "payload_not_available": "offline"
                }

                # Ajout des champs optionnels
                if capteur['device_class']:
                    config['device_class'] = capteur['device_class']
                if capteur['unit_of_measurement']:
                    config['unit_of_measurement'] = capteur['unit_of_measurement']
                if capteur['icon']:
                    config['icon'] = capteur['icon']

                # Topic de configuration spécifique
                config_topic = f"homeassistant/sensor/compteur_e450/{capteur['unique_id']}/config"

                # Publication
                self.client.publish(
                    config_topic,
                    json.dumps(config),
                    qos=1,
                    retain=True
                )

                logger.info(f"Configuration HA publiée pour {capteur['name']}")

            return True

        except Exception as e:
            logger.error(f"Erreur publication configuration HA: {e}")
            return False

    def obtenir_statistiques(self):
        """Retourne les statistiques du service"""
        return {
            'connected': self.connected.is_set(),
            'broker': f"{self.broker}:{self.port}",
            'stats': self.stats.copy()
        }
```

### Intégration avec le service de lecture

```python
# app/services/lecture_compteur.py (extension)
def _sauvegarder_mesure(self, donnees):
    # ... code existant ...

    # Publication MQTT après sauvegarde
    try:
        from app.services import mqtt_service
        if mqtt_service:
            mqtt_service.publier_donnees(donnees)
    except Exception as e:
        logger.error(f"Erreur publication MQTT: {e}")

    # ... reste du code ...
```

### Configuration dans l'application

```python
# config.py (extension)
class Config:
    # ...

    # MQTT Configuration
    MQTT_BROKER = os.environ.get('MQTT_BROKER', 'localhost')
    MQTT_PORT = int(os.environ.get('MQTT_PORT', '1883'))
    MQTT_USERNAME = os.environ.get('MQTT_USERNAME')
    MQTT_PASSWORD = os.environ.get('MQTT_PASSWORD')
    MQTT_BASE_TOPIC = os.environ.get('MQTT_BASE_TOPIC', 'homeassistant/sensor/compteur_e450')
    MQTT_CLIENT_ID = os.environ.get('MQTT_CLIENT_ID', 'compteur_e450_app')

    # Configuration MQTT complète
    MQTT_CONFIG = {
        'broker': MQTT_BROKER,
        'port': MQTT_PORT,
        'username': MQTT_USERNAME,
        'password': MQTT_PASSWORD,
        'base_topic': MQTT_BASE_TOPIC,
        'client_id': MQTT_CLIENT_ID,
    }
```

### Démarrage du service MQTT

```python
# app/services/__init__.py (extension)
from .mqtt_service import ServiceMQTT

mqtt_service = None

def init_services(app):
    # ... autres services ...

    # Service MQTT
    mqtt_config = app.config.get('MQTT_CONFIG')
    if mqtt_config:
        global mqtt_service
        mqtt_service = ServiceMQTT(mqtt_config)

        # Démarrage automatique
        if app.config.get('AUTO_START_MQTT', True):
            mqtt_service.demarrer()

            # Publication de la configuration HA après connexion
            import time
            def publier_config_ha():
                time.sleep(2)  # Attente de connexion
                if mqtt_service:
                    mqtt_service.publier_configuration_ha()

            import threading
            threading.Thread(target=publier_config_ha, daemon=True).start()
```

## 🔍 Détection automatique dans Home Assistant

### Configuration YAML pour HA

```yaml
# configuration.yaml
mqtt:
  broker: localhost  # ou votre broker MQTT
  port: 1883
  username: !secret mqtt_username
  password: !secret mqtt_password
  discovery: true
  discovery_prefix: homeassistant

# Activation du discovery automatique
# Les capteurs seront automatiquement détectés
```

### Vérification de l'intégration

```python
# scripts/verifier_ha.py
#!/usr/bin/env python3
"""
Vérification de l'intégration Home Assistant
"""

import requests
import json
import time
from datetime import datetime

def verifier_integration_ha(ha_url="http://localhost:8123", ha_token=None):
    """Vérification de l'intégration du compteur dans HA"""

    headers = {}
    if ha_token:
        headers['Authorization'] = f'Bearer {ha_token}'
        headers['Content-Type'] = 'application/json'

    print("🔍 Vérification de l'intégration Home Assistant")
    print("=" * 50)

    # 1. Test de connexion à HA
    try:
        response = requests.get(f"{ha_url}/api/", headers=headers, timeout=10)
        if response.status_code == 200:
            print("✅ Home Assistant accessible")
            ha_info = response.json()
            print(f"   Version: {ha_info.get('version', 'N/A')}")
        else:
            print(f"❌ Home Assistant non accessible (HTTP {response.status_code})")
            return False
    except Exception as e:
        print(f"❌ Erreur de connexion à HA: {e}")
        return False

    # 2. Recherche des entités compteur
    try:
        response = requests.get(f"{ha_url}/api/states", headers=headers, timeout=10)
        if response.status_code == 200:
            entities = response.json()

            # Filtrage des entités compteur
            compteur_entities = [
                entity for entity in entities
                if entity['entity_id'].startswith('sensor.compteur_e450')
            ]

            print(f"📊 Entités compteur trouvées: {len(compteur_entities)}")

            for entity in compteur_entities:
                state = entity.get('state', 'unknown')
                last_updated = entity.get('last_updated', 'N/A')
                print(f"   • {entity['entity_id']}: {state} (mis à jour: {last_updated})")

            if len(compteur_entities) == 0:
                print("⚠️  Aucune entité compteur trouvée")
                print("   Vérifications:")
                print("   - Le service MQTT est-il démarré?")
                print("   - La configuration HA est-elle publiée?")
                print("   - Les topics MQTT sont-ils corrects?")
                return False
            else:
                print("✅ Intégration Home Assistant réussie!")
                return True

    except Exception as e:
        print(f"❌ Erreur récupération entités: {e}")
        return False

def tester_notifications_ha(ha_url="http://localhost:8123", ha_token=None):
    """Test des notifications HA"""

    headers = {}
    if ha_token:
        headers['Authorization'] = f'Bearer {ha_token}'
        headers['Content-Type'] = 'application/json'

    # Test de création d'une notification persistante
    notification_data = {
        "message": f"Test compteur E450 - {datetime.now().strftime('%H:%M:%S')}",
        "title": "Test Intégration",
        "notification_id": "test_compteur_e450"
    }

    try:
        response = requests.post(
            f"{ha_url}/api/services/persistent_notification/create",
            headers=headers,
            json=notification_data,
            timeout=10
        )

        if response.status_code == 200:
            print("✅ Notification HA créée avec succès")
            return True
        else:
            print(f"❌ Échec création notification (HTTP {response.status_code})")
            print(f"   Réponse: {response.text}")
            return False

    except Exception as e:
        print(f"❌ Erreur notification HA: {e}")
        return False

def main():
    """Fonction principale"""
    import argparse

    parser = argparse.ArgumentParser(description='Vérification intégration Home Assistant')
    parser.add_argument('--ha-url', default='http://localhost:8123', help='URL de Home Assistant')
    parser.add_argument('--ha-token', help='Token d\'accès Home Assistant (long-lived token)')
    parser.add_argument('--test-notifications', action='store_true', help='Tester les notifications')

    args = parser.parse_args()

    success = verifier_integration_ha(args.ha_url, args.ha_token)

    if args.test_notifications and success:
        print("\n" + "="*30 + " TEST NOTIFICATIONS " + "="*30)
        tester_notifications_ha(args.ha_url, args.ha_token)

    if success:
        print("\n🎉 Intégration Home Assistant validée!")
        print("Votre compteur E450 est maintenant intégré à votre domotique.")
    else:
        print("\n❌ Problèmes d'intégration détectés.")
        print("Consultez les logs et vérifiez la configuration.")

    return success

if __name__ == "__main__":
    success = main()
    exit(0 if success else 1)
```

#### Test de l'intégration

```bash
# Test de base
python scripts/verifier_ha.py

# Avec token HA
python scripts/verifier_ha.py --ha-token votre_token_long_lived

# Test complet avec notifications
python scripts/verifier_ha.py --test-notifications
```

### Dashboard Home Assistant

```yaml
# Exemple de dashboard HA pour le compteur
views:
  - title: "Énergie"
    icon: mdi:flash
    cards:
      - type: entities
        title: "Compteur E450"
        entities:
          - sensor.compteur_e450_puissance_active
          - sensor.compteur_e450_tension_l1
          - sensor.compteur_e450_tension_l2
          - sensor.compteur_e450_tension_l3
          - sensor.compteur_e450_courant_l1
          - sensor.compteur_e450_energie_importee
          - sensor.compteur_e450_frequence

      - type: history-graph
        title: "Évolution de la puissance"
        hours_to_show: 24
        entities:
          - sensor.compteur_e450_puissance_active
```

> **💡 À retenir** : MQTT + Home Assistant transforme votre compteur en capteur domotique intelligent, avec historique, alertes et automatisation.

> **⚠️ Astuce** : Commencez par tester la connexion MQTT avant d'activer la publication automatique, et utilisez MQTT Explorer pour déboguer les topics.

Dans le prochain chapitre, nous explorerons l'intégration avec InfluxDB et Grafana pour des visualisations temps réel avancées !

---

**Navigation**
- [Chapitre précédent : Lecture réelle du compteur](Chapitre_14_Lecture_Reelle_Compteur.md)
- [Chapitre suivant : Webhook vers InfluxDB + Grafana](Chapitre_16_Webhook_InfluxDB_Grafana.md)
- [Retour à la table des matières](../../README.md)
