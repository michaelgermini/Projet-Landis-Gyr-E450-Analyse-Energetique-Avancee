# 💻 Chapitre 12 : Conception de l'interface utilisateur

## 🎨 Design system : Néon bleu/rose

Notre interface adopte un **design futuriste** inspiré des dashboards cyberpunk, avec des dégradés néon et des animations fluides.

### Palette de couleurs

```scss
// static/css/themes/neon-theme.scss
:root {
  // Couleurs principales
  --primary-blue: #00d4ff;
  --primary-pink: #ff0080;
  --accent-cyan: #00ffff;
  --accent-purple: #8000ff;

  // Fonds dégradés
  --bg-primary: linear-gradient(135deg, #0a0a0a 0%, #1a1a2e 100%);
  --bg-secondary: linear-gradient(135deg, #16213e 0%, #0f3460 100%);
  --bg-card: rgba(255, 255, 255, 0.03);
  --bg-card-hover: rgba(255, 255, 255, 0.08);

  // Bordures et effets
  --border-glow: 0 0 20px rgba(0, 212, 255, 0.3);
  --border-glow-hover: 0 0 30px rgba(0, 212, 255, 0.6);
  --text-glow: 0 0 10px rgba(0, 212, 255, 0.5);

  // États
  --success: #00ff88;
  --warning: #ffaa00;
  --error: #ff4444;
  --info: #44aaff;

  // Textes
  --text-primary: #ffffff;
  --text-secondary: #b0b0b0;
  --text-muted: #666666;

  // Espacement
  --spacing-xs: 0.25rem;
  --spacing-sm: 0.5rem;
  --spacing-md: 1rem;
  --spacing-lg: 1.5rem;
  --spacing-xl: 2rem;

  // Bordures
  --radius-sm: 4px;
  --radius-md: 8px;
  --radius-lg: 12px;
  --radius-xl: 16px;
}
```

### Typographie

```scss
// Fonts modernes
@import url('https://fonts.googleapis.com/css2?family=JetBrains+Mono:wght@300;400;500;700&family=Rajdhani:wght@300;400;500;700&display=swap');

body {
  font-family: 'Rajdhani', sans-serif;
  font-weight: 400;
  line-height: 1.6;
  color: var(--text-primary);
}

h1, h2, h3, h4, h5, h6 {
  font-family: 'JetBrains Mono', monospace;
  font-weight: 500;
  text-transform: uppercase;
  letter-spacing: 1px;
}

.text-glow {
  text-shadow: var(--text-glow);
}

.mono {
  font-family: 'JetBrains Mono', monospace;
}
```

## 🏠 Dashboard principal

### Structure HTML

```html
<!-- templates/dashboard.html -->
{% extends "base.html" %}

{% block title %}Dashboard - Compteur E450{% endblock %}

{% block extra_head %}
<link rel="stylesheet" href="{{ url_for('static', filename='css/dashboard.css') }}">
<script src="https://cdn.jsdelivr.net/npm/chart.js"></script>
{% endblock %}

{% block content %}
<div class="dashboard-container">
    <!-- Header avec métriques principales -->
    <div class="dashboard-header">
        <div class="metric-card main-power">
            <div class="metric-icon">
                <i class="fas fa-bolt"></i>
            </div>
            <div class="metric-content">
                <div class="metric-value">{{ (mesure_actuelle.puissance_active_totale or 0)|round(0) }}</div>
                <div class="metric-unit">Watts</div>
                <div class="metric-label">Puissance actuelle</div>
            </div>
            <div class="metric-sparkline">
                <canvas id="power-sparkline" width="100" height="30"></canvas>
            </div>
        </div>

        <div class="metric-card energy-today">
            <div class="metric-icon">
                <i class="fas fa-chart-line"></i>
            </div>
            <div class="metric-content">
                <div class="metric-value">{{ cout_estime|round(2) }}</div>
                <div class="metric-unit">€</div>
                <div class="metric-label">Coût estimé aujourd'hui</div>
            </div>
        </div>

        <div class="metric-card voltage">
            <div class="metric-icon">
                <i class="fas fa-wave-square"></i>
            </div>
            <div class="metric-content">
                <div class="metric-value">{{ (mesure_actuelle.tension_l1 or 230)|round(1) }}</div>
                <div class="metric-unit">Volts</div>
                <div class="metric-label">Tension moyenne</div>
            </div>
        </div>
    </div>

    <!-- Graphiques principaux -->
    <div class="dashboard-charts">
        <div class="chart-container power-chart">
            <div class="chart-header">
                <h3>Évolution de la puissance</h3>
                <div class="chart-controls">
                    <button class="time-btn active" data-period="1h">1H</button>
                    <button class="time-btn" data-period="24h">24H</button>
                    <button class="time-btn" data-period="7d">7J</button>
                </div>
            </div>
            <div class="chart-wrapper">
                <canvas id="power-chart"></canvas>
            </div>
        </div>

        <div class="chart-container voltage-chart">
            <div class="chart-header">
                <h3>Tensions par phase</h3>
            </div>
            <div class="chart-wrapper">
                <canvas id="voltage-chart"></canvas>
            </div>
        </div>
    </div>

    <!-- Section événements et alertes -->
    <div class="dashboard-events">
        <div class="events-header">
            <h3>Événements récents</h3>
            <a href="{{ url_for('main.historique') }}" class="see-all">Voir tout</a>
        </div>

        <div class="events-list">
            {% for event in evenements %}
            <div class="event-item event-{{ event.severite }}">
                <div class="event-icon">
                    {% if event.severite == 'error' %}
                        <i class="fas fa-exclamation-triangle"></i>
                    {% elif event.severite == 'warning' %}
                        <i class="fas fa-exclamation-circle"></i>
                    {% else %}
                        <i class="fas fa-info-circle"></i>
                    {% endif %}
                </div>
                <div class="event-content">
                    <div class="event-title">{{ event.titre or event.type_evenement }}</div>
                    <div class="event-description">{{ event.description }}</div>
                    <div class="event-time">{{ event.timestamp|strftime('%H:%M') }}</div>
                </div>
            </div>
            {% endfor %}
        </div>
    </div>

    <!-- Statistiques rapides -->
    <div class="dashboard-stats">
        <div class="stat-card">
            <div class="stat-icon">
                <i class="fas fa-calendar-day"></i>
            </div>
            <div class="stat-content">
                <div class="stat-value">{{ stats_24h.puissance_moyenne|round(0) }}W</div>
                <div class="stat-label">Moyenne 24h</div>
            </div>
        </div>

        <div class="stat-card">
            <div class="stat-icon">
                <i class="fas fa-arrow-up"></i>
            </div>
            <div class="stat-content">
                <div class="stat-value">{{ stats_24h.puissance_max|round(0) }}W</div>
                <div class="stat-label">Pic journalier</div>
            </div>
        </div>

        <div class="stat-card">
            <div class="stat-icon">
                <i class="fas fa-tachometer-alt"></i>
            </div>
            <div class="stat-content">
                <div class="stat-value">{{ stats_24h.nb_mesures }}</div>
                <div class="stat-label">Mesures aujourd'hui</div>
            </div>
        </div>
    </div>
</div>
{% endblock %}

{% block extra_scripts %}
<script src="{{ url_for('static', filename='js/dashboard.js') }}"></script>
{% endblock %}
```

### Styles CSS du dashboard

```css
/* static/css/dashboard.css */

/* Layout principal */
.dashboard-container {
  display: grid;
  grid-template-columns: 1fr;
  gap: var(--spacing-lg);
  padding: var(--spacing-lg);
}

/* Header avec métriques */
.dashboard-header {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(280px, 1fr));
  gap: var(--spacing-md);
}

/* Cartes métriques */
.metric-card {
  background: var(--bg-card);
  border: 1px solid rgba(0, 212, 255, 0.2);
  border-radius: var(--radius-lg);
  padding: var(--spacing-lg);
  display: flex;
  align-items: center;
  gap: var(--spacing-md);
  transition: all 0.3s ease;
  position: relative;
  overflow: hidden;
}

.metric-card::before {
  content: '';
  position: absolute;
  top: 0;
  left: 0;
  right: 0;
  height: 2px;
  background: linear-gradient(90deg, var(--primary-blue), var(--primary-pink));
}

.metric-card:hover {
  transform: translateY(-2px);
  box-shadow: var(--border-glow-hover);
  border-color: var(--primary-blue);
}

.metric-icon {
  width: 60px;
  height: 60px;
  border-radius: 50%;
  background: linear-gradient(135deg, var(--primary-blue), var(--primary-pink));
  display: flex;
  align-items: center;
  justify-content: center;
  color: white;
  font-size: 1.5rem;
  flex-shrink: 0;
}

.metric-content {
  flex: 1;
}

.metric-value {
  font-size: 2rem;
  font-weight: 700;
  font-family: 'JetBrains Mono', monospace;
  background: linear-gradient(45deg, var(--primary-blue), var(--primary-pink));
  -webkit-background-clip: text;
  -webkit-text-fill-color: transparent;
  background-clip: text;
  text-shadow: var(--text-glow);
}

.metric-unit {
  font-size: 0.9rem;
  color: var(--text-secondary);
  margin-bottom: var(--spacing-xs);
}

.metric-label {
  font-size: 0.8rem;
  color: var(--text-muted);
  text-transform: uppercase;
  letter-spacing: 0.5px;
}

.metric-sparkline {
  width: 100px;
  height: 30px;
}

/* Graphiques */
.dashboard-charts {
  display: grid;
  grid-template-columns: 2fr 1fr;
  gap: var(--spacing-lg);
}

.chart-container {
  background: var(--bg-card);
  border: 1px solid rgba(255, 255, 255, 0.1);
  border-radius: var(--radius-lg);
  overflow: hidden;
}

.chart-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: var(--spacing-md) var(--spacing-lg);
  border-bottom: 1px solid rgba(255, 255, 255, 0.1);
}

.chart-header h3 {
  margin: 0;
  color: var(--text-primary);
  font-size: 1.1rem;
}

.chart-controls {
  display: flex;
  gap: var(--spacing-xs);
}

.time-btn {
  background: transparent;
  border: 1px solid rgba(255, 255, 255, 0.2);
  color: var(--text-secondary);
  padding: var(--spacing-xs) var(--spacing-sm);
  border-radius: var(--radius-sm);
  cursor: pointer;
  transition: all 0.2s ease;
  font-size: 0.8rem;
}

.time-btn:hover,
.time-btn.active {
  background: var(--primary-blue);
  border-color: var(--primary-blue);
  color: white;
}

.chart-wrapper {
  padding: var(--spacing-lg);
  position: relative;
}

/* Événements */
.dashboard-events {
  background: var(--bg-card);
  border: 1px solid rgba(255, 255, 255, 0.1);
  border-radius: var(--radius-lg);
  overflow: hidden;
}

.events-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: var(--spacing-md) var(--spacing-lg);
  border-bottom: 1px solid rgba(255, 255, 255, 0.1);
}

.events-header h3 {
  margin: 0;
}

.see-all {
  color: var(--primary-blue);
  text-decoration: none;
  font-size: 0.9rem;
}

.see-all:hover {
  text-decoration: underline;
}

.events-list {
  max-height: 300px;
  overflow-y: auto;
}

.event-item {
  display: flex;
  align-items: center;
  gap: var(--spacing-md);
  padding: var(--spacing-md) var(--spacing-lg);
  border-bottom: 1px solid rgba(255, 255, 255, 0.05);
  transition: background-color 0.2s ease;
}

.event-item:hover {
  background: rgba(255, 255, 255, 0.02);
}

.event-item:last-child {
  border-bottom: none;
}

.event-icon {
  width: 40px;
  height: 40px;
  border-radius: 50%;
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 1.1rem;
  flex-shrink: 0;
}

.event-error .event-icon {
  background: var(--error);
}

.event-warning .event-icon {
  background: var(--warning);
}

.event-info .event-icon {
  background: var(--info);
}

.event-content {
  flex: 1;
}

.event-title {
  font-weight: 500;
  margin-bottom: var(--spacing-xs);
}

.event-description {
  color: var(--text-secondary);
  font-size: 0.9rem;
  margin-bottom: var(--spacing-xs);
}

.event-time {
  color: var(--text-muted);
  font-size: 0.8rem;
}

/* Statistiques */
.dashboard-stats {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(200px, 1fr));
  gap: var(--spacing-md);
}

.stat-card {
  background: var(--bg-card);
  border: 1px solid rgba(255, 255, 255, 0.1);
  border-radius: var(--radius-lg);
  padding: var(--spacing-md);
  display: flex;
  align-items: center;
  gap: var(--spacing-md);
  text-align: center;
}

.stat-icon {
  width: 50px;
  height: 50px;
  border-radius: 50%;
  background: linear-gradient(135deg, var(--accent-cyan), var(--accent-purple));
  display: flex;
  align-items: center;
  justify-content: center;
  color: white;
  font-size: 1.2rem;
  flex-shrink: 0;
}

.stat-content {
  flex: 1;
}

.stat-value {
  font-size: 1.5rem;
  font-weight: 700;
  font-family: 'JetBrains Mono', monospace;
  color: var(--text-primary);
}

.stat-label {
  font-size: 0.8rem;
  color: var(--text-muted);
  text-transform: uppercase;
  letter-spacing: 0.5px;
}

/* Responsive */
@media (max-width: 1024px) {
  .dashboard-charts {
    grid-template-columns: 1fr;
  }

  .dashboard-header {
    grid-template-columns: 1fr;
  }
}

@media (max-width: 768px) {
  .dashboard-container {
    padding: var(--spacing-md);
    gap: var(--spacing-md);
  }

  .metric-card {
    flex-direction: column;
    text-align: center;
    gap: var(--spacing-sm);
  }

  .metric-sparkline {
    display: none;
  }
}
```

## 📊 Graphiques interactifs

### Configuration Chart.js

```javascript
// static/js/dashboard.js

// Configuration globale Chart.js
Chart.defaults.color = '#b0b0b0';
Chart.defaults.borderColor = 'rgba(255, 255, 255, 0.1)';
Chart.defaults.plugins.legend.labels.color = '#b0b0b0';

// Thème néon
const neonTheme = {
  primary: '#00d4ff',
  secondary: '#ff0080',
  accent: '#00ffff',
  success: '#00ff88',
  warning: '#ffaa00',
  error: '#ff4444'
};

// Graphique de puissance
function createPowerChart(data) {
  const ctx = document.getElementById('power-chart').getContext('2d');

  return new Chart(ctx, {
    type: 'line',
    data: {
      labels: data.labels,
      datasets: [{
        label: 'Puissance (W)',
        data: data.values,
        borderColor: neonTheme.primary,
        backgroundColor: 'rgba(0, 212, 255, 0.1)',
        borderWidth: 2,
        fill: true,
        tension: 0.4,
        pointRadius: 0,
        pointHoverRadius: 4,
        pointBackgroundColor: neonTheme.primary,
        pointBorderColor: '#ffffff',
        pointBorderWidth: 2
      }]
    },
    options: {
      responsive: true,
      maintainAspectRatio: false,
      interaction: {
        intersect: false,
        mode: 'index'
      },
      plugins: {
        legend: {
          display: false
        },
        tooltip: {
          backgroundColor: 'rgba(0, 0, 0, 0.8)',
          titleColor: neonTheme.primary,
          bodyColor: '#ffffff',
          borderColor: neonTheme.primary,
          borderWidth: 1,
          cornerRadius: 8,
          callbacks: {
            label: function(context) {
              return context.parsed.y + ' W';
            }
          }
        }
      },
      scales: {
        x: {
          grid: {
            color: 'rgba(255, 255, 255, 0.1)'
          },
          ticks: {
            color: '#b0b0b0'
          }
        },
        y: {
          grid: {
            color: 'rgba(255, 255, 255, 0.1)'
          },
          ticks: {
            color: '#b0b0b0',
            callback: function(value) {
              return value + 'W';
            }
          }
        }
      }
    }
  });
}

// Graphique des tensions
function createVoltageChart(data) {
  const ctx = document.getElementById('voltage-chart').getContext('2d');

  return new Chart(ctx, {
    type: 'line',
    data: {
      labels: data.labels,
      datasets: [
        {
          label: 'Phase 1',
          data: data.phase1,
          borderColor: neonTheme.primary,
          backgroundColor: 'transparent',
          borderWidth: 2,
          tension: 0.4,
          pointRadius: 0
        },
        {
          label: 'Phase 2',
          data: data.phase2,
          borderColor: neonTheme.secondary,
          backgroundColor: 'transparent',
          borderWidth: 2,
          tension: 0.4,
          pointRadius: 0
        },
        {
          label: 'Phase 3',
          data: data.phase3,
          borderColor: neonTheme.accent,
          backgroundColor: 'transparent',
          borderWidth: 2,
          tension: 0.4,
          pointRadius: 0
        }
      ]
    },
    options: {
      responsive: true,
      maintainAspectRatio: false,
      plugins: {
        legend: {
          position: 'bottom',
          labels: {
            usePointStyle: true,
            padding: 20
          }
        }
      },
      scales: {
        x: {
          grid: {
            color: 'rgba(255, 255, 255, 0.1)'
          },
          ticks: {
            color: '#b0b0b0'
          }
        },
        y: {
          grid: {
            color: 'rgba(255, 255, 255, 0.1)'
          },
          ticks: {
            color: '#b0b0b0',
            callback: function(value) {
              return value + 'V';
            }
          }
        }
      }
    }
  });
}

// Chargement des données
async function loadDashboardData(period = '24h') {
  try {
    const response = await fetch(`/api/data?hours=${period === '1h' ? 1 : 24}&limit=100`);
    const result = await response.json();

    if (result.success) {
      const data = result.data;

      // Formatage pour les graphiques
      const chartData = {
        labels: data.map(d => new Date(d.timestamp).toLocaleTimeString()),
        values: data.map(d => d.puissances.active),
        phase1: data.map(d => d.tensions.l1),
        phase2: data.map(d => d.tensions.l2),
        phase3: data.map(d => d.tensions.l3)
      };

      // Mise à jour des graphiques
      if (window.powerChart) {
        window.powerChart.destroy();
      }
      window.powerChart = createPowerChart(chartData);

      if (window.voltageChart) {
        window.voltageChart.destroy();
      }
      window.voltageChart = createVoltageChart(chartData);
    }
  } catch (error) {
    console.error('Erreur chargement données:', error);
  }
}

// Initialisation
document.addEventListener('DOMContentLoaded', function() {
  // Chargement initial
  loadDashboardData();

  // Gestion des boutons de période
  document.querySelectorAll('.time-btn').forEach(btn => {
    btn.addEventListener('click', function() {
      // Mise à jour des boutons actifs
      document.querySelectorAll('.time-btn').forEach(b => b.classList.remove('active'));
      this.classList.add('active');

      // Rechargement des données
      const period = this.dataset.period;
      loadDashboardData(period);
    });
  });

  // Mise à jour automatique toutes les 30 secondes
  setInterval(() => {
    loadDashboardData(document.querySelector('.time-btn.active').dataset.period);
  }, 30000);
});
```

## 📱 Thèmes et modes

### Gestionnaire de thèmes

```javascript
// static/js/theme-manager.js

class ThemeManager {
  constructor() {
    this.themes = {
      neon: 'neon-theme',
      dark: 'dark-theme',
      light: 'light-theme'
    };

    this.currentTheme = localStorage.getItem('theme') || 'neon';
    this.init();
  }

  init() {
    this.applyTheme(this.currentTheme);

    // Bouton de changement de thème (à ajouter dans le HTML)
    const themeBtn = document.getElementById('theme-toggle');
    if (themeBtn) {
      themeBtn.addEventListener('click', () => this.cycleTheme());
    }
  }

  applyTheme(themeName) {
    // Suppression des thèmes existants
    Object.values(this.themes).forEach(theme => {
      document.body.classList.remove(theme);
    });

    // Application du nouveau thème
    const themeClass = this.themes[themeName];
    if (themeClass) {
      document.body.classList.add(themeClass);
      localStorage.setItem('theme', themeName);
      this.currentTheme = themeName;
    }
  }

  cycleTheme() {
    const themeNames = Object.keys(this.themes);
    const currentIndex = themeNames.indexOf(this.currentTheme);
    const nextIndex = (currentIndex + 1) % themeNames.length;
    const nextTheme = themeNames[nextIndex];

    this.applyTheme(nextTheme);
  }

  getCurrentTheme() {
    return this.currentTheme;
  }
}

// Initialisation globale
window.themeManager = new ThemeManager();
```

### Thème sombre alternatif

```scss
// static/css/themes/dark-theme.scss
.dark-theme {
  --bg-primary: linear-gradient(135deg, #1a1a1a 0%, #2d2d2d 100%);
  --bg-secondary: linear-gradient(135deg, #2d2d2d 0%, #404040 100%);
  --bg-card: rgba(255, 255, 255, 0.05);
  --bg-card-hover: rgba(255, 255, 255, 0.1);

  --text-primary: #ffffff;
  --text-secondary: #cccccc;
  --text-muted: #888888;

  --border-glow: 0 0 20px rgba(100, 100, 100, 0.3);
  --border-glow-hover: 0 0 30px rgba(150, 150, 150, 0.5);
}
```

## 🔄 Mise à jour temps réel

### WebSocket pour live updates

```javascript
// static/js/websocket.js

class LiveUpdates {
  constructor() {
    this.ws = null;
    this.reconnectAttempts = 0;
    this.maxReconnectAttempts = 5;
    this.reconnectDelay = 1000;
  }

  connect() {
    const protocol = window.location.protocol === 'https:' ? 'wss:' : 'ws:';
    const wsUrl = `${protocol}//${window.location.host}/ws/live`;

    try {
      this.ws = new WebSocket(wsUrl);

      this.ws.onopen = (event) => {
        console.log('WebSocket connecté');
        this.reconnectAttempts = 0;
        this.send({ type: 'subscribe', channel: 'mesures' });
      };

      this.ws.onmessage = (event) => {
        const data = JSON.parse(event.data);
        this.handleMessage(data);
      };

      this.ws.onclose = (event) => {
        console.log('WebSocket déconnecté');
        this.handleReconnect();
      };

      this.ws.onerror = (error) => {
        console.error('Erreur WebSocket:', error);
      };

    } catch (error) {
      console.error('Erreur connexion WebSocket:', error);
      this.handleReconnect();
    }
  }

  send(data) {
    if (this.ws && this.ws.readyState === WebSocket.OPEN) {
      this.ws.send(JSON.stringify(data));
    }
  }

  handleMessage(data) {
    switch (data.type) {
      case 'nouvelle_mesure':
        this.updateDashboard(data.mesure);
        this.showNotification('Nouvelle mesure reçue', 'info');
        break;

      case 'alerte':
        this.showAlert(data.alerte);
        break;

      case 'evenement':
        this.addEvent(data.evenement);
        break;
    }
  }

  updateDashboard(mesure) {
    // Mise à jour des métriques principales
    const powerElement = document.querySelector('.main-power .metric-value');
    if (powerElement) {
      powerElement.textContent = Math.round(mesure.puissances.active);
    }

    // Mise à jour des graphiques
    if (window.powerChart) {
      // Ajout du nouveau point
      window.powerChart.data.labels.push(new Date(mesure.timestamp).toLocaleTimeString());
      window.powerChart.data.datasets[0].data.push(mesure.puissances.active);

      // Limitation à 50 points
      if (window.powerChart.data.labels.length > 50) {
        window.powerChart.data.labels.shift();
        window.powerChart.data.datasets[0].data.shift();
      }

      window.powerChart.update('none');
    }
  }

  showNotification(message, type = 'info') {
    // Création d'une notification toast
    const toast = document.createElement('div');
    toast.className = `toast toast-${type}`;
    toast.innerHTML = `
      <div class="toast-content">
        <i class="fas fa-info-circle"></i>
        <span>${message}</span>
      </div>
    `;

    document.body.appendChild(toast);

    // Auto-suppression après 3 secondes
    setTimeout(() => {
      toast.remove();
    }, 3000);
  }

  showAlert(alerte) {
    // Affichage d'une alerte importante
    const alertDiv = document.createElement('div');
    alertDiv.className = `alert alert-${alerte.severite}`;
    alertDiv.innerHTML = `
      <div class="alert-icon">
        <i class="fas fa-exclamation-triangle"></i>
      </div>
      <div class="alert-content">
        <div class="alert-title">${alerte.titre}</div>
        <div class="alert-description">${alerte.description}</div>
      </div>
      <button class="alert-close" onclick="this.parentElement.remove()">×</button>
    `;

    document.querySelector('.dashboard-events .events-list').prepend(alertDiv);
  }

  handleReconnect() {
    if (this.reconnectAttempts < this.maxReconnectAttempts) {
      this.reconnectAttempts++;
      console.log(`Tentative de reconnexion ${this.reconnectAttempts}/${this.maxReconnectAttempts}`);

      setTimeout(() => {
        this.connect();
      }, this.reconnectDelay * this.reconnectAttempts);
    } else {
      console.error('Échec de reconnexion WebSocket');
      this.showNotification('Connexion perdue', 'error');
    }
  }

  disconnect() {
    if (this.ws) {
      this.ws.close();
    }
  }
}

// Initialisation
document.addEventListener('DOMContentLoaded', function() {
  window.liveUpdates = new LiveUpdates();
  window.liveUpdates.connect();
});
```

## 🎯 Optimisations UX

### Chargement progressif

```javascript
// static/js/lazy-loading.js

class LazyLoader {
  constructor() {
    this.observer = null;
    this.init();
  }

  init() {
    // Intersection Observer pour le chargement différé
    this.observer = new IntersectionObserver((entries) => {
      entries.forEach(entry => {
        if (entry.isIntersecting) {
          this.loadElement(entry.target);
        }
      });
    }, {
      rootMargin: '50px'
    });

    // Observation des éléments différés
    document.querySelectorAll('[data-lazy]').forEach(el => {
      this.observer.observe(el);
    });
  }

  loadElement(element) {
    const lazyType = element.dataset.lazy;

    switch (lazyType) {
      case 'chart':
        this.loadChart(element);
        break;
      case 'image':
        this.loadImage(element);
        break;
      case 'data':
        this.loadData(element);
        break;
    }

    // Arrêt de l'observation
    this.observer.unobserve(element);
  }

  loadChart(element) {
    const chartId = element.id;
    const chartType = element.dataset.chartType;

    // Chargement des données et création du graphique
    fetch(`/api/chart-data?type=${chartType}`)
      .then(response => response.json())
      .then(data => {
        if (chartType === 'power') {
          window[chartId] = createPowerChart(data);
        }
        // Autres types de graphiques...
      });
  }

  loadImage(element) {
    const src = element.dataset.src;
    if (src) {
      element.src = src;
      element.classList.remove('lazy');
    }
  }

  loadData(element) {
    // Chargement de données supplémentaires
    const endpoint = element.dataset.endpoint;
    fetch(endpoint)
      .then(response => response.json())
      .then(data => {
        element.innerHTML = this.renderData(data);
      });
  }

  renderData(data) {
    // Rendu des données chargées
    return `<div class="loaded-data">${JSON.stringify(data)}</div>`;
  }
}

// Initialisation
document.addEventListener('DOMContentLoaded', function() {
  window.lazyLoader = new LazyLoader();
});
```

### Animations et transitions

```scss
// static/css/animations.scss

// Animations d'entrée
@keyframes fadeInUp {
  from {
    opacity: 0;
    transform: translateY(30px);
  }
  to {
    opacity: 1;
    transform: translateY(0);
  }
}

@keyframes glow {
  0%, 100% {
    box-shadow: 0 0 20px rgba(0, 212, 255, 0.3);
  }
  50% {
    box-shadow: 0 0 30px rgba(0, 212, 255, 0.6);
  }
}

.fade-in-up {
  animation: fadeInUp 0.6s ease-out;
}

.glow-animation {
  animation: glow 2s ease-in-out infinite;
}

// Transitions fluides
.metric-card, .chart-container, .event-item {
  transition: all 0.3s cubic-bezier(0.4, 0, 0.2, 1);
}

// Hover effects
.metric-card:hover {
  transform: translateY(-4px) scale(1.02);
}

.chart-container:hover {
  border-color: var(--primary-blue);
}
```

> **💡 À retenir** : Un bon design UI/UX combine esthétique moderne, performance optimale et accessibilité. Le thème néon apporte une touche futuriste tout en restant fonctionnel.

> **⚠️ Astuce** : Testez toujours votre interface sur différents appareils et résolutions. Les media queries et le design responsive sont essentiels pour une bonne expérience utilisateur.

Dans le dernier chapitre de cette partie, nous implémenterons l'**authentification et la sécurité** pour protéger notre application !

---

**Navigation**
- [Chapitre précédent : Structure du projet Flask](Chapitre_11_Structure_Projet_Flask.md)
- [Chapitre suivant : Authentification et sécurité](Chapitre_13_Authentification_Securite.md)
- [Retour à la table des matières](../../README.md)
