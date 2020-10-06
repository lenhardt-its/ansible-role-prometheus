# Ansible Role: prometheus

## Description

Deploy [Prometheus](https://github.com/prometheus/prometheus) monitoring system using ansible.

### Upgradability notice

When upgrading from <= 2.4.0 version of this role to >= 2.4.1 please turn off your prometheus instance. More in [2.4.1 release notes](https://github.com/cloudalchemy/ansible-prometheus/releases/tag/2.4.1)

## Requirements

- Ansible >= 2.6 (It might work on previous versions, but we cannot guarantee it)

## Role Variables

All variables which can be overridden are stored in [defaults/main.yml](defaults/main.yml) file as well as in table below.

| Name           | Default Value | Description                        |
| -------------- | ------------- | -----------------------------------|
| `proxy_env` | {} | Proxy environment variables |
| `prometheus_version` | 2.13.1 | Prometheus package version. Also accepts `latest` as parameter. Only prometheus 2.x is supported |
| `prometheus_config_file` | "{{ prometheus_config_dir }}/config.yml" | Path to directory with prometheus configuration file|
| `prometheus_config_dir` | /etc/prometheus | Path to directory with prometheus configuration |
| `prometheus_db_dir` | /var/lib/prometheus | Path to directory with prometheus database |
| `prometheus_binary_install_dir` | /usr/local/bin | Path to directory with prometheus binaries |
| `prometheus_system_user` | prometheus | Prometheus system user |
| `prometheus_system_group` | prometheus | Prometheus system group |
| `prometheus_firewalld_state` | disabled | Install and configure Firewalld and allow prometheus_web_listen_port (enabled/disabled) |
| `prometheus_web_listen_address` | "0.0.0.0" | Address on which prometheus will be listening |
| `prometheus_web_listen_port` | "9090" | Port on which prometheus will be listening |
| `prometheus_web_external_url` | "http://localhost:{{ prometheus_web_listen_port }}" | External address on which prometheus is available. Useful when behind reverse proxy. Ex. `http://example.org/prometheus` |
| `prometheus_storage_retention_time` | "7d" | Data retention period |
| `prometheus_log_level` | warn | Set loglevel |
| `prometheus_log_format` | json | Set logformat |
| `prometheus_create_consul_agent_service` | "true" | Add consul agent config snipped |
| `prometheus_alertmanager_config` | [] | Configuration responsible for pointing where alertmanagers are. This should be specified as list in yaml format. It is compatible with official [<alertmanager_config>](https://prometheus.io/docs/prometheus/latest/configuration/configuration/#alertmanager_config) |
| `prometheus_alert_relabel_configs` | [] | Alert relabeling rules. This should be specified as list in yaml format. It is compatible with the official [<alert_relabel_configs>](https://prometheus.io/docs/prometheus/latest/configuration/configuration/#alert_relabel_configs) |
| `prometheus_global` | { scrape_interval: 60s, scrape_timeout: 15s, evaluation_interval: 15s } | Prometheus global config. Compatible with [official configuration](https://prometheus.io/docs/prometheus/latest/configuration/configuration/#configuration-file) |
| `prometheus_remote_write` | [] | Remote write. Compatible with [official configuration](https://prometheus.io/docs/prometheus/latest/configuration/configuration/#<remote_write>) |
| `prometheus_remote_read` | [] | Remote read. Compatible with [official configuration](https://prometheus.io/docs/prometheus/latest/configuration/configuration/#<remote_read>) |
| `prometheus_external_labels` | environment: "{{ ansible_fqdn \| default(ansible_host) \| default(inventory_hostname) }}" | Provide map of additional labels which will be added to any time series or alerts when communicating with external systems |
| `prometheus_targets` | {} | Targets which will be scraped. Better example is provided in our [demo site](https://github.com/cloudalchemy/demo-site/blob/2a8a56fc10ce613d8b08dc8623230dace6704f9a/group_vars/all/vars#L8) |
| `prometheus_scrape_configs` | [defaults/main.yml#L58](https://github.com/cloudalchemy/ansible-prometheus/blob/ff7830d06ba57be1177f2b6fca33a4dd2d97dc20/defaults/main.yml#L47) | Prometheus scrape jobs provided in same format as in [official docs](https://prometheus.io/docs/prometheus/latest/configuration/configuration/#scrape_config) |
| `prometheus_config_file` | "prometheus.yml.j2" | Variable used to provide custom prometheus configuration file in form of ansible template |
| `prometheus_alert_rules_files` | [defaults/main.yml#L79](https://github.com/cloudalchemy/ansible-prometheus/blob/73d6df05a775ee5b736ac8f28d5605f2a975d50a/defaults/main.yml#L78) | List of folders were ansible will look for files containing alerting rules which will be copied to `{{ prometheus_config_dir }}/rules/`. Files must have `*.rules` extension |
| `prometheus_targets` | [defaults/main.yml#L61](https://github.com/cloudalchemy/ansible-prometheus/blob/73d6df05a775ee5b736ac8f28d5605f2a975d50a/defaults/main.yml#L81) | List of folders were ansible will look for files containing custom static target configuration files which will be copied to `{{ prometheus_config_dir }}/file_sd/`. |


### Relation between `prometheus_scrape_configs` and `prometheus_targets`

#### Short version

`prometheus_targets` is just a map used to create multiple files located in "{{ prometheus_config_dir }}/file_sd" directory. Where file names are composed from top-level keys in that map with `.yml` suffix. Those files store [file_sd scrape targets data](https://prometheus.io/docs/prometheus/latest/configuration/configuration/#file_sd_config) and they need to be read in `prometheus_scrape_configs`.

#### Long version

A part of *prometheus.yml* configuration file which describes what is scraped by prometheus is stored in `prometheus_scrape_configs`. For this variable same configuration options as described in [prometheus docs](https://prometheus.io/docs/prometheus/latest/configuration/configuration/#<scrape_config>) are used.

Meanwhile `prometheus_targets` is our way of adopting [prometheus scrape type `file_sd`](https://prometheus.io/docs/prometheus/latest/configuration/configuration/#<file_sd_config>). It defines a map of files with their content. A top-level keys are base names of files which need to have their own scrape job in `prometheus_scrape_configs` and values are a content of those files.

All this mean that you CAN use custom `prometheus_scrape_configs` with `prometheus_targets` set to `{}`. However when you set anything in `prometheus_targets` it needs to be mapped to `prometheus_scrape_configs`. If it isn't you'll get an error in preflight checks.

#### Example

Lets look at our default configuration, which shows all features. By default we have this `prometheus_targets`:
```
prometheus_targets:
  node:  # This is a base file name. File is located in "{{ prometheus_config_dir }}/file_sd/<<BASENAME>>.yml"
    - targets:              #
        - localhost:9100    # All this is a targets section in file_sd format
      labels:               #
        env: test           #
```
Such config will result in creating one file named `node.yml` in `{{ prometheus_config_dir }}/file_sd` directory.

Next this file needs to be loaded into scrape config. Here is modified version of our default `prometheus_scrape_configs`:
```
prometheus_scrape_configs:
  - job_name: "prometheus"    # Custom scrape job, here using `static_config`
    metrics_path: "/metrics"
    static_configs:
      - targets:
          - "localhost:9090"
  - job_name: "example-node-file-servicediscovery"
    file_sd_configs:
      - files:
          - "{{ prometheus_config_dir }}/file_sd/node.yml" # This line loads file created from `prometheus_targets`
```

## Example

### Playbook

```yaml
---
- hosts: all
  roles:
  - ansible-role-prometheus
  vars:
    prometheus_targets:
      node:
      - targets:
        - localhost:9100
        - useless.everywhere:9100
        labels:
          env: demosite
```

## Contributing

See [contributor guideline](CONTRIBUTING.md).

## License

This project is licensed under MIT License. See [LICENSE](/LICENSE) for more details.
