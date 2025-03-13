# MCP Server Deployment Guide for Blockchain Integration

This guide outlines the process for deploying a Model Context Protocol (MCP) server that provides interfaces to multiple blockchain networks. The deployment uses Terraform for infrastructure provisioning, Ansible for configuration management, and includes monitoring with Prometheus and Grafana.

## Table of Contents
- [Prerequisites](#prerequisites)
- [1. Architecture Overview](#1-architecture-overview)
- [2. Infrastructure Provisioning with Terraform](#2-infrastructure-provisioning-with-terraform)
- [3. Configuration Management with Ansible](#3-configuration-management-with-ansible)
- [4. MCP Server Setup](#4-mcp-server-setup)
- [5. Blockchain Node Integration](#5-blockchain-node-integration)
- [6. Monitoring and Maintenance](#6-monitoring-and-maintenance)
- [7. Web Interface & API Development](#7-web-interface--api-development)
- [8. Security Considerations](#8-security-considerations)
- [9. Troubleshooting](#9-troubleshooting)

## Prerequisites

Before beginning deployment, ensure you have the following tools installed:

- Terraform (v1.0.0+)
- Ansible (v2.12.0+)
- Docker and Docker Compose
- Node.js (v16+) for MCP server development
- Git for version control
- Access to your chosen cloud provider or virtualization platform (Proxmox in this example)

## 1. Architecture Overview

The architecture consists of:

- **MCP Server**: Provides standardized protocol for LLM interaction with blockchain data
- **Blockchain Nodes**: Running containerized instances of multiple blockchain networks
- **Monitoring Stack**: Prometheus and Grafana for metrics collection and visualization
- **Web Interface**: React frontend with Node.js backend for management and interaction

```
┌─────────────────┐       ┌───────────────────┐
│                 │       │                   │
│   LLM Client    │◄─────►│    MCP Server     │
│  (Claude, etc)  │       │                   │
│                 │       └───────┬───────────┘
└─────────────────┘               │
                                  ▼
        ┌───────────────────────────────────────────┐
        │                                           │
        │              Blockchain Nodes             │
        │                                           │
        ├───────────┬───────────┬───────────┬───────┤
        │   Flux    │ Filecoin  │ Liberland │  ...  │
        └───────────┴───────────┴───────────┴───────┘
                            │
                            ▼
                  ┌─────────────────────┐
                  │  Monitoring Stack   │
                  │ Prometheus/Grafana  │
                  └─────────────────────┘
```

## 2. Infrastructure Provisioning with Terraform

### 2.1 Terraform Configuration

Create a directory structure for your project:

```bash
mkdir -p mcp-blockchain-deployment/{terraform,ansible,src}
cd mcp-blockchain-deployment/terraform
```

Create the following Terraform files:

**providers.tf**
```hcl
terraform {
  required_providers {
    proxmox = {
      source = "telmate/proxmox"
      version = "2.9.14"
    }
  }
}

provider "proxmox" {
  pm_api_url = var.proxmox_api_url
  pm_user = var.proxmox_user
  pm_password = var.proxmox_password
  pm_tls_insecure = true
}
```

**variables.tf**
```hcl
variable "proxmox_api_url" {
  description = "The URL of the Proxmox API"
  type = string
}

variable "proxmox_user" {
  description = "Proxmox user for API access"
  type = string
}

variable "proxmox_password" {
  description = "Proxmox password for API access"
  type = string
  sensitive = true
}

variable "ssh_public_key" {
  description = "SSH public key for instance access"
  type = string
}

variable "node_count" {
  description = "Number of blockchain nodes to provision"
  type = number
  default = 6
}

variable "proxmox_node" {
  description = "Proxmox node to deploy VMs to"
  type = string
}

variable "template_name" {
  description = "Name of the VM template to clone"
  type = string
  default = "ubuntu-cloud-template"
}
```

**main.tf**
```hcl
resource "proxmox_vm_qemu" "mcp_server" {
  name = "mcp-server"
  target_node = var.proxmox_node
  clone = var.template_name
  
  cores = 4
  sockets = 1
  memory = 8192
  
  disk {
    type = "scsi"
    storage = "local-lvm"
    size = "50G"
  }
  
  network {
    model = "virtio"
    bridge = "vmbr0"
  }
  
  os_type = "cloud-init"
  ipconfig0 = "ip=dhcp"
  sshkeys = var.ssh_public_key
  
  # Generate Ansible inventory entry for MCP server
  provisioner "local-exec" {
    command = "echo '[mcp_servers]' > ../ansible/inventory && echo 'mcp-server ansible_host=${self.default_ipv4_address}' >> ../ansible/inventory"
  }
}

resource "proxmox_vm_qemu" "blockchain_nodes" {
  count = var.node_count
  
  name = "blockchain-node-${count.index + 1}"
  target_node = var.proxmox_node
  clone = var.template_name
  
  cores = 4
  sockets = 1
  memory = 8192
  
  disk {
    type = "scsi"
    storage = "local-lvm"
    # Filecoin requires more storage
    size = count.index == 1 ? "200G" : "100G"
  }
  
  network {
    model = "virtio"
    bridge = "vmbr0"
  }
  
  os_type = "cloud-init"
  ipconfig0 = "ip=dhcp"
  sshkeys = var.ssh_public_key
  
  # Add to Ansible inventory based on node type
  provisioner "local-exec" {
    command = <<-EOT
      if [ ${count.index} -eq 0 ]; then
        echo '[blockchain_nodes]' >> ../ansible/inventory
      fi
      echo 'blockchain-node-${count.index + 1} ansible_host=${self.default_ipv4_address} node_type=${
        count.index == 0 ? "flux" :
        count.index == 1 ? "filecoin" :
        count.index == 2 ? "liberland" :
        count.index == 3 ? "chainlink" :
        count.index == 4 ? "arweave" : "iota"
      }' >> ../ansible/inventory
    EOT
  }
}

# Output IP addresses for reference
output "mcp_server_ip" {
  value = proxmox_vm_qemu.mcp_server.default_ipv4_address
}

output "blockchain_node_ips" {
  value = {
    for idx, node in proxmox_vm_qemu.blockchain_nodes:
    "blockchain-node-${idx + 1}" => node.default_ipv4_address
  }
}
```

Create a `terraform.tfvars` file (DO NOT commit this to version control):

```hcl
proxmox_api_url = "https://your-proxmox-host:8006/api2/json"
proxmox_user = "root@pam"
proxmox_password = "your-secure-password"
ssh_public_key = "ssh-rsa AAAA..."
proxmox_node = "pve"
```

### 2.2 Deploying the Infrastructure

Deploy the infrastructure:

```bash
terraform init
terraform plan
terraform apply
```

## 3. Configuration Management with Ansible

### 3.1 Ansible Configuration

Navigate to the Ansible directory:

```bash
cd ../ansible
```

Create the playbook files:

**ansible.cfg**
```ini
[defaults]
inventory = ./inventory
host_key_checking = False
```

**mcp_server_setup.yml**
```yaml
---
- name: Configure MCP Server
  hosts: mcp_servers
  become: true
  vars:
    mcp_version: "1.0.0"
    node_version: "16.x"
  
  tasks:
    - name: Update apt cache
      apt:
        update_cache: yes
        cache_valid_time: 3600

    - name: Install dependencies
      apt:
        name:
          - apt-transport-https
          - ca-certificates
          - curl
          - gnupg
          - lsb-release
          - git
        state: present

    - name: Add NodeJS repository
      shell: |
        curl -fsSL https://deb.nodesource.com/setup_{{ node_version }} | bash -
      args:
        warn: false

    - name: Install NodeJS
      apt:
        name: nodejs
        state: present

    - name: Install Docker
      block:
        - name: Add Docker GPG key
          apt_key:
            url: https://download.docker.com/linux/ubuntu/gpg
            state: present

        - name: Add Docker repository
          apt_repository:
            repo: deb [arch=amd64] https://download.docker.com/linux/ubuntu {{ ansible_distribution_release }} stable
            state: present

        - name: Install Docker packages
          apt:
            name:
              - docker-ce
              - docker-ce-cli
              - containerd.io
              - docker-compose-plugin
            state: present

        - name: Ensure Docker service is running
          service:
            name: docker
            state: started
            enabled: yes

    - name: Create MCP server directory
      file:
        path: /opt/mcp-server
        state: directory
        mode: '0755'

    - name: Clone MCP server repository
      git:
        repo: https://github.com/your-org/mcp-blockchain-server.git
        dest: /opt/mcp-server
        version: main

    - name: Install MCP server dependencies
      npm:
        path: /opt/mcp-server
        state: present

    - name: Build MCP server
      shell: npm run build
      args:
        chdir: /opt/mcp-server

    - name: Setup MCP server systemd service
      template:
        src: templates/mcp-server.service.j2
        dest: /etc/systemd/system/mcp-server.service
      notify: Restart MCP service

    - name: Enable MCP server service
      systemd:
        name: mcp-server
        enabled: yes
        daemon_reload: yes

  handlers:
    - name: Restart MCP service
      systemd:
        name: mcp-server
        state: restarted
```

**blockchain_node_setup.yml**
```yaml
---
- name: Configure Blockchain Nodes
  hosts: blockchain_nodes
  become: true
  vars:
    data_dir: /opt/blockchain-data
  
  tasks:
    - name: Update apt cache
      apt:
        update_cache: yes
        cache_valid_time: 3600

    - name: Install dependencies
      apt:
        name:
          - apt-transport-https
          - ca-certificates
          - curl
          - gnupg
          - lsb-release
        state: present

    - name: Install Docker
      block:
        - name: Add Docker GPG key
          apt_key:
            url: https://download.docker.com/linux/ubuntu/gpg
            state: present

        - name: Add Docker repository
          apt_repository:
            repo: deb [arch=amd64] https://download.docker.com/linux/ubuntu {{ ansible_distribution_release }} stable
            state: present

        - name: Install Docker packages
          apt:
            name:
              - docker-ce
              - docker-ce-cli
              - containerd.io
              - docker-compose-plugin
            state: present

        - name: Ensure Docker service is running
          service:
            name: docker
            state: started
            enabled: yes

    - name: Create blockchain data directory
      file:
        path: "{{ data_dir }}/{{ node_type }}"
        state: directory
        mode: '0755'

    - name: Deploy Flux node
      when: node_type == "flux"
      block:
        - name: Create Flux configuration directory
          file:
            path: "{{ data_dir }}/flux/config"
            state: directory
            mode: '0755'
            
        - name: Deploy Flux node container
          docker_container:
            name: flux_node
            image: runonflux/fluxnode:latest
            restart_policy: always
            state: started
            volumes:
              - "{{ data_dir }}/flux:/flux/data"
              - "{{ data_dir }}/flux/config:/flux/config"
            ports:
              - "16125:16125"
              - "16126:16126"

    - name: Deploy Filecoin node
      when: node_type == "filecoin"
      block:
        - name: Create Filecoin configuration directory
          file:
            path: "{{ data_dir }}/filecoin/config"
            state: directory
            mode: '0755'
            
        - name: Deploy Filecoin node container
          docker_container:
            name: filecoin_node
            image: filecoin/lotus:latest
            restart_policy: always
            state: started
            volumes:
              - "{{ data_dir }}/filecoin:/data/lotus"
            ports:
              - "1234:1234"

    - name: Deploy Liberland node
      when: node_type == "liberland"
      block:
        - name: Create Liberland configuration directory
          file:
            path: "{{ data_dir }}/liberland/config"
            state: directory
            mode: '0755'
            
        - name: Deploy Liberland node container
          docker_container:
            name: liberland_node
            image: liberland/validator:latest
            restart_policy: always
            state: started
            volumes:
              - "{{ data_dir }}/liberland:/data"
            ports:
              - "26656:26656"
              - "26657:26657"

    - name: Deploy Chainlink node
      when: node_type == "chainlink"
      block:
        - name: Create Chainlink configuration directory
          file:
            path: "{{ data_dir }}/chainlink/config"
            state: directory
            mode: '0755'
            
        - name: Deploy Chainlink node container
          docker_container:
            name: chainlink_node
            image: smartcontract/chainlink:latest
            restart_policy: always
            state: started
            volumes:
              - "{{ data_dir }}/chainlink:/chainlink"
            ports:
              - "6688:6688"

    - name: Deploy Arweave node
      when: node_type == "arweave"
      block:
        - name: Create Arweave configuration directory
          file:
            path: "{{ data_dir }}/arweave/config"
            state: directory
            mode: '0755'
            
        - name: Deploy Arweave node container
          docker_container:
            name: arweave_node
            image: arweave/arweave-server:latest
            restart_policy: always
            state: started
            volumes:
              - "{{ data_dir }}/arweave:/arweave/data"
            ports:
              - "1984:1984"

    - name: Deploy IOTA node
      when: node_type == "iota"
      block:
        - name: Create IOTA configuration directory
          file:
            path: "{{ data_dir }}/iota/config"
            state: directory
            mode: '0755'
            
        - name: Deploy IOTA node container
          docker_container:
            name: iota_node
            image: iotaledger/hornet:latest
            restart_policy: always
            state: started
            volumes:
              - "{{ data_dir }}/iota:/app/hornet/data"
            ports:
              - "14265:14265"
              - "8081:8081"
              - "8091:8091"
```

**monitoring_setup.yml**
```yaml
---
- name: Configure Monitoring Stack
  hosts: mcp_servers
  become: true
  vars:
    monitoring_dir: /opt/monitoring
    
  tasks:
    - name: Create monitoring directory
      file:
        path: "{{ monitoring_dir }}"
        state: directory
        mode: '0755'

    - name: Copy docker-compose file
      template:
        src: templates/monitoring-docker-compose.yml.j2
        dest: "{{ monitoring_dir }}/docker-compose.yml"
        
    - name: Copy Prometheus config
      template:
        src: templates/prometheus.yml.j2
        dest: "{{ monitoring_dir }}/prometheus.yml"
        
    - name: Copy Grafana provisioning files
      block:
        - name: Create Grafana provisioning directories
          file:
            path: "{{ monitoring_dir }}/grafana/{{ item }}"
            state: directory
            mode: '0755'
          loop:
            - provisioning/datasources
            - provisioning/dashboards
            - dashboards
            
        - name: Copy datasource config
          template:
            src: templates/datasource.yml.j2
            dest: "{{ monitoring_dir }}/grafana/provisioning/datasources/datasource.yml"
            
        - name: Copy dashboard config
          template:
            src: templates/dashboards.yml.j2
            dest: "{{ monitoring_dir }}/grafana/provisioning/dashboards/dashboards.yml"
            
        - name: Copy blockchain dashboard
          copy:
            src: files/blockchain-dashboard.json
            dest: "{{ monitoring_dir }}/grafana/dashboards/blockchain-dashboard.json"

    - name: Deploy monitoring stack
      community.docker.docker_compose:
        project_src: "{{ monitoring_dir }}"
        state: present
```

Create template files:

Mkdir templates directory:
```bash
mkdir -p templates files
```

**templates/mcp-server.service.j2**
```
[Unit]
Description=MCP Blockchain Server
After=network.target

[Service]
Type=simple
User=root
WorkingDirectory=/opt/mcp-server
ExecStart=/usr/bin/node build/index.js
Restart=on-failure
RestartSec=10
StandardOutput=syslog
StandardError=syslog
SyslogIdentifier=mcp-server

[Install]
WantedBy=multi-user.target
```

**templates/monitoring-docker-compose.yml.j2**
```yaml
version: '3'

services:
  prometheus:
    image: prom/prometheus
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus_data:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
      - '--web.console.libraries=/etc/prometheus/console_libraries'
      - '--web.console.templates=/etc/prometheus/consoles'
      - '--web.enable-lifecycle'
    ports:
      - "9090:9090"
    restart: always

  grafana:
    image: grafana/grafana
    volumes:
      - grafana_data:/var/lib/grafana
      - ./grafana/provisioning:/etc/grafana/provisioning
      - ./grafana/dashboards:/var/lib/grafana/dashboards
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=secure-password-here
      - GF_USERS_ALLOW_SIGN_UP=false
      - GF_SERVER_ROOT_URL=%(protocol)s://%(domain)s:%(http_port)s/grafana/
      - GF_SERVER_SERVE_FROM_SUB_PATH=true
    ports:
      - "3000:3000"
    depends_on:
      - prometheus
    restart: always

  node-exporter:
    image: prom/node-exporter
    volumes:
      - /proc:/host/proc:ro
      - /sys:/host/sys:ro
      - /:/rootfs:ro
    command:
      - '--path.procfs=/host/proc'
      - '--path.rootfs=/rootfs'
      - '--path.sysfs=/host/sys'
      - '--collector.filesystem.mount-points-exclude=^/(sys|proc|dev|host|etc)($$|/)'
    ports:
      - "9100:9100"
    restart: always

volumes:
  prometheus_data:
  grafana_data:
```

**templates/prometheus.yml.j2**
```yaml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

alerting:
  alertmanagers:
    - static_configs:
        - targets:
          # - alertmanager:9093

rule_files:
  # - "first_rules.yml"
  # - "second_rules.yml"

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']

  - job_name: 'node-exporter'
    static_configs:
      - targets: ['node-exporter:9100']

  - job_name: 'mcp-server'
    static_configs:
      - targets: ['{{ hostvars[inventory_hostname].ansible_default_ipv4.address }}:8000']

  - job_name: 'blockchain-nodes'
    static_configs:
      {% for host in groups['blockchain_nodes'] %}
      - targets: ['{{ hostvars[host].ansible_host }}:9100']
        labels:
          node: {{ hostvars[host].node_type }}
      {% endfor %}
```

**templates/datasource.yml.j2**
```yaml
apiVersion: 1

datasources:
  - name: Prometheus
    type: prometheus
    access: proxy
    url: http://prometheus:9090
    isDefault: true
```

**templates/dashboards.yml.j2**
```yaml
apiVersion: 1

providers:
  - name: 'Blockchain'
    orgId: 1
    folder: ''
    type: file
    disableDeletion: false
    editable: true
    options:
      path: /var/lib/grafana/dashboards
```

Create a basic dashboard file:

**files/blockchain-dashboard.json**
```json
{
  "annotations": {
    "list": []
  },
  "editable": true,
  "fiscalYearStartMonth": 0,
  "graphTooltip": 0,
  "id": 1,
  "links": [],
  "liveNow": false,
  "panels": [
    {
      "datasource": {
        "type": "prometheus",
        "uid": "PBFA97CFB590B2093"
      },
      "fieldConfig": {
        "defaults": {
          "color": {
            "mode": "palette-classic"
          },
          "custom": {
            "axisCenteredZero": false,
            "axisColorMode": "text",
            "axisLabel": "",
            "axisPlacement": "auto",
            "barAlignment": 0,
            "drawStyle": "line",
            "fillOpacity": 10,
            "gradientMode": "none",
            "hideFrom": {
              "legend": false,
              "tooltip": false,
              "viz": false
            },
            "lineInterpolation": "linear",
            "lineWidth": 1,
            "pointSize": 5,
            "scaleDistribution": {
              "type": "linear"
            },
            "showPoints": "auto",
            "spanNulls": false,
            "stacking": {
              "group": "A",
              "mode": "none"
            },
            "thresholdsStyle": {
              "mode": "off"
            }
          },
          "mappings": [],
          "thresholds": {
            "mode": "absolute",
            "steps": [
              {
                "color": "green",
                "value": null
              }
            ]
          },
          "unit": "bytes"
        },
        "overrides": []
      },
      "gridPos": {
        "h": 8,
        "w": 12,
        "x": 0,
        "y": 0
      },
      "id": 1,
      "options": {
        "legend": {
          "calcs": [],
          "displayMode": "list",
          "placement": "bottom",
          "showLegend": true
        },
        "tooltip": {
          "mode": "single",
          "sort": "none"
        }
      },
      "targets": [
        {
          "datasource": {
            "type": "prometheus",
            "uid": "PBFA97CFB590B2093"
          },
          "expr": "node_filesystem_avail_bytes{mountpoint=\"/\"}",
          "refId": "A"
        }
      ],
      "title": "Available Disk Space",
      "type": "timeseries"
    },
    {
      "datasource": {
        "type": "prometheus",
        "uid": "PBFA97CFB590B2093"
      },
      "fieldConfig": {
        "defaults": {
          "color": {
            "mode": "palette-classic"
          },
          "custom": {
            "axisCenteredZero": false,
            "axisColorMode": "text",
            "axisLabel": "",
            "axisPlacement": "auto",
            "barAlignment": 0,
            "drawStyle": "line",
            "fillOpacity": 10,
            "gradientMode": "none",
            "hideFrom": {
              "legend": false,
              "tooltip": false,
              "viz": false
            },
            "lineInterpolation": "linear",
            "lineWidth": 1,
            "pointSize": 5,
            "scaleDistribution": {
              "type": "linear"
            },
            "showPoints": "auto",
            "spanNulls": false,
            "stacking": {
              "group": "A",
              "mode": "none"
            },
            "thresholdsStyle": {
              "mode": "off"
            }
          },
          "mappings": [],
          "thresholds": {
            "mode": "absolute",
            "steps": [
              {
                "color": "green",
                "value": null
              }
            ]
          },
          "unit": "percent"
        },
        "overrides": []
      },
      "gridPos": {
        "h": 8,
        "w": 12,
        "x": 12,
        "y": 0
      },
      "id": 2,
      "options": {
        "legend": {
          "calcs": [],
          "displayMode": "list",
          "placement": "bottom",
          "showLegend": true
        },
        "tooltip": {
          "mode": "single",
          "sort": "none"
        }
      },
      "targets": [
        {
          "datasource": {
            "type": "prometheus",
            "uid": "PBFA97CFB590B2093"
          },
          "expr": "100 - ((node_filesystem_avail_bytes{mountpoint=\"/\"} * 100) / node_filesystem_size_bytes{mountpoint=\"/\"})",
          "refId": "A"
        }
      ],
      "title": "Disk Usage",
      "type": "timeseries"
    },
    {
      "datasource": {
        "type": "prometheus",
        "uid": "PBFA97CFB590B2093"
      },
      "fieldConfig": {
        "defaults": {
          "color": {
            "mode": "palette-classic"
          },
          "custom": {
            "axisCenteredZero": false,
            "axisColorMode": "text",
            "axisLabel": "",
            "axisPlacement": "auto",
            "barAlignment": 0,
            "drawStyle": "line",
            "fillOpacity": 10,
            "gradientMode": "none",
            "hideFrom": {
              "legend": false,
              "tooltip": false,
              "viz": false
            },
            "lineInterpolation": "linear",
            "lineWidth": 1,
            "pointSize": 5,
            "scaleDistribution": {
              "type": "linear"
            },
            "showPoints": "auto",
            "spanNulls": false,
            "stacking": {
              "group": "A",
              "mode": "none"
            },
            "thresholdsStyle": {
              "mode": "off"
            }
          },
          "mappings": [],
          "thresholds": {
            "mode": "absolute",
            "steps": [
              {
                "color": "green",
                "value": null
              }
            ]
          },
          "unit": "percent"
        },
        "overrides": []
      },
      "gridPos": {
        "h": 8,
        "w": 12,
        "x": 0,
        "y": 8
      },
      "id": 3,
      "options": {
        "legend": {
          "calcs": [],
          "displayMode": "list",
          "placement": "bottom",
          "showLegend": true
        },
        "tooltip": {
          "mode": "single",
          "sort": "none"
        }
      },
      "targets": [
        {
          "datasource": {
            "type": "prometheus",
            "uid": "PBFA97CFB590B2093"
          },
          "expr": "100 * (1 - avg by(instance)(irate(node_cpu_seconds_total{mode=\"idle\"}[5m])))",
          "refId": "A"
        }
      ],
      "title": "CPU Usage",
      "type": "timeseries"
    },
    {
      "datasource": {
        "type": "prometheus",
        "uid": "PBFA97CFB590B2093"
      },
      "fieldConfig": {
        "defaults": {
          "color": {
            "mode": "palette-classic"
          },
          "custom": {
            "axisCenteredZero": false,
            "axisColorMode": "text",
            "axisLabel": "",
            "axisPlacement": "auto",
            "barAlignment": 0,
            "drawStyle": "line",
            "fillOpacity": 10,
            "gradientMode": "none",
            "hideFrom": {
              "legend": false,
              "tooltip": false,
              "viz": false
            },
            "lineInterpolation": "linear",
            "lineWidth": 1,
            "pointSize": 5,
            "scaleDistribution": {
              "type": "linear"
            },
            "showPoints": "auto",
            "spanNulls": false,
            "stacking": {
              "group": "A",
              "mode": "none"
            },
            "thresholdsStyle": {
              "mode": "off"
            }
          },
          "mappings": [],
          "thresholds": {
            "mode": "absolute",
            "steps": [
              {
                "color": "green",
                "value": null
              }
            ]
          },
          "unit": "bytes"
        },
        "overrides": []
      },
      "gridPos": {
        "h": 8,
        "w": 12,
        "x": 12,
        "y": 8
      },
      "id": 4,
      "options": {
        "legend": {
          "calcs": [],
          "displayMode": "list",
          "placement": "bottom",
          "showLegend": true
        },
        "tooltip": {
          "mode": "single",
          "sort": "none"
        }
      },
      "targets": [
        {
          "datasource": {
            "type": "prometheus",
            "uid": "PBFA97CFB590B2093"
          },
          "expr": "node_memory_MemTotal_bytes - node_memory_MemFree_bytes - node_memory_Buffers_bytes - node_memory_Cached_bytes",
          "refId": "A"
        }
      ],
      "title": "Memory Usage",
      "type": "timeseries"
    }
  ],
  "refresh": "",
  "schemaVersion": 38,
  "style": "dark",
  "tags": [],
  "templating": {
    "list": []
  },
  "time": {
    "from": "now-6h",
    "to": "now"
  },
  "timepicker": {},
  "timezone": "",
  "title": "Blockchain Nodes Dashboard",
  "version": 1,
  "weekStart": ""
}
```

### 3.2 Run Ansible Playbooks

Execute the playbooks:

```bash
ansible-playbook mcp_server_setup.yml
ansible-playbook blockchain_node_setup.yml
ansible-playbook monitoring_setup.yml
```

## 4. MCP Server Setup

### 4.1 MCP Server Structure

Create the MCP server code in your development environment:

```bash
cd ../src
mkdir -p {server,blockchain-integrations,api}
```

### 4.2 Server Implementation

The MCP server will consist of these core components:

**package.json**
```json
{
  "name": "mcp-blockchain-server",
  "version": "1.0.0",
  "description": "MCP server for blockchain integration",
  "main": "build/index.js",
  "type": "module",
  "scripts": {
    "build": "tsc",
    "start": "node build/index.js",
    "dev": "tsx watch src/index.ts"
  },
  "dependencies": {
    "@modelcontextprotocol/sdk": "^0.5.0",
    "express": "^4.18.2",
    "zod": "^3.22.4",
    "web3": "^4.0.3",
    "axios": "^1.6.0",
    "dotenv": "^16.3.1",
    "cors": "^2.8.5",
    "prom-client": "^14.2.0"
  },
  "devDependencies": {
    "@types/express": "^4.17.20",
    "@types/cors": "^2.8.15",
    "@types/node": "^20.8.7",
    "typescript": "^5.2.2",
    "tsx": "^3.14.0"
  }
}
```

**tsconfig.json**
```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "Node16",
    "moduleResolution": "Node16",
    "outDir": "./build",
    "rootDir": "./src",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules"]
}
```

**src/index.ts**
```typescript
import dotenv from "dotenv";
import { McpServer } from "@modelcontextprotocol/sdk/server/mcp.js";
import { StdioServerTransport } from "@modelcontextprotocol/sdk/server/stdio.js";
import { setupMetricsServer } from "./metrics.js";
import { setupBlockchainTools } from "./blockchain-integrations/index.js";
import { setupBlockchainResources } from "./blockchain-integrations/resources.js";
import { setupBlockchainPrompts } from "./blockchain-integrations/prompts.js";
import { setupAPIServer } from "./api/index.js";

// Load environment variables
dotenv.config();

async function main() {
  console.error("Starting MCP Blockchain Server...");
  
  // Create MCP server instance
  const server = new McpServer({
    name: "blockchain-mcp-server",
    version: "1.0.0",
  });
  
  // Setup blockchain tools, resources, and prompts
  setupBlockchainTools(server);
  setupBlockchainResources(server);
  setupBlockchainPrompts(server);
  
  // Start metrics server for Prometheus scraping
  setupMetricsServer(8000);
  
  // Start the API server for web interface
  setupAPIServer(3001);
  
  // Connect to transport
  console.error("Connecting to stdio transport...");
  const transport = new StdioServerTransport();
  await server.connect(transport);
  console.error("MCP Server running on stdio transport");
}

// Start the server
main().catch(error => {
  console.error("Fatal error in MCP server:", error);
  process.exit(1);
});
```

**src/metrics.ts**
```typescript
import express from "express";
import { register, collectDefaultMetrics } from "prom-client";

export function setupMetricsServer(port: number) {
  // Set up metrics collection
  collectDefaultMetrics();
  
  // Create a custom metric for blockchain operations
  const blockchainOperationsCounter = new register.Counter({
    name: "blockchain_operations_total",
    help: "Count of blockchain operations performed",
    labelNames: ["blockchain", "operation"]
  });
  
  // Create express app for metrics exposure
  const app = express();
  
  // Metrics endpoint
  app.get("/metrics", async (req, res) => {
    res.set("Content-Type", register.contentType);
    res.end(await register.metrics());
  });
  
  // Start server
  app.listen(port, () => {
    console.error(`Metrics server listening on port ${port}`);
  });
  
  return { blockchainOperationsCounter };
}
```

**src/blockchain-integrations/index.ts**
```typescript
import { McpServer } from "@modelcontextprotocol/sdk/server/mcp.js";
import { z } from "zod";
import { fluxTools } from "./flux.js";
import { filecoinTools } from "./filecoin.js";
import { liberlandTools } from "./liberland.js";
import { chainlinkTools } from "./chainlink.js";
import { arweaveTools } from "./arweave.js";
import { iotaTools } from "./iota.js";

export function setupBlockchainTools(server: McpServer) {
  // Register all blockchain tools
  
  // Flux tools
  fluxTools.forEach(tool => {
    server.tool(tool.name, tool.schema, tool.handler);
  });
  
  // Filecoin tools
  filecoinTools.forEach(tool => {
    server.tool(tool.name, tool.schema, tool.handler);
  });
  
  // Liberland tools
  liberlandTools.forEach(tool => {
    server.tool(tool.name, tool.schema, tool.handler);
  });
  
  // Chainlink tools
  chainlinkTools.forEach(tool => {
    server.tool(tool.name, tool.schema, tool.handler);
  });
  
  // Arweave tools
  arweaveTools.forEach(tool => {
    server.tool(tool.name, tool.schema, tool.handler);
  });
  
  // IOTA tools
  iotaTools.forEach(tool => {
    server.tool(tool.name, tool.schema, tool.handler);
  });
  
  // Generic wallet info tool that works across blockchains
  server.tool(
    "get-wallet-info",
    "Get information about a wallet address from any supported blockchain",
    {
      blockchain: z.enum(["flux", "filecoin", "liberland", "chainlink", "arweave", "iota"]),
      address: z.string()
    },
    async ({ blockchain, address }) => {
      try {
        // Route to specific blockchain handler
        let walletInfo;
        switch (blockchain) {
          case "flux":
            walletInfo = await fluxTools.find(t => t.name === "flux-wallet-info")?.handler({ address });
            break;
          case "filecoin":
            walletInfo = await filecoinTools.find(t => t.name === "filecoin-wallet-info")?.handler({ address });
            break;
          // Add other blockchain handlers
          default:
            return {
              content: [{ 
                type: "text", 
                text: `Blockchain ${blockchain} not supported for wallet info.` 
              }],
              isError: true
            };
        }
        
        return walletInfo;
      } catch (error) {
        return {
          content: [{ 
            type: "text", 
            text: `Error retrieving wallet info: ${error instanceof Error ? error.message : String(error)}` 
          }],
          isError: true
        };
      }
    }
  );
}
```

**src/blockchain-integrations/resources.ts**
```typescript
import { McpServer, ResourceTemplate } from "@modelcontextprotocol/sdk/server/mcp.js";

export function setupBlockchainResources(server: McpServer) {
  // Register blockchain resources
  
  // Flux blockchain info resource
  server.resource(
    "flux-info",
    "flux://info",
    async (uri) => ({
      contents: [{
        uri: uri.href,
        text: "Flux is a decentralized computational network providing high-performance computing services. The main token is FLUX.",
        mimeType: "text/plain"
      }]
    })
  );
  
  // Filecoin blockchain info resource
  server.resource(
    "filecoin-info",
    "filecoin://info",
    async (uri) => ({
      contents: [{
        uri: uri.href,
        text: "Filecoin is a decentralized storage network designed to store humanity's most important information. The main token is FIL.",
        mimeType: "text/plain"
      }]
    })
  );
  
  // Blockchain transaction resource template
  server.resource(
    "transaction",
    new ResourceTemplate("{blockchain}://tx/{txid}", { list: undefined }),
    async (uri, { blockchain, txid }) => {
      try {
        // Different handling based on blockchain
        let txData;
        switch (blockchain) {
          case "flux":
            txData = `Flux transaction ${txid}`;
            break;
          case "filecoin":
            txData = `Filecoin transaction ${txid}`;
            break;
          case "liberland":
            txData = `Liberland transaction ${txid}`;
            break;
          case "chainlink":
            txData = `Chainlink transaction ${txid}`;
            break;
          case "arweave":
            txData = `Arweave transaction ${txid}`;
            break;
          case "iota":
            txData = `IOTA transaction ${txid}`;
            break;
          default:
            return {
              contents: [{
                uri: uri.href,
                text: `Unsupported blockchain: ${blockchain}`,
                mimeType: "text/plain"
              }]
            };
        }
        
        return {
          contents: [{
            uri: uri.href,
            text: txData,
            mimeType: "text/plain"
          }]
        };
      } catch (error) {
        return {
          contents: [{
            uri: uri.href,
            text: `Error retrieving transaction data: ${error instanceof Error ? error.message : String(error)}`,
            mimeType: "text/plain"
          }]
        };
      }
    }
  );
  
  // Blockchain wallet address resource template
  server.resource(
    "wallet",
    new ResourceTemplate("{blockchain}://wallet/{address}", { list: undefined }),
    async (uri, { blockchain, address }) => {
      try {
        // Different handling based on blockchain
        let walletData;
        switch (blockchain) {
          case "flux":
            walletData = `Flux wallet ${address}`;
            break;
          case "filecoin":
            walletData = `Filecoin wallet ${address}`;
            break;
          case "liberland":
            walletData = `Liberland wallet ${address}`;
            break;
          case "chainlink":
            walletData = `Chainlink wallet ${address}`;
            break;
          case "arweave":
            walletData = `Arweave wallet ${address}`;
            break;
          case "iota":
            walletData = `IOTA wallet ${address}`;
            break;
          default:
            return {
              contents: [{
                uri: uri.href,
                text: `Unsupported blockchain: ${blockchain}`,
                mimeType: "text/plain"
              }]
            };
        }
        
        return {
          contents: [{
            uri: uri.href,
            text: walletData,
            mimeType: "text/plain"
          }]
        };
      } catch (error) {
        return {
          contents: [{
            uri: uri.href,
            text: `Error retrieving wallet data: ${error instanceof Error ? error.message : String(error)}`,
            mimeType: "text/plain"
          }]
        };
      }
    }
  );
}
```

**src/blockchain-integrations/prompts.ts**
```typescript
import { McpServer } from "@modelcontextprotocol/sdk/server/mcp.js";
import { z } from "zod";

export function setupBlockchainPrompts(server: McpServer) {
  // Register blockchain prompts
  
  // Transaction analysis prompt
  server.prompt(
    "analyze-transaction",
    "Analyzes a blockchain transaction",
    { 
      blockchain: z.enum(["flux", "filecoin", "liberland", "chainlink", "arweave", "iota"]),
      txid: z.string()
    },
    ({ blockchain, txid }) => ({
      messages: [{
        role: "user",
        content: {
          type: "text",
          text: `Please analyze this ${blockchain} transaction: ${txid}. What type of transaction is it? What are the key details about it?`
        }
      }]
    })
  );
  
  // Wallet balance inquiry prompt
  server.prompt(
    "check-wallet-balance",
    "Checks the balance of a blockchain wallet",
    { 
      blockchain: z.enum(["flux", "filecoin", "liberland", "chainlink", "arweave", "iota"]),
      address: z.string()
    },
    ({ blockchain, address }) => ({
      messages: [{
        role: "user",
        content: {
          type: "text",
          text: `Please check the balance of this ${blockchain} wallet address: ${address}. What is the current balance and transaction history?`
        }
      }]
    })
  );
  
  // Blockchain comparison prompt
  server.prompt(
    "compare-blockchains",
    "Compares different blockchain technologies",
    { 
      blockchain1: z.enum(["flux", "filecoin", "liberland", "chainlink", "arweave", "iota"]),
      blockchain2: z.enum(["flux", "filecoin", "liberland", "chainlink", "arweave", "iota"])
    },
    ({ blockchain1, blockchain2 }) => ({
      messages: [{
        role: "user",
        content: {
          type: "text",
          text: `Please compare ${blockchain1} and ${blockchain2} blockchains. What are the key differences in terms of technology, use cases, consensus mechanisms, and token economics?`
        }
      }]
    })
  );
}
```

**src/api/index.ts**
```typescript
import express from "express";
import cors from "cors";
import { apiRoutes } from "./routes.js";

export function setupAPIServer(port: number) {
  const app = express();
  
  // Middleware
  app.use(cors());
  app.use(express.json());
  
  // API routes
  app.use("/api", apiRoutes);
  
  // Serve frontend
  app.use(express.static("frontend/build"));
  
  // Start server
  app.listen(port, () => {
    console.error(`API server listening on port ${port}`);
  });
}
```

**src/api/routes.ts**
```typescript
import express from "express";

const router = express.Router();

// Wallet management
router.get("/wallets", (req, res) => {
  // Get saved wallets
  res.json({ wallets: [] });
});

router.post("/wallets", (req, res) => {
  // Save a new wallet
  const { blockchain, address } = req.body;
  // Implementation here
  res.json({ success: true });
});

// Node status
router.get("/nodes/status", (req, res) => {
  // Get status of all blockchain nodes
  res.json({
    nodes: [
      { blockchain: "flux", status: "running", syncStatus: "98%" },
      { blockchain: "filecoin", status: "running", syncStatus: "92%" },
      { blockchain: "liberland", status: "running", syncStatus: "100%" },
      { blockchain: "chainlink", status: "running", syncStatus: "99%" },
      { blockchain: "arweave", status: "running", syncStatus: "95%" },
      { blockchain: "iota", status: "running", syncStatus: "97%" }
    ]
  });
});

export const apiRoutes = router;
```

### 4.3 Individual Blockchain Implementation

Let's create an example implementation for one of the blockchains (Flux):

**src/blockchain-integrations/flux.ts**
```typescript
import axios from "axios";
import { z } from "zod";
import { BlockchainTool } from "./types.js";

// Define Flux node URL
const FLUX_NODE_URL = "http://blockchain-node-1:16127";

export const fluxTools: BlockchainTool[] = [
  {
    name: "flux-balance",
    description: "Get the balance of a Flux wallet",
    schema: {
      address: z.string().describe("Flux wallet address")
    },
    handler: async ({ address }) => {
      try {
        const response = await axios.get(`${FLUX_NODE_URL}/explorer/balance/${address}`);
        
        return {
          content: [{
            type: "text",
            text: JSON.stringify(response.data, null, 2)
          }]
        };
      } catch (error) {
        return {
          content: [{
            type: "text",
            text: `Error retrieving Flux balance: ${error instanceof Error ? error.message : String(error)}`
          }],
          isError: true
        };
      }
    }
  },
  {
    name: "flux-transaction",
    description: "Get details of a Flux transaction",
    schema: {
      txid: z.string().describe("Flux transaction ID")
    },
    handler: async ({ txid }) => {
      try {
        const response = await axios.get(`${FLUX_NODE_URL}/explorer/tx/${txid}`);
        
        return {
          content: [{
            type: "text",
            text: JSON.stringify(response.data, null, 2)
          }]
        };
      } catch (error) {
        return {
          content: [{
            type: "text",
            text: `Error retrieving Flux transaction: ${error instanceof Error ? error.message : String(error)}`
          }],
          isError: true
        };
      }
    }
  },
  {
    name: "flux-wallet-info",
    description: "Get information about a Flux wallet",
    schema: {
      address: z.string().describe("Flux wallet address")
    },
    handler: async ({ address }) => {
      try {
        const [balanceRes, txHistoryRes] = await Promise.all([
          axios.get(`${FLUX_NODE_URL}/explorer/balance/${address}`),
          axios.get(`${FLUX_NODE_URL}/explorer/transactions/${address}`)
        ]);
        
        return {
          content: [{
            type: "text",
            text: JSON.stringify({
              address,
              balance: balanceRes.data,
              transactions: txHistoryRes.data.slice(0, 10) // Return only the 10 most recent transactions
            }, null, 2)
          }]
        };
      } catch (error) {
        return {
          content: [{
            type: "text",
            text: `Error retrieving Flux wallet info: ${error instanceof Error ? error.message : String(error)}`
          }],
          isError: true
        };
      }
    }
  }
];
```

**src/blockchain-integrations/types.ts**
```typescript
import { z, ZodType } from "zod";

export interface BlockchainTool {
  name: string;
  description: string;
  schema: Record<string, ZodType>;
  handler: (args: Record<string, any>) => Promise<{
    content: Array<{
      type: string;
      text: string;
    }>;
    isError?: boolean;
  }>;
}
```

## 5. Blockchain Node Integration

This section focuses on how to integrate with each specific blockchain node.

### 5.1 Node Configuration

Each blockchain node should be configured with:

- Appropriate API endpoints enabled
- Security measures in place
- Proper storage allocation
- Backup procedures

### 5.2 Node APIs

Each blockchain node exposes different APIs. Here's a template for implementing the MCP blockchain tools for each:

**Flux**
- REST API: Port 16127
- Key functionality: Wallet balances, transaction history, node status

**Filecoin**
- Lotus API: Port 1234
- Key functionality: Storage deals, retrieval, network metrics

**Liberland**
- REST API: Port 26657
- Key functionality: Governance voting, transaction history

**Chainlink**
- REST API: Port 6688
- Key functionality: Oracle data feeds, automation status

**Arweave**
- HTTP API: Port 1984
- Key functionality: Permanent storage, transaction verification

**IOTA**
- REST API: Port 14265
- Key functionality: Transaction validation, Tangle exploration

## 6. Monitoring and Maintenance

### 6.1 Prometheus Metrics

The MCP server exposes metrics for monitoring at `http://<server-ip>:8000/metrics`. Key metrics include:

- System metrics (CPU, memory, disk usage)
- Node status metrics
- Transaction processing metrics
- API request metrics

### 6.2 Grafana Dashboards

The setup includes a Grafana instance at `http://<server-ip>:3000` with pre-configured dashboards:

- Node Status Dashboard
- Blockchain Performance Dashboard
- System Resources Dashboard

### 6.3 Logging

All components output logs to standard locations:

- MCP Server: `/var/log/mcp-server.log`
- Blockchain Nodes: `/opt/blockchain-data/<node>/logs`
- API Server: `/var/log/blockchain-api.log`

### 6.4 Backup Procedures

Regular backups should be implemented:

- Daily snapshot of VM state
- Blockchain node data directories backup
- Configuration files backup
- Database backup (if applicable)

## 7. Web Interface & API Development

### 7.1 Frontend (React)

The web interface provides:

- Dashboard with node status overview
- Wallet management interface
- Transaction explorer
- Blockchain metrics visualization

**Key Components:**
- Node Status Panel
- Wallet Registration Form
- Transaction Search
- Metrics Charts

### 7.2 Backend (Node.js)

The API server provides endpoints for:

- Wallet management (CRUD operations)
- Node status monitoring
- Transaction lookup
- Metrics collection

**Security Measures:**
- JWT-based authentication
- Rate limiting
- Input validation
- HTTPS encryption
- CORS configuration

## 8. Security Considerations

### 8.1 Network Security

- All traffic should be encrypted using TLS
- Firewall rules should restrict access to specific ports
- VPN for administrative access
- Regular security audits

### 8.2 Node Security

- Non-root user execution
- Least privilege principle
- Regular security updates
- Intrusion detection

### 8.3 API Security

- Authentication for all endpoints
- Input validation and sanitization
- Rate limiting to prevent abuse
- CORS restrictions

## 9. Troubleshooting

### 9.1 Common Issues

**Node Connection Issues**
- Check firewall rules
- Verify network configuration
- Ensure nodes are running
- Check API access credentials

**MCP Server Issues**
- Check logs at `/var/log/mcp-server.log`
- Verify configuration files
- Restart service: `systemctl restart mcp-server`

**Monitoring Issues**
- Check Prometheus connectivity
- Verify Grafana datasource configuration
- Ensure metrics endpoints are accessible

### 9.2 Support Resources

- MCP Documentation: https://modelcontextprotocol.io/
- Blockchain Node Documentation:
  - Flux: https://runonflux.io/docs/
  - Filecoin: https://docs.filecoin.io/
  - Liberland: https://liberland.org/documents/
  - Chainlink: https://docs.chain.link/
  - Arweave: https://docs.arweave.org/
  - IOTA: https://wiki.iota.org/
