# Raspberry Pi Cluster Ansible Setup

This Ansible project sets up a Raspberry Pi 4 with a Cluster HAT and four Pi Zero nodes for:

- SSH management
- NFS shared storage from the Pi 4
- OpenMPI and `mpi4py`
- Prometheus on the Pi 4
- Prometheus node exporter on every Pi
- Grafana on the Pi 4
- Mosquitto MQTT broker on the Pi 4
- MQTT status publishing from every Pi

It intentionally excludes Docker and K3s.

## Files

```text
inventory.ini
requirements.yml
site.yml
group_vars/all.yml
```

## Before running

Run this playbook from the Pi 4 controller. The `pi4` inventory host uses
`ansible_connection=local`, so running it from another machine will configure
that machine as the controller.

Edit `inventory.ini` if your nodes are not reachable as:

```text
pi4
p1
p2
p3
p4
```

Edit `group_vars/all.yml` if the Zero nodes cannot reach the Pi 4 as `pi4`:

```yaml
controller_cluster_addr: "pi4"
```

Replace it with the Pi 4's Cluster HAT-facing IP or hostname if needed.

Set `cluster_user` in `group_vars/all.yml` to the user account that exists on
every Pi. The Ansible SSH user is derived from that same value.

## Run

From the Pi 4:

```bash
cd pi-cluster-playbook
ansible-galaxy collection install -r requirements.yml --force
ansible-inventory -i inventory.ini --list
ansible-playbook -i inventory.ini site.yml --syntax-check
ansible all -i inventory.ini -m ping
ansible-playbook -i inventory.ini site.yml --ask-become-pass
```

If your cluster user has passwordless sudo, use:

```bash
ansible-playbook -i inventory.ini site.yml
```

## Useful URLs

Prometheus:

```text
http://pi4:9090
```

Grafana:

```text
http://pi4:3000
```

Default Grafana login is usually:

```text
admin / admin
```

## MQTT test

From any node:

```bash
mosquitto_sub -h pi4 -t 'cluster/+/status'
```

## MPI test

From the Pi 4:

```bash
mpirun --hostfile /etc/openmpi/cluster-hostfile hostname
```

Python MPI test:

```bash
mpirun --hostfile /etc/openmpi/cluster-hostfile python3 - <<'PY'
from mpi4py import MPI
import socket

comm = MPI.COMM_WORLD
print(f"rank={comm.rank} size={comm.size} host={socket.gethostname()}")
PY
```

## Security note

The Mosquitto config allows anonymous MQTT access and listens on `0.0.0.0`. This is convenient for a private lab network, but you should add authentication if this is exposed beyond your private cluster network.
