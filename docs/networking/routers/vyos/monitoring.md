## Monitoring

### Metrics

```bash
set service monitoring prometheus node-exporter listen-address 192.168.1.1 # (1)!
set service monitoring prometheus node-exporter port 9100 # (2)!
```

1. The IP address or hostname of the router’s Prometheus metrics endpoint
2. The TCP port used by the router to expose Prometheus metrics



Internet connection monitoring
https://docs.vyos.io/en/latest/configuration/service/monitoring.html#blackbox-exporter

### Logs
Not sure this is the correct way.. lets test.
https://docs.vyos.io/en/latest/configuration/service/monitoring.html#loki
