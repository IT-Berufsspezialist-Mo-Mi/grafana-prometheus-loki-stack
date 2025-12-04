# Vollständige Anleitung: Backend, Frontend und Nginx manuell installieren und einrichten

Diese Anleitung beschreibt detailliert und nachvollziehbar, wie du das zweite Setup-Skript manuell Schritt für Schritt ausführst.  
Alle Schritte entsprechen exakt dem Skript, das folgende Komponenten installiert:

1. Node.js Backend mit Prometheus-Metriken und Loki-Logging  
2. Nginx als Reverse Proxy  
3. Frontend unter /var/www/frontend  
4. Prometheus- und Promtail-Konfigurationsanpassungen  

Ziel: Ein vollständiger Webstack mit Monitoring und Logging.

---

## 1. Grundvoraussetzungen

Du solltest bereits eine Linux-Maschine installiert haben, z. B. über Multipass:

```bash
multipass launch --name app --mem 4G --disk 20G --cpus 2
multipass shell app
```

---

## 2. Node.js installieren

Zuerst Node.js (LTS-Version) installieren, wie im Skript:

```bash
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt install -y nodejs nginx
```

Prüfen:

```bash
node -v
npm -v
```

---

## 3. Backend-Verzeichnis erstellen

```bash
sudo mkdir -p /opt/backend
cd /opt/backend
```

---

## 4. Backend-Code erstellen

Speichere die Datei:

```bash
sudo tee /opt/backend/server.js >/dev/null << 'EOF'
[...hier steht dein kompletter Backend-Code...]
EOF
```

Dieser Node.js-Server enthält:

- Express API  
- Prometheus Counter + Histogram  
- Default-Metriken  
- Winston-Loki Transport für Logs  
- Routen für Fehler, Last-Tests, Spam-Logs  

---

## 5. Backend-Abhängigkeiten installieren

```bash
cd /opt/backend
sudo npm init -y
sudo npm install express prom-client winston winston-loki cors
```

---

## 6. Systemd-Service für Backend erstellen

Damit das Backend automatisch startet:

```bash
sudo tee /etc/systemd/system/backend.service >/dev/null <<EOF
[Unit]
Description=Demo Backend Service
After=network.target

[Service]
ExecStart=/usr/bin/node /opt/backend/server.js
Restart=always
User=root
Environment=NODE_ENV=production

[Install]
WantedBy=multi-user.target
EOF
```

Aktivieren:

```bash
sudo systemctl daemon-reload
sudo systemctl enable backend
sudo systemctl start backend
```

Status prüfen:

```bash
systemctl status backend
```

---

## 7. Frontend-Verzeichnis erstellen

```bash
sudo mkdir -p /var/www/frontend
```

---

## 8. index.html erstellen

```bash
sudo tee /var/www/frontend/index.html >/dev/null << 'EOF'
[...deine vollständige HTML-Datei...]
EOF
```

---

## 9. script.js erstellen

```bash
sudo tee /var/www/frontend/script.js >/dev/null << 'EOF'
async function callApi(path) {
    const output = document.getElementById("output");
    output.textContent = "Loading...";

    try {
        const res = await fetch("/api" + path);
        const text = await res.text();

        output.textContent =
            "Status: " + res.status + " " + res.statusText + "\n\n" + text;
    } catch (err) {
        output.textContent = "Error: " + err.toString();
    }
}

async function checkStatus() {
    const output = document.getElementById("output");
    output.textContent = "Checking Backend Status...";

    try {
        const res = await fetch("/api/");
        output.textContent = "Backend reachable ✓ (Status " + res.status + ")";
    } catch (err) {
        output.textContent = "Backend unreachable ✗\n" + err.toString();
    }
}
EOF
```

---

## 10. Dateirechte setzen

```bash
sudo chown -R www-data:www-data /var/www/frontend
```

---

## 11. Nginx als Reverse Proxy einrichten

Config-Datei erstellen:

```bash
sudo tee /etc/nginx/sites-available/frontend >/dev/null << 'EOF'
server {
    listen 80;
    server_name _;

    root /var/www/frontend;
    index index.html;

    location / {
        try_files $uri /index.html;
    }

    location /api/ {
        proxy_pass http://localhost:3001/;
        proxy_set_header Host $host;
    }
}
EOF
```

Aktivieren:

```bash
sudo ln -sf /etc/nginx/sites-available/frontend /etc/nginx/sites-enabled/frontend
sudo rm -f /etc/nginx/sites-enabled/default
sudo systemctl restart nginx
```

Testen:

```
http://<vm-ip>/
```

---

## 12. Prometheus-Konfiguration erweitern

Zuerst prüfen:

```bash
sudo grep backend /opt/prometheus/prometheus.yml
```

Falls nicht vorhanden, ergänzen:

```bash
sudo tee -a /opt/prometheus/prometheus.yml >/dev/null << 'EOF'

  - job_name: "backend"
    static_configs:
      - targets: ["localhost:3001"]
EOF
```

Service neu starten:

```bash
sudo systemctl restart prometheus
```

Targets prüfen:

```
http://<vm-ip>:9090/targets
```

---

## 13. Promtail um Nginx-Logs erweitern

Test:

```bash
sudo grep nginx /etc/promtail.yaml
```

Falls nicht vorhanden:

```bash
sudo tee -a /etc/promtail.yaml >/dev/null << 'EOF'

  - job_name: nginx
    static_configs:
      - targets:
          - localhost
        labels:
          job: nginx
          host: logvm
          __path__: /var/log/nginx/*.log
EOF
```

Service neu starten:

```bash
sudo systemctl restart promtail
```

Logs prüfen in Grafana Explore:

```
{job="nginx"}
```

---

## 14. Komponenten prüfen

### Backend:

```bash
curl http://localhost:3001
```

### Prometheus:

```
http://<vm-ip>:9090
```

### Prometheus Metriken:

```bash
curl http://localhost:3001/metrics
```

### Loki Logs über API:

```bash
curl -G http://localhost:3100/loki/api/v1/query --data-urlencode 'query={app="demo-backend"}'
```

### Node Exporter:

```
http://<vm-ip>:9100/metrics
```

### Grafana:

```
http://<vm-ip>:3000
```

---

## 15. Zusammenfassung

Nach Abschluss der Anleitung ist folgendes eingerichtet:

- Voll funktionsfähiges Node.js Backend inklusive Prometheus-Metriken und Loki-Logging  
- Frontend, das über Nginx ausgeliefert wird  
- Nginx Reverse Proxy `/api → Backend`  
- Prometheus liest Backend-Metriken  
- Promtail liest Nginx-Logs  
- Grafana zeigt dir alles in Dashboards  

Der gesamte Stack ist jetzt observierbar, skalierbar und zentral überwacht.

