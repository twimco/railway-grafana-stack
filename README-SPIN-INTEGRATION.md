# Spin Platform Integration Guide

This guide explains how to integrate the Spin platform with this Railway Grafana stack template.

## Datasource UIDs

The template uses these datasource UIDs (consistent across all configurations):
- **Prometheus**: `prometheus`
- **Loki**: `loki`
- **Tempo**: `tempo`

These UIDs are used in:
- Grafana datasource provisioning (`grafana/datasources/datasources.yml`)
- Dashboard JSON files (target datasource references)
- Tempo configuration (tracesToLogs, tracesToMetrics)

## Dashboard Upload

The Spin app includes a script to automatically upload and configure dashboards:

```bash
# From your Rails app server
cd /rails
./bin/configure-railway-monitoring
```

This script:
1. Generates Prometheus configuration with scrape targets
2. Creates Grafana datasource provisioning files
3. Uploads dashboards with proper datasource UID references
4. Replaces Kafka references with LavinMQ automatically

## Prometheus Scrape Configuration

Update `prometheus/prom.yml` with your Railway service targets:

```yaml
scrape_configs:
  - job_name: 'rails-app'
    static_configs:
      - targets: ['spin.railway.internal:8080']  # Your Rails app service
    metrics_path: '/metrics'
    scrape_interval: 30s
    scrape_timeout: 10s
    honor_labels: true

  - job_name: 'sidekiq-worker'
    static_configs:
      - targets: ['worker.railway.internal:8080']  # Your Sidekiq worker service
    metrics_path: '/metrics'
    scrape_interval: 30s
    scrape_timeout: 10s
    honor_labels: true

  - job_name: 'lavinmq'
    static_configs:
      - targets: ['your-lavinmq-host:15692']  # Your LavinMQ metrics endpoint
    metrics_path: '/metrics'
    scrape_interval: 30s
    scrape_timeout: 10s
    honor_labels: true
```

## Dashboard Files

Spin platform dashboards are located in:
- `spin-apps/spin/infra/monitoring/dashboards/`

Available dashboards:
- `01-system-overview.json` - System health and overview
- `02-device-fleet.json` - Device fleet monitoring
- `03-rails-performance.json` - Rails application performance
- `04-audio-streaming.json` - Audio streaming metrics
- `06-real-time-performance.json` - Real-time performance metrics
- `07-business-metrics.json` - Business and analytics metrics
- `08-lavinmq-event-system.json` - LavinMQ messaging system

**Note**: Kafka dashboards (`05-kafka-event-system.json`) are skipped as Spin uses LavinMQ.

## Alert Rules

Alert rules are defined in:
- `spin-apps/spin/infra/monitoring/alert-rules.yml`

To use these in Prometheus:
1. Copy `alert-rules.yml` to your Prometheus service
2. Update `prometheus/prom.yml` to reference it:
   ```yaml
   rule_files:
     - '/etc/prometheus/alert-rules.yml'
   ```

## Log Shipping

### Rails App Logs to Loki

Configure your Rails app to send logs to Loki using the internal URL:

```ruby
# In your Rails app
LOKI_URL = ENV['LOKI_INTERNAL_URL']  # http://loki.railway.internal:3100
```

Use a log shipper like Promtail or direct HTTP push to Loki's push API.

### Sidekiq Worker Logs

Same configuration as Rails app - send logs to Loki.

### Raspberry Pi Device Logs

Raspberry Pi devices can send logs via public Loki URL (if configured):

```yaml
# Promtail config on Raspberry Pi
clients:
  - url: https://your-loki-public-url.com/loki/api/v1/push
```

## Trace Shipping

### Rails App Traces to Tempo

Configure OpenTelemetry in your Rails app:

```ruby
# config/initializers/opentelemetry.rb
OpenTelemetry::SDK.configure do |c|
  c.service_name = 'rails-app'
  c.add_span_processor(
    OpenTelemetry::SDK::Trace::Export::BatchSpanProcessor.new(
      OpenTelemetry::Exporter::OTLP::Exporter.new(
        endpoint: ENV.fetch('TEMPO_INTERNAL_URL', 'http://tempo.railway.internal:4318')
      )
    )
  )
end
```

### Raspberry Pi Device Traces

Configure OpenTelemetry on Raspberry Pi devices to send to public Tempo URL.

## Metrics Shipping

### Rails App Metrics

Rails app exposes metrics at `/metrics` endpoint. Prometheus scrapes this endpoint.

### Sidekiq Worker Metrics

Sidekiq worker also exposes metrics at `/metrics`. Add to Prometheus scrape config.

### Raspberry Pi Device Metrics

Devices can push metrics via Prometheus remote write (if public URL configured).

## Verification

After setup, verify:

1. **Prometheus Targets**: Check `http://prometheus.railway.internal:9090/targets`
   - All targets should be "UP"
   - Rails app, Sidekiq worker, and LavinMQ should be scraping

2. **Grafana Datasources**: Check Grafana UI → Configuration → Data Sources
   - Prometheus (uid: prometheus) - should be default
   - Loki (uid: loki)
   - Tempo (uid: tempo)

3. **Dashboards**: Check Grafana UI → Dashboards
   - All Spin Market dashboards should be visible
   - No parse errors in panels
   - Data should be displaying

4. **Loki Logs**: Check Grafana → Explore → Loki
   - Should see logs from Rails app and Sidekiq worker

5. **Tempo Traces**: Check Grafana → Explore → Tempo
   - Should see traces from Rails app

## Troubleshooting

### Dashboards Show Parse Errors

- Verify datasource UIDs match: `prometheus`, `loki`, `tempo`
- Check that datasources are provisioned correctly
- Re-upload dashboards using `configure-railway-monitoring` script

### No Data in Dashboards

- Verify Prometheus is scraping targets (check `/targets` endpoint)
- Check that metrics are being exported from Rails app (`/metrics` endpoint)
- Verify job names in queries match scrape config job names

### Loki Not Receiving Logs

- Verify Loki is accessible at internal URL
- Check log shipper configuration
- Verify log format matches Loki expectations

### Tempo Not Receiving Traces

- Verify OTLP endpoints are correct (port 4317 for gRPC, 4318 for HTTP)
- Check OpenTelemetry configuration in Rails app
- Verify trace format (should be OTLP)

