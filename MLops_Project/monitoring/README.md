# Monitoring (plan step 12)

Prometheus scrapes the API's /metrics every 15 s and keeps 30 days of history. Grafana draws it.

| Thing | Address | Login |
|---|---|---|
| API and console | http://localhost:8000 | none |
| Prometheus | http://localhost:9090 (Status > Targets, Alerts) | none |
| Grafana | http://localhost:3000 | admin / admin (change on first login, or set GRAFANA_PASSWORD in a .env file) |

Files
- prometheus/prometheus.yml : what to scrape and how often
- prometheus/alerts.yml : 5 alert rules (no notifications are sent yet)
- grafana/provisioning/datasources/prometheus.yml : connects Grafana to Prometheus automatically
- grafana/starter_dashboard.json : import in Grafana (Dashboards > New > Import) once you have built your first panels

History survives API restarts (Prometheus keeps it in the named volume prometheus_data). `docker compose down -v` deletes it.
Counters in the API reset when it restarts, so always chart rate() or increase(), never raw totals.
