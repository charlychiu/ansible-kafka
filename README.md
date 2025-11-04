# Apache Kafka

![Lint Code Base] ![Molecule]

Ansible role to install and configure [Apache Kafka] 4.0

[Apache Kafka] is a distributed event streaming platform using publish-subscribe
topics. Applications and streaming components can produce and consume messages
by subscribing to these topics. Kafka is extremely fast, handling megabytes of
reads and writes per second from thousands of clients. Messages are persisted
and replicated to prevent data loss. Data streams are partitioned and can be
elastically scaled with no downtime.

## WARNING

This Ansible role does not handle the migration process of upgrading from older
versions of Kafka. Please ensure that you read the upgrade documentation and
update the relevant configuration files before running this role.

<https://kafka.apache.org/40/documentation.html#upgrade>

**Important for Kafka 4.x:** 
- **KRaft mode is required.** ZooKeeper is completely removed in Kafka 4.0. This role configures Kafka in KRaft mode only.
- **Log4j2 is required.** The old Log4j 1.x configuration is no longer supported. This role automatically uses Log4j2 configuration.
- If migrating from ZooKeeper-based Kafka (3.x or earlier), you must migrate to KRaft before upgrading to 4.0. See the [ZooKeeper to KRaft migration guide](https://kafka.apache.org/40/documentation/zk2kraft.html).

## Supported Platforms

- RedHat 7
- RedHat 8
- RedHat 9
- Debian 10.x (Buster)
- Debian 11.x (Bullseye)
- Ubuntu 18.04.x (Bionic)
- Ubuntu 20.04.x (Focal)
- Ubuntu 22.04.x (Jammy)

## Requirements

- Java 17 or higher (required)
- Kafka 4.0+ runs in KRaft mode (no ZooKeeper required)

Ansible 2.9.16 or 2.10.4 are the minimum required versions to workaround an
issue with certain kernels that have broken the `systemd` status check. The
error message "`Service is in unknown state`" will be output when attempting to
start the service via the Ansible role and the task will fail. The service will
start as expected if the `systemctl start` command is run on the physical host.
See <https://github.com/ansible/ansible/issues/71528> for more information.

## Role Variables

| Variable                                       | Default                              | Comments                                                         |
| ---------------------------------------------- | ------------------------------------ | ---------------------------------------------------------------- |
| kafka_download_base_url                        | <https://downloads.apache.org/kafka> |                                                                  |
| kafka_download_validate_certs                  | yes                                  |                                                                  |
| kafka_version                                  | 4.0.1                                |                                                                  |
| kafka_scala_version                            | 2.13                                 |                                                                  |
| kafka_create_user_group                        | true                                 |                                                                  |
| kafka_user                                     | kafka                                |                                                                  |
| kafka_group                                    | kafka                                |                                                                  |
| kafka_root_dir                                 | /opt                                 |                                                                  |
| kafka_dir                                      | {{ kafka_root_dir }}/kafka           |                                                                  |
| kafka_start                                    | yes                                  |                                                                  |
| kafka_restart                                  | yes                                  |                                                                  |
| kafka_log_dir                                  | /var/log/kafka                       |                                                                  |
| kafka_node_id                                  | 1                                    | Unique node ID for KRaft mode (replaces broker.id)              |
| kafka_process_roles                            | broker,controller                    | Roles: broker, controller, or broker,controller                  |
| kafka_controller_quorum_voters                 | 1@localhost:9093                     | Controller quorum voters for KRaft mode (static quorum)          |
| kafka_controller_quorum_bootstrap_servers      | ""                                   | Bootstrap servers for dynamic quorum (Kafka 4.x+)                |
| kafka_controller_listener_names                | CONTROLLER                           | Controller listener name for KRaft mode                          |
| kafka_cluster_uuid                             | ""                                   | Pre-defined cluster UUID (generated if not set)                  |
| kafka_storage_format_mode                      | ""                                   | Format mode: initial-controllers, no-initial-controllers, standalone, or empty (static) |
| kafka_initial_controllers                      | ""                                   | Initial controllers list for dynamic quorum (Approach B)         |
| kafka_java_heap                                | -Xms1G -Xmx1G                        |                                                                  |
| kafka_java_packages                            | []                                   | Java packages to install for Java 17+ requirement                |
| kafka_java_home                                | ""                                   | Custom JAVA_HOME path                                            |
| kafka_java_system_path                         | /usr/local/sbin:...:/bin             | System PATH for Java binaries                                    |
| kafka_listeners                                | []                                   | Broker and controller listeners (auto-configured based on roles) |
| kafka_num_network_threads                      | 3                                    |                                                                  |
| kafka_num_io_threads                           | 8                                    |                                                                  |
| kafka_num_replica_fetchers                     | 1                                    |                                                                  |
| kafka_socket_send_buffer_bytes                 | 102400                               |                                                                  |
| kafka_socket_receive_buffer_bytes              | 102400                               |                                                                  |
| kafka_socket_request_max_bytes                 | 104857600                            |                                                                  |
| kafka_replica_socket_receive_buffer_bytes      | 65536                                |                                                                  |
| kafka_data_log_dirs                            | /var/lib/kafka/logs                  |                                                                  |
| kafka_num_partitions                           | 1                                    |                                                                  |
| kafka_num_recovery_threads_per_data_dir        | 1                                    |                                                                  |
| kafka_log_cleaner_threads                      | 1                                    |                                                                  |
| kafka_offsets_topic_replication_factor         | 3                                    | Replication factor for __consumer_offsets topic                  |
| kafka_transaction_state_log_replication_factor | 3                                    | Replication factor for __transaction_state topic                 |
| kafka_transaction_state_log_min_isr            | 2                                    | Min in-sync replicas for transaction state log                   |
| kafka_log_retention_hours                      | 168                                  |                                                                  |
| kafka_log_segment_bytes                        | 1073741824                           |                                                                  |
| kafka_log_retention_check_interval_ms          | 300000                               |                                                                  |
| kafka_auto_create_topics_enable                | false                                |                                                                  |
| kafka_delete_topic_enable                      | true                                 |                                                                  |
| kafka_default_replication_factor               | 1                                    |                                                                  |
| kafka_bootstrap_servers                        | localhost:9092                       |                                                                  |
| kafka_consumer_group_id                        | kafka-consumer-group                 |                                                                  |
| kafka_opts                                     |                                      | Custom JVM options (e.g., for JMX Exporter)                      |
| kafka_server_config_params                     |                                      | General dictionary that will be templated into server.properties |

### JMX Exporter for Prometheus

This role includes support for [Prometheus JMX Exporter](https://github.com/prometheus/jmx_exporter), which exposes Kafka JMX metrics in Prometheus format for monitoring.

| Variable                      | Default                            | Comments                                                         |
| ----------------------------- | ---------------------------------- | ---------------------------------------------------------------- |
| kafka_jmx_exporter_enabled    | false                              | Enable JMX Exporter for Prometheus monitoring                    |
| kafka_jmx_exporter_version    | 1.0.1                              | JMX Exporter version to download                                 |
| kafka_jmx_exporter_port       | 7071                               | Port for Prometheus to scrape metrics                            |
| kafka_jmx_exporter_dir        | {{ kafka_dir }}/jmx_exporter       | JMX Exporter installation directory                              |
| kafka_jmx_exporter_jar        | {{ kafka_jmx_exporter_dir }}/jmx_prometheus_javaagent-{{ kafka_jmx_exporter_version }}.jar | JMX Exporter jar file path |
| kafka_jmx_exporter_config     | {{ kafka_jmx_exporter_dir }}/kafka-jmx-exporter.yml | JMX Exporter configuration file path |
| kafka_jmx_exporter_url        | https://repo1.maven.org/maven2/... | Download URL for JMX Exporter jar                                |

**Example usage:**

```yaml
- hosts: kafka-nodes
  roles:
    - sleighzy.kafka
  vars:
    kafka_jmx_exporter_enabled: true
    kafka_jmx_exporter_port: 7071
```

Once enabled, Prometheus can scrape metrics from `http://<kafka-host>:7071/metrics`.

The role includes a comprehensive JMX Exporter configuration that exposes:
- Kafka broker metrics (topics, partitions, replication)
- Network request metrics
- Controller metrics
- Log metrics
- JVM metrics (memory, GC, threads)

See [log4j.yml](./defaults/main/002-log4j.yml) for detailed
Log4j2-related available variables. Note: Kafka 4.x uses Log4j2 exclusively.

## Starting and Stopping Kafka services using systemd

- The Kafka service can be started via: `systemctl start kafka`
- The Kafka service can be stopped via: `systemctl stop kafka`

## Starting and Stopping Kafka services using initd

- The Kafka service can be started via: `service kafka start`
- The Kafka service can be stopped via: `service kafka stop`

## Default Properties

| Property                                | Value                |
| --------------------------------------- | -------------------- |
| Kafka node ID (KRaft)                   | 1                    |
| Process roles (KRaft)                   | broker,controller    |
| Kafka bootstrap servers                 | localhost:9092       |
| Kafka consumer group ID                 | kafka-consumer-group |
| Number of partitions                    | 1                    |
| Data log file retention period          | 168 hours            |
| Enable auto topic creation              | false                |
| Enable topic deletion                   | true                 |
| Offsets topic replication factor        | 3                    |
| Transaction state log replication factor| 3                    |
| Transaction state log min ISR           | 2                    |
| Default replication factor              | 1                    |

### Ports

| Port | Description                  |
| ---- | ---------------------------- |
| 9092 | Kafka broker listener port   |
| 9093 | Kafka controller listener port (KRaft) |
| 7071 | JMX Exporter metrics port (when enabled) |
| JMX  | JMX metrics (configurable)   |

Note: JMX metrics are available by default for monitoring Kafka broker performance.
Configure JMX port via `kafka_jmx_port` variable if needed. For Prometheus monitoring,
enable the JMX Exporter which exposes metrics on port 7071 (configurable).

### Directories and Files

| Directory / File                                             |                                         |
| ------------------------------------------------------------ | --------------------------------------- |
| Kafka installation directory (symlink to installed version)  | `/opt/kafka`                            |
| Kafka configuration directory (symlink to /opt/kafka/config) | `/etc/kafka`                            |
| Directory to store data files                                | `/var/lib/kafka/logs`                   |
| Directory to store logs files                                | `/var/log/kafka`                        |
| Log4j2 configuration file                                    | `/etc/kafka/log4j2.yaml`                |
| Kafka service                                                | `/usr/lib/systemd/system/kafka.service` |

## Example Playbook

Add the below to a playbook to run those role against hosts belonging to the
`kafka-nodes` group.

```yaml
- hosts: kafka-nodes
  roles:
    - sleighzy.kafka
```

## Resetting / Cleaning Up Kafka Installation

Two reset playbooks are provided to clean up Kafka installations:

### reset-kafka.yml (Safe Reset)

Stops the Kafka service and removes all Kafka files, but preserves the kafka user and group:

```sh
ansible-playbook -i inventory reset-kafka.yml
```

This playbook will:
- Stop and disable the Kafka service
- Remove all Kafka directories in `/opt/kafka*`
- Remove `/var/lib/kafka` (data directory)
- Remove `/var/log/kafka` (log directory)
- Remove `/etc/kafka` (configuration directory)
- **Keep** the kafka user and group

### reset-kafka-with-user.yml (Complete Reset)

Performs a complete cleanup including the kafka user and group:

```sh
ansible-playbook -i inventory reset-kafka-with-user.yml
```

This playbook includes a safety confirmation prompt. To skip the prompt:

```sh
ansible-playbook -i inventory reset-kafka-with-user.yml -e "confirm_reset=yes"
```

To target specific hosts:

```sh
ansible-playbook -i inventory reset-kafka.yml --limit kafka-node-1
```

**WARNING:** These operations are destructive and will permanently delete all Kafka data. Use with caution!

## Linting

Linting should be done using [ansible-lint].

```sh
pip3 install ansible-lint --user

ansible-lint -c ./.ansible-lint .
```

## Testing

This module uses the [Ansible Molecule] testing framework. This test suite
creates a Kafka cluster consisting of three nodes running in KRaft mode within
Docker containers. Each container runs a different OS to test the supported
platforms for this Ansible role.

As per the [Molecule Installation guide] this should be done using a virtual
environment. The commands below will create a Python virtual environment and
install Molecule including the Docker driver.

```sh
$ python3 -m venv molecule-venv
$ source molecule-venv/bin/activate
(molecule-venv) $ pip3 install ansible docker "molecule-plugins[docker]"
```

Run playbook and tests. Linting errors need to be corrected before Molecule will
execute any tests. This will run all tests and then destroy the Docker
containers.

```sh
molecule test
```

The below command can be used to run the playbook without the tests. This can be
run multiple times when making changes to the role, and ensuring that operations
are idempotent.

```sh
molecule converge
```

The below commands can be used to just run the tests without tearing everything
down. The command `molecule verify` can be repeated for each test run.

```sh
molecule create
molecule converge
molecule verify
```

Tear down Molecule tests and Docker containers.

```sh
molecule destroy
```

## License

![MIT license]

[ansible-lint]: https://docs.ansible.com/ansible-lint/
[ansible molecule]: https://molecule.readthedocs.io/
[apache kafka]: http://kafka.apache.org/
[lint code base]:
  https://github.com/sleighzy/ansible-kafka/workflows/Lint%20Code%20Base/badge.svg
[molecule]:
  https://github.com/sleighzy/ansible-kafka/workflows/Molecule/badge.svg
[molecule installation guide]:
  https://molecule.readthedocs.io/en/stable/installation.html
[mit license]: https://img.shields.io/badge/License-MIT-blue.svg
