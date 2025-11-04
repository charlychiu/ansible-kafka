# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is an Ansible role for installing and configuring Apache Kafka 4.0.1 on RHEL/CentOS and Debian/Ubuntu systems. The role handles downloading Kafka binaries, creating users/groups, configuring the broker in KRaft mode, setting up systemd services, and formatting storage.

**Current Version:** Kafka 4.0.1 with Scala 2.13

**Critical Changes in Kafka 4.x:**
- **KRaft mode is required** - ZooKeeper has been completely removed in Kafka 4.0
- **Log4j2 is required** - Log4j 1.x is no longer supported
- Migration from ZooKeeper-based Kafka (3.x or earlier) requires following the [ZooKeeper to KRaft migration guide](https://kafka.apache.org/40/documentation/zk2kraft.html)

## Development Commands

### Linting
```bash
# Install ansible-lint
pip3 install ansible-lint --user

# Run ansible-lint
ansible-lint -c ./.ansible-lint .
```

### Testing with Molecule

Molecule creates a 3-node Kafka cluster in KRaft mode using Docker containers (Debian 10 and RHEL 9).

```bash
# Setup virtual environment (first time only)
python3 -m venv molecule-venv
source molecule-venv/bin/activate
pip3 install ansible docker "molecule-plugins[docker]"

# Full test suite (lint, create, converge, verify, destroy)
molecule test

# Iterative development (converge only, can run multiple times)
molecule converge

# Create and test without destroying
molecule create
molecule converge
molecule verify

# Cleanup
molecule destroy
```

## Architecture

### Role Structure

The role follows standard Ansible role layout:
- `tasks/main.yaml` - Main task orchestration (267 lines)
- `defaults/main/001-kafka.yml` - Kafka configuration variables (KRaft-specific)
- `defaults/main/002-log4j.yml` - Log4j2 configuration variables
- `templates/` - Jinja2 templates for Kafka configuration files
- `handlers/main.yaml` - Service restart handlers
- `vars/` - OS-specific variables (systemd unit paths differ between RedHat/Debian)
- `molecule/default/` - Molecule test configuration and scenarios

### Task Execution Flow

The main task file (`tasks/main.yaml`) executes in this order:

1. **User/Group Creation** (lines 11-29): Creates kafka user/group if `kafka_create_user_group` is true
2. **Download & Install** (lines 31-56): Downloads Kafka tarball, unpacks to `/opt/kafka_<version>`, creates symlink at `/opt/kafka`
3. **Directory Setup** (lines 57-116): Creates data dirs (`/var/lib/kafka/logs`), log dirs (`/var/log/kafka`), config symlinks (`/etc/kafka`)
4. **Configuration** (lines 118-238): Templates all `.properties` files and `log4j2.xml` to `/opt/kafka/config/`, creates symlinks in `/etc/kafka`
5. **Service Setup** (lines 240-266): Installs systemd service or initd script depending on OS
6. **KRaft Storage Initialization** (lines 267-309): Generates cluster UUID, formats KRaft storage using `kafka-storage.sh`
7. **Service Start** (lines 311-318): Starts and enables kafka service if `kafka_start: yes`
8. **Cleanup** (lines 320-325): Removes downloaded tarball

### KRaft Mode Architecture

Kafka 4.x uses KRaft (Kafka Raft) consensus protocol instead of ZooKeeper:

- **Node ID**: Each server has a unique `node.id` (replaces `broker.id`)
- **Process Roles**: Servers can be `broker`, `controller`, or `broker,controller` (combined mode)
- **Controller Quorum**: Controllers form a Raft quorum (e.g., `1@server-1:9093,2@server-2:9093,3@server-3:9093`)
- **Listeners**: Requires separate `CONTROLLER` listener (typically port 9093) in addition to `PLAINTEXT` listener (port 9092)
- **Storage Format**: Before first start, storage must be formatted with `kafka-storage.sh format` using a cluster UUID

The role automatically handles cluster UUID generation and storage formatting (tasks/main.yaml:326-383).

#### Dynamic Quorum vs Static Quorum (Kafka 4.x+)

Kafka 4.x introduces **dynamic quorum** functionality for KRaft mode:

**Static Quorum (Default)**
- Controller quorum is defined in `controller.quorum.voters` configuration
- All controller nodes must be known at format time
- Use this for traditional fixed-topology clusters
- Set `kafka_storage_format_mode: ""` (empty/default)

**Dynamic Quorum (Kafka 4.x+)**
- Controllers can be added/removed dynamically after cluster is running
- Uses `controller.quorum.bootstrap.servers` for controller discovery
- Requires special format flags when initializing storage

Format modes for dynamic quorum:
1. **`--initial-controllers`**: Used for the first set of nodes when creating a new cluster
   - Set `kafka_storage_format_mode: "initial-controllers"`
   - Requires `kafka_initial_controllers` list (format: `id@host:port:directory_uuid`)
   - All initial nodes must be formatted with the same initial controllers list
   - Example: `"1@kafka-1:9093:MvDxzVmcRsaTz33bUuRU6A,2@kafka-2:9093:07R5amHmR32VDA6jHkGbTA"`

2. **`--no-initial-controllers`**: Used for nodes joining an existing dynamic quorum cluster
   - Set `kafka_storage_format_mode: "no-initial-controllers"`
   - Node joins using `controller.quorum.bootstrap.servers` configuration
   - Must be added to quorum via admin operations after starting

3. **`--standalone`**: Used for single-node dynamic quorum (development/testing)
   - Set `kafka_storage_format_mode: "standalone"`
   - Creates a single-node controller quorum that can be expanded later

**Example: Dynamic Quorum Setup**
```yaml
# First 3 nodes (initial controllers)
kafka_storage_format_mode: "initial-controllers"
kafka_initial_controllers: "1@kafka-1:9093:uuid1,2@kafka-2:9093:uuid2,3@kafka-3:9093:uuid3"
kafka_controller_quorum_bootstrap_servers: "kafka-1:9093,kafka-2:9093,kafka-3:9093"

# Additional nodes joining later
kafka_storage_format_mode: "no-initial-controllers"
kafka_controller_quorum_bootstrap_servers: "kafka-1:9093,kafka-2:9093,kafka-3:9093"
```

### Configuration Templates

The role templates these Kafka configuration files:
- `server.properties` - Main broker configuration (KRaft-specific with `node.id`, `process.roles`, `controller.quorum.voters`)
- `connect-standalone.properties` / `connect-distributed.properties` - Kafka Connect
- `producer.properties` / `consumer.properties` - Client configs
- `log4j2.xml` - Log4j2 configuration (required for Kafka 4.x)

Note: `zookeeper.properties` is no longer included as ZooKeeper is removed in Kafka 4.0.

### OS-Specific Handling

- **RedHat 6**: Uses initd service (`kafka.initd.j2`)
- **RedHat 7+/Debian**: Uses systemd service (`kafka.service.j2`)
- Systemd unit path differs: `/usr/lib/systemd/system/` (RedHat) vs `/lib/systemd/system/` (Debian)
- Variables set in `vars/RedHat.yml` and `vars/Debian.yml`

### Variable Layering

Variables are loaded from multiple sources with precedence:
1. Playbook vars (highest priority)
2. `vars/{{ ansible_os_family }}.yml` - OS family-specific vars (loaded in tasks/main.yaml:2-9)
3. `defaults/main/` - Default values for all vars

### Handlers

Two handlers for service restarts (handlers/main.yaml):
- `Restart kafka service` - For general service restarts (uses `service` module)
- `Restart kafka systemd` - For systemd-specific restarts with daemon-reload
- Both check `kafka_restart` variable before executing

## Molecule Testing

The test environment (`molecule/default/molecule.yml`) creates:
- 3-node cluster with network `172.40.0.0/16`
- server-1: Debian 10 (172.40.10.1) - node.id=1
- server-2: RHEL 9 (172.40.10.2) - node.id=2
- server-3: RHEL 9 (172.40.10.3) - node.id=3
- All nodes run in combined `broker,controller` mode
- Controller quorum: `1@server-1:9093,2@server-2:9093,3@server-3:9093`

The verify stage (`molecule/default/verify.yml`) validates:
- kafka user and group exist
- Installation directory created with correct ownership
- Symlinks created correctly (`/opt/kafka` → `/opt/kafka_2.13-<version>`)
- Log directory created (`/var/log/kafka`)
- Config directory created (`/etc/kafka`)
- kafka.service is running and enabled

## Key Variables

**Installation:**
- `kafka_version: 4.0.1` - Kafka version to install
- `kafka_scala_version: 2.13` - Scala version
- `kafka_root_dir: /opt` - Installation root
- `kafka_dir: /opt/kafka` - Symlink to installation

**KRaft Configuration:**
- `kafka_node_id: 1` - Unique node ID (must be different per node in cluster)
- `kafka_process_roles: "broker,controller"` - Server roles (broker, controller, or both)
- `kafka_controller_quorum_voters: "1@localhost:9093"` - Controller quorum voter list (static quorum)
- `kafka_controller_quorum_bootstrap_servers: ""` - Bootstrap servers for dynamic quorum (Kafka 4.x+)
- `kafka_controller_listener_names: "CONTROLLER"` - Controller listener name
- `kafka_cluster_uuid: ""` - Pre-defined cluster UUID (if not set, randomly generated)

**KRaft Storage Format (Kafka 4.x Dynamic Quorum):**
- `kafka_storage_format_mode: ""` - Format mode: `initial-controllers`, `no-initial-controllers`, `standalone`, or empty (default)
- `kafka_initial_controllers: ""` - Initial controllers list for dynamic quorum (format: `id@host:port:directory_uuid`)

**Network:**
- `kafka_listeners: ["PLAINTEXT://:9092", "CONTROLLER://:9093"]` - Listener configuration
- `kafka_java_heap: "-Xms1G -Xmx1G"` - JVM heap settings

**Storage:**
- `kafka_data_log_dirs: /var/lib/kafka/logs` - Data storage location
- `kafka_log_dir: /var/log/kafka` - Application log directory

**General Dictionary:**
- `kafka_server_config_params` - Dictionary for additional server.properties entries

## Linting Configuration

- `.ansible-lint` - Excludes molecule-venv, mocks dependencies
- `.yamllint.yaml` - Custom YAML rules (braces spacing, no octal values, Unix line endings)
- **Important**: Octal values must be quoted strings (e.g., `mode: '0755'` not `mode: 0755`)

## Migration from Kafka 3.x or Earlier

This role is for Kafka 4.0+ only. If upgrading from older versions:
1. Review upgrade docs at <https://kafka.apache.org/40/documentation.html#upgrade>
2. ZooKeeper-based clusters must first migrate to KRaft mode using <https://kafka.apache.org/40/documentation/zk2kraft.html>
3. The role does NOT handle migration - this must be done manually before using this role
