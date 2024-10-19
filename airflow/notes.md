# Install

Use the community chart which is the stable version - other helm charts from guides may not work:
https://github.com/airflow-helm/charts/blob/main/charts/airflow/docs/guides/quickstart.md

# Troubleshooting

1. Web server container is restarting `exit code: 15`. This indicates that the liveness/ readiness probes are timing out too quickly - possibly caused by having weaker RAM/CPU and the Postgres db taking longer to run all setups. 

    This should was sufficient for the RPi to complete the setup:
        
        readinessProbe:
            enabled: true
            initialDelaySeconds: 10
            periodSeconds: 30
            timeoutSeconds: 30
            failureThreshold: 600

        livenessProbe:
            enabled: true
            initialDelaySeconds: 5
            periodSeconds: 30
            timeoutSeconds: 60
            failureThreshold: 600

        startupProbe:
            enabled: true
            initialDelaySeconds: 5
            periodSeconds: 10
            timeoutSeconds: 15
            failureThreshold: 600

    Different between the probes: https://kubernetes.io/docs/concepts/configuration/liveness-readiness-startup-probes/

    https://apache-airflow.slack.com/archives/CCV3FV9KL/p1724236899651329

# Homelab

https://dev.to/gjrdiesel/the-ultimate-kubernetes-homelab-setup-558f