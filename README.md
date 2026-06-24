# Private Cloud Architect Skill

An AI skill for designing, evaluating, and optimizing on-premises and private cloud infrastructure — including classified and sensitive environments.

## Overview

This repository contains a structured skill definition that provides expert guidance on:

- **Private cloud platforms**: OpenStack, Nutanix, Proxmox, KubeVirt, VMware vSphere, bare-metal Kubernetes, hyperconverged infrastructure
- **Data center networking**: Cisco ACI (EPGs, contracts, service graphs), Nexus fabrics, VXLAN-EVPN, spine-leaf design
- **L4-L7 services**: F5 BIG-IP, Citrix NetScaler/ADC, Palo Alto firewalls, HAProxy, load balancer and firewall insertion into ACI
- **Classified/sensitive systems**: Air-gapped environments, cross-domain solutions, national security frameworks (Sikkerhetsloven, Säkerhetsskyddslagen, BSI IT-Grundschutz, NATO classification), military/defense cloud accreditation
- **Hybrid cloud**: On-prem to AWS/Azure/GCP connectivity, cloud bursting, sensitive data boundary design, Direct Connect/ExpressRoute
- **Automation**: Ansible, Terraform/OpenTofu, ArgoCD, Crossplane

## Repository Structure

| Path | Description |
|------|-------------|
| [`cloud-architect.md`](cloud-architect.md) | Standalone skill definition |
| [`private-cloud-architect/SKILL.md`](private-cloud-architect/SKILL.md) | Packaged skill definition for directory-based skill loading |
| [`INDEX.md`](INDEX.md) | Navigable table of contents |
| `dev/` | Performance evaluation and benchmark outputs |

## Usage

Load this skill in any AI assistant that supports the skill format. The skill provides architecture guidance, evaluation frameworks, and decision trees for private cloud infrastructure design.
