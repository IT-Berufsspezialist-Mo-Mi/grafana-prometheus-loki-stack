# Anleitung: Alertmanager mit Slack-Notifications einrichten  
Diese Anleitung beschreibt Schritt für Schritt, wie du:

1. Alertmanager installierst  
2. Eine Slack Incoming Webhook URL erzeugst  
3. Die Alertmanager-Konfiguration für Slack anpasst  
4. Prometheus so konfigurierst, dass Alerts ausgelöst werden  
5. Ein Test-Alert-Rule-File erstellst  
6. Alles überprüfst (Alertmanager UI, Prometheus UI, Slack-Channel)

Alle Schritte orientieren sich am Installationsskript, das du verwendest.

---

# 1. Voraussetzungen

Du benötigst:

- Eine Linux-Maschine mit Prometheus (unter `/opt/prometheus`)  
- Dein Monitoring-Stack sollte bereits laufen:  
  - Prometheus: `http://<vm-ip>:9090`  
  - Grafana: `http://<vm-ip>:3000`  
  - Loki & Promtail (optional)  

Alertmanager wird zusätzlich installiert und läuft später auf:

```
http://<vm-ip>:9093
```

---

# 2. Slack Incoming Webhook einrichten

## 2.1 Slack öffnen

1. Öffne deinen Slack Workspace im Browser  
2. Klicke in der linken Seitenleiste auf „Apps“  
3. Suche nach „Incoming WebHooks“  

Falls die App nicht existiert:

- In der Slack App Directory danach suchen  
- Füge sie für den Workspace hinzu  

## 2.2 Neuen Webhook erstellen

1. Klicke auf „Add to Slack“  
2. Wähle einen Channel, z. B.  

```
#test
```  

3. Klicke auf „Add Incoming Webhook Integration“  

Slack erzeugt dann eine Webhook URL wie:

```
https://hooks.slack.com/services/T00000000/B00000000/XXXXXXXXXXXXXXXXXXXX
```

Diese URL kopieren und sicher speichern.

## 2.3 Bedeutung des Webhooks

Der Webhook ist der Endpunkt, an den Alertmanager seine Nachrichten sendet.  
Jeder Alert wird als JSON-Payload an diese URL geschickt.

---

# 3. Alertmanager installieren

Die Installation erfolgt typischerweise so:

```bash
wget https://github.com/prometheus/alertmanager/releases/download/v0.27.0/alertmanager-0.27.0.linux-amd64.tar.gz
tar -xzf alertmanager-*.tar.gz
sudo mv alertmanager-*/ /opt/alertmanager
sudo mkdir -p /opt/alertmanager/data
```

Alertmanager Version prüfen:

```bash
/opt/alertmanager/alertmanager --version
```

---

# 4. Alertmanager konfigurieren (mit Slack)

Konfigurationsdatei:

```
/opt/alertmanager/alertmanager.yml
```

Bearbeiten:

```bash
sudo nano /opt/alertmanager/alertmanager.yml
```

Inhalt einfügen:

```yaml
global:
  resolve_timeout: 5m

route:
  receiver: slack-notifications

receivers:
  - name: slack-notifications
    slack_configs:
      - channel: '#test'
        api_url: 'https://hooks.slack.com/services/DEIN/SLACK/WEBHOOK'
        send_resolved: true
        title: |-
          [{{ .Status | toUpper }}] Alert: {{ .CommonLabels.alertname }}
        text: >-
          {{ range .Alerts }}
          *Alert:* {{ .Annotations.summary }}
          *Beschreibung:* {{ .Annotations.description }}
          *Labels:* {{ .Labels }}
          {{ end }}
```

Erklärung:

- `channel`: Slack Channel, in dem Alerts erscheinen  
- `api_url`: hier muss deine Webhook URL rein  
- `send_resolved`: sendet eine Nachricht, wenn ein Alert beendet wurde  
- `title` und `text`: steuern die Darstellung in Slack  

---

# 5. Alertmanager als Service registrieren

```bash
sudo tee /etc/systemd/system/alertmanager.service >/dev/null <<EOF
[Unit]
Description=Prometheus Alertmanager
After=network.target

[Service]
ExecStart=/opt/alertmanager/alertmanager   --config.file=/opt/alertmanager/alertmanager.yml   --storage.path=/opt/alertmanager/data
Restart=always

[Install]
WantedBy=multi-user.target
EOF
```

Service starten:

```bash
sudo systemctl daemon-reload
sudo systemctl enable alertmanager
sudo systemctl start alertmanager
```

Status prüfen:

```bash
systemctl status alertmanager
```

---

# 6. Prometheus für Alerting konfigurieren

Datei öffnen:

```
sudo nano /opt/prometheus/prometheus.yml
```

Folgende Zeilen hinzufügen:

```yaml
alerting:
  alertmanagers:
    - static_configs:
        - targets: ["localhost:9093"]

rule_files:
  - /opt/prometheus/rules/*.yml
```

### Erklärung:
- `targets`: Alertmanager läuft auf Port `9093`  
- `rule_files`: Dateien in diesem Ordner enthalten Alert-Regeln  

Prometheus neu starten:

```bash
sudo systemctl restart prometheus
```

---

# 7. Rule File für einen Test-Alert erstellen

Verzeichnis erzeugen:

```bash
sudo mkdir -p /opt/prometheus/rules
```

Datei anlegen:

```bash
sudo nano /opt/prometheus/rules/test-alert.yml
```

Inhalt:

```yaml
groups:
  - name: test-alert
    rules:
      - alert: TestAlert
        expr: vector(1)
        for: 15s
        labels:
          severity: warning
        annotations:
          summary: "Testalarm"
          description: "Dieser Alert wird absichtlich immer ausgelöst."
```

Erklärung:

- `expr: vector(1)` bedeutet: Bedingung ist immer wahr  
- `for: 15s` bedeutet: Alarm löst erst nach 15 Sekunden aus  
- `annotations`: Nachrichten, die in Slack ankommen  

Prometheus neu laden:

```bash
sudo systemctl restart prometheus
```

---

# 8. Alert-Funktion testen

## 8.1 Prometheus UI prüfen

Öffnen:

```
http://<vm-ip>:9090/alerts
```

Du solltest sehen:

```
ALERT TestAlert
Status: firing
```

## 8.2 Alertmanager UI prüfen

```
http://<vm-ip>:9093
```

Unter "Alerts" muss dein TestAlert erscheinen.

## 8.3 Slack prüfen

In `#test` sollte eine Nachricht erscheinen:

```
[ FIRING ] Alert: TestAlert
Beschreibung: Dieser Alert wird absichtlich immer ausgelöst.
```

Wenn `send_resolved: true`, dann nach dem Stoppen auch:

```
[ RESOLVED ] Alert: TestAlert
```

---

# 9. Troubleshooting Guide

### Alert wird nicht ausgelöst
- Prometheus Logs prüfen:
```bash
sudo journalctl -u prometheus -f
```

### Slack bekommt keine Nachrichten
- Webhook URL falsch  
- Channel existiert nicht  
- Netzwerkblockade  

### Alertmanager zeigt Fehler
```bash
sudo journalctl -u alertmanager -f
```

### rule_files laden nicht
- Datei-Endung muss `.yml` sein  
- YAML-Einrückungen prüfen  

---

# 10. Zusammenfassung

Du hast jetzt:

- Einen vollständig installierten Alertmanager  
- Slack Webhook eingerichtet  
- Alerts an Slack gesendet  
- Prometheus um Alerting erweitert  
- Test-Alerts erfolgreich ausgelöst  

Das System ist nun bereit für echte Monitoring-Regeln für CPU, Speicher, Response-Time und Fehleralarme.
