# Part 17 - Alerting (Prometheus + Alert Manager)

In this part we implement basic Monitoring and Alerting mechanisms with Prometheus and Alert Manager

## Objectives

- Provide a screenshot of a graph of your EC2 instance including available memory, available disk space, and CPU usage. [**SCREENSHOT11**]

- Provide a screenshot of an alert that was sent by Prometheus. [**SCREENSHOT12**]

## Implementation

### Basic Resources Monitoring

Using the Web Portal you can the following queries to extract basic resource usage from the nodes

CPU Usage

```
avg(rate(node_cpu_seconds_total{mode!="idle"}[1m])) by (instance) * 100
```

Available Memory

```
node_memory_Cached_bytes + node_memory_Buffers_bytes + node_memory_MemFree_bytes
```

Available Disk Space (root volume)

```
node_filesystem_free_bytes{device="/dev/root"}
```

Wait for a while and refresh the Web Portal, now you can take a screenshot of a graph of your EC2 instance including available memory, available disk space, and CPU usage. [**SCREENSHOT11**]

![](../assets/part-17/metrics-cpu.png)

![](../assets/part-17/metrics-memory.png)

![](../assets/part-17/metrics-disk.png)

### Alerting with Alert Manager

#### Alerting Destination

Now we need to implement an alerting mechanism, but first we need to set a destination for our alerts, we can use either Email or a **Personal** slack workspace, or both

**Email**

For the Email approach you need to create an app password on your account

You can find extensive guides on how to create app passwords from your account provider, here are some:

- [Google Account](https://support.google.com/accounts/answer/185833?hl=en)
- [Microsoft Account](https://support.microsoft.com/en-us/account-billing/create-app-passwords-from-the-security-info-preview-page-d8bc744a-ce3f-4d4d-89c9-eb38ab9d4137)

**Personal Slack Workspace**

If you decided to go with the slack workspace approach, you can find the [complete instructions](https://api.slack.com/messaging/webhooks) on the official slack documentation

The webhook URL will be later used in alerting configuration

```
https://hooks.slack.com/services/T00000000/B00000000/XXXXXXXXXXXXXXXXXXXXXXXX
```

### Install Alert Manager

Get the latest Alert Manager binaries, at the time of writing the latest version of Alert Manager is 2.41.0

```sh
wget https://github.com/prometheus/alertmanager/releases/download/v0.25.0/alertmanager-0.25.0.linux-amd64.tar.gz
```

Then extract the tarball

```sh
tar xvfz alertmanager-0.25.0.linux-amd64.tar.gz
```

Copy the files to their destinations

```sh
sudo cp alertmanager-0.25.0.linux-amd64/alertmanager /usr/local/bin
sudo cp alertmanager-0.25.0.linux-amd64/amtool /usr/local/bin/
```

Create library folder

```sh
sudo mkdir /var/lib/alertmanager
```

Now you can remove the tarball

```sh
rm -rf alertmanager*
```

Create Alert Manager configuration file

```sh
sudo nano /etc/prometheus/alertmanager.yml
```

Make sure to replace the `your@email.address`, `mail.server.address`, and SLACK_URL with correct values

```yml
global:
  resolve_timeout: 1m
  slack_api_url: "SLACK_URL" # Comment this line if you didn't use slack

route:
  group_by: [Alertname]
  receiver: alert-me

receivers:
  - name: alert-me
    # Email
    email_configs:
      - to: your@email.address
        from: your@email.address
        smarthost: mail.server.address:587
        auth_username: your@email.address
        auth_identity: your@email.address
        auth_password: abcdefghijklmnop
    # Slack
    slack_configs:
      - channel: "#udapeople-alerts"
        send_resolved: true
        icon_url: https://avatars3.githubusercontent.com/u/3380462
        title: |-
          [{{ .Status | toUpper }}{{ if eq .Status "firing" }}:{{ .Alerts.Firing | len }}{{ end }}] {{ .CommonLabels.alertname }} for {{ .CommonLabels.job }}
          {{- if gt (len .CommonLabels) (len .GroupLabels) -}}
          {{" "}}(
          {{- with .CommonLabels.Remove .GroupLabels.Names }}
              {{- range $index, $label := .SortedPairs -}}
              {{ if $index }}, {{ end }}
              {{- $label.Name }}="{{ $label.Value -}}"
              {{- end }}
          {{- end -}}
          )
          {{- end }}
        text: >-
          {{ range .Alerts -}}
          *Alert:* {{ .Annotations.title }}{{ if .Labels.severity }} - `{{ .Labels.severity }}`{{ end }}
          *Description:* {{ .Annotations.description }}
          *Details:*
          {{ range .Labels.SortedPairs }} • *{{ .Name }}:* `{{ .Value }}`
          {{ end }}
          {{ end }}
```

Create systemd services for Prometheus and Node Exporter

```sh
sudo nano /etc/systemd/system/alertmanager.service
```

```
[Unit]
Description=Alert Manager
Wants=network-online.target
After=network-online.target

[Service]
Type=simple
User=prometheus
Group=prometheus
ExecStart=/usr/local/bin/alertmanager \
  --config.file=/etc/prometheus/alertmanager.yml \
  --storage.path=/var/lib/alertmanager

Restart=always

[Install]
WantedBy=multi-user.target
```

Create alerting rules file, the following alert checks if an instance has been down for at least 1 minute

```sh
sudo nano /etc/prometheus/rules.yml
```

```yml
groups:
  - name: Down
    rules:
      - alert: InstanceDown
        expr: up == 0
        for: 1m
        labels:
          severity: "critical"
        annotations:
          summary: "Instance {{ $labels.instance }} down"
          description: |
            {{ $labels.instance }} of job {{ $labels.job }} has been down for more than 1 minute.
            Instance ID: {{ $labels.instance_id }}
            DNS Name: {{ $labels.public_dns_name }}"
```

Update the permissions for application files

```sh
sudo chown -R prometheus:prometheus /etc/prometheus
```

Update Prometheus configuration to use Alert Manager with the specified rules

`/etc/prometheus/prometheus.yml`

```yml
global:
  scrape_interval: 1s
  evaluation_interval: 1s

rule_files:
  - /etc/prometheus/rules.yml

alerting:
  alertmanagers:
    - static_configs:
        - targets:
            - localhost:9093
# The rest of the configuration
```

Restart the Prometheus Service

```
sudo systemctl restart prometheus
```

Make sure that all services are running correctly

```sh
sudo systemctl status prometheus
sudo systemctl status node_exporter
sudo systemctl status alertmanager
```

All services should indicate active services

![](../assets/part-17/alertmanager-service.png)

## Triggering an Alert

We can now trigger an alert by stopping the backend instance and wait for a minute

![](../assets/part-17/stopped-backend-instance.png)

Now check the Alerts tab on the Web Portal, you can see the alert is now active

![](../assets/part-17/alert-pending.png)

Wait for a minute and refresh the portal, now you can see that the alert is firing

![](../assets/part-17/alert-firing.png)

This indicates that the alert message has been sent to its designated routes, i.e. Email and/or Slack

You can now take a screenshot of one of the alerts to provide the twelfth and final screenshot [**SCREENSHOT12**]

**Slack Alert**

![](../assets/part-17/alert-pending.png)

**Email Alert**

![](../assets/part-17/alert-firing.png)

---

Now you can finally prepare the project submission, which we will discuss in the final part

## Troubleshooting

In case you find out that one of the services is not active and throwing an error, it's probably an error in the configuration syntax in one of the files, you can pinpoint where the issue is raised by checking out the logs of the services

```sh
sudo journalctl -u prometheus
sudo journalctl -u alertmanager
sudo journalctl -u node_exporter
```
