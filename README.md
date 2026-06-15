## SOC Network Detection Lab — Suricata + Zeek + Wazuh
> Laboratorio de detección de amenazas a nivel de red construido sobre AWS,
> integrando Suricata 8.0.5 y Zeek 8.2.0 como sensores de red con Wazuh 4.14.5
> como SIEM central — extiende el SOC Home Lab V1
> agregando visibilidad completa de tráfico de red.

---
## Objetivo

Demostrar habilidades de detección a nivel de red mediante la integración de
sensores IDS/NSM con un SIEM existente, cubriendo técnicas reales del framework
MITRE ATT&CK que el endpoint detection del V1 no podía detectar por sí solo.

---
## ¿Por qué este proyecto?

El SOC Home Lab V1 demostró detección basada en logs de host — agentes Wazuh,
Sysmon, y eventos del sistema operativo. Lo que V1 no cubre es visibilidad de
tráfico de red: escaneos de puertos, comunicación C2, exfiltración, movimiento
lateral. Este lab cierra esa brecha.

Un SOC real necesita ambas capas. Este proyecto demuestra que sé construirlas
y correlacionarlas.

---
## Arquitectura
![Architecture](infrastructure/architecture.png)

| Componente | Tecnología | Specs |
|---|---|---|
| SIEM | Wazuh | 4.14.5EC2 t3.small, Ubuntu 24.04, us-east-1 |
| Sensor de red | Suricata 8.0.5 + Zeek 8.2.0 | EC2 t3.small, Ubuntu 24.04 |
| Víctima Linux | Ubuntu 24.04 | EC2 t3.micro, Wazuh Agent|
| Víctima Windows | Windows Server 2022 | EC2 t3.micro, Sysmon + Wazuh Agent |
| Atacante Kali | Linux 2025.4 | VirtualBox local|
| Captura de tráfico | VPC Traffic Mirroring | VXLAN → ENI del sensor | 
| Integración SIEM | Filebeat | eve.json + Zeek logs → Wazuh Indexer |

---
## Cobertura MITRE ATT&CK
![MITRE Coverage](infrastructure/mitre-coverage.svg)

| Técnica | Táctica | Sensor | Detección | Regla Custom |
|---|---|---|---|---|
| T1046 Network Service Scanning | Reconnaissance | Suricata + Zeek | En progreso | Planned |
| T1071.001 C2 over HTTP/S | Command & Control | Suricata + Zeek | En progreso | Planned |

---

## Stack técnico
Sensor de red — network-sensor
| Herramienta | Versión | Rol | 
|---|---|---|
| Suricata | 8.0.5IDS | detección basada en firmas + anomalías |
| Zeek | 8.2.0NSM | análisis de protocolos y logs de red |
| Filebeat | Pending |Transporte de logs al Wazuh Indexer |
| Emerging Threats Open | 66,430 reglas | Ruleset de detección para Suricata |

---
## Infraestructura AWS
| Recurso | Detalle |
|---|---|
| Región | us-east-1 (N. Virginia) |
| VPC | Default VPC |
| Traffic Mirroring | ENI linux-victim + windows-victim → network-sensor |
| Security Group | TCP 22 (SSH) + UDP 4789 (VXLAN) |

---
## Decisiones de arquitectura

VPC Traffic Mirroring sobre sensor inline o agente local
AWS replica el tráfico de las instancias víctima a nivel de ENI hacia el
network-sensor vía VXLAN. Esta arquitectura replica un entorno empresarial
real donde el sensor de red es un componente dedicado e independiente de
los hosts monitoreados. Las alternativas (sensor inline o Suricata/Zeek
corriendo en las mismas víctimas) no demuestran arquitectura de SOC real.

Filebeat sobre rsyslog para integración con Wazuh
Wazuh tiene integración nativa con Filebeat — los módulos predefinidos
mapean automáticamente los campos de eve.json y los logs de Zeek a los
índices del Wazuh Indexer sin transformaciones manuales. Rsyslog requeriría
decoders custom y produce campos menos limpios en el Dashboard.
