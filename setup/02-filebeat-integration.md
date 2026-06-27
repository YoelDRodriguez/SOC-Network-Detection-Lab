# **02 — VPC Traffic Mirroring + Filebeat Integration**

Este documento cubre la Semana 2 del proyecto: la configuración de VPC Traffic
Mirroring para replicar tráfico de red hacia el network-sensor, y la
integración de Suricata + Zeek con Wazuh vía Filebeat.

Esta semana incluyó troubleshooting real no trivial — incompatibilidades de
versión entre Filebeat y OpenSearch, un servicio crasheado sin supervisión,
y un error operacional en la instancia equivocada. Todo está documentado
porque refleja el proceso real de integrar herramientas en un SOC.

---
## **Prerrequisitos**

1. network-sensor con Suricata 8.0.5 y Zeek 8.2.0 operativos (ver 01-network-sensor.md)
2. wazuh-server con Wazuh 4.14.5 corriendo (Manager, Indexer, Dashboard)
3. Instancias linux-victim y windows-victim corriendo (requerido para crear Mirror Sessions)

---
## 1. VPC Traffic Mirroring

### 1.1 Crear el Mirror Target

El Mirror Target es la ENI del network-sensor — el destino del tráfico replicado.
VPC → Traffic Mirroring → Mirror Targets → Create traffic mirror target:
| Campo | Valor |
|---|---|
| Name Tag | network-sensor-mirror-target |
| Target Type |Network Interface |
| Target | ENI del network-sensor |

### 1.2 Crear el Mirror Filter

Define qué tráfico se replica — para este lab, todo el tráfico en ambas direcciones.
VPC → Traffic Mirroring → Mirror Filters → Create traffic mirror filter:
| Campo | Valor |
|---|---|
| Name Tag | lab-mirror-filter |
| Inbound Rule | Number 100, accept, all protocols, 0.0.0.0/0 → 0.0.0.0/0 |
| Outbound Rule | Number 100, accept, all protocols, 0.0.0.0/0 → 0.0.0.0/0 |

### 1.3 Crear las Mirror Sessions

Requisito crítico: las instancias víctima deben estar corriendo para crear la session — no se puede mirror una ENI de una instancia detenida.
| Campo | linux-victim-mirror | windows-victim-mirror |
|---|---|---|
| Mirror source | ENI de linux-victim | ENI de windows-victim |
| Mirror target | network-sensor-mirror-target | network-sensor-mirror-target |
| Filter | lab-mirror-filter | lab-mirror-filter |
|Session number | 1 | 2 |

### 1.4 Verificar que el tráfico llega
Desde el network-sensor:

```xml
 sudo tcpdump -i ens5 udp port 4789 -c 10
```

Resultado esperado: paquetes VXLAN encapsulando el tráfico real de las víctimas. En la prueba de este lab, el tráfico capturado mostró conexiones de los agentes Wazuh hacia el puerto 1514 del Manager — confirmando que el mirroring captura tráfico real de producción del lab, no solo tráfico sintético.
```xml
  IP ip-172-31-44-223...65488 > ip-172-31-69-43...4789: VXLAN, flags [I], vni 12994019
  IP ip-172-31-44-223...49716 > ip-172-31-35-177...1514: Flags [P.], length 254
```
Dos VNI distintos confirman ambas sessions activas simultáneamente.
---

## 2. Instalación de Filebeat

Decisión crítica: qué versión de Filebeat usar
No usar el Filebeat oficial de Elastic 8.x contra Wazuh Indexer (OpenSearch).
Wazuh 4.14.5 usa OpenSearch 7.10.2 como motor de indexado, no Elasticsearch.
Filebeat 8.x es un producto de Elastic con lógica interna que verifica
compatibilidad consultando el endpoint _license — un endpoint exclusivo de
Elasticsearch que no existe en OpenSearch. Esto produce un error fatal de
conexión sin importar la configuración:

```xml
ERROR Connection marked as failed: could not connect to a compatible version
of Elasticsearch: 400 Bad Request: invalid_index_name_exception [_license]
```
Ningún flag de configuración (allow_older_versions, etc.) resuelve esto en
Filebeat 8.x contra OpenSearch 7.10.2.

Solución: usar Filebeat 7.10.2 desde el repositorio oficial de Wazuh — es
la rama OSS sin la lógica de licenciamiento de Elastic, y coincide en versión
exacta con el OpenSearch del wazuh-server.

### 2.1 Instalar Filebeat desde el repositorio de Wazuh
```xml
curl -s https://packages.wazuh.com/key/GPG-KEY-WAZUH \
  | sudo gpg --no-default-keyring --keyring gnupg-ring:/usr/share/keyrings/wazuh.gpg --import
sudo chmod 644 /usr/share/keyrings/wazuh.gpg

echo "deb [signed-by=/usr/share/keyrings/wazuh.gpg] https://packages.wazuh.com/4.x/apt/ stable main" \
  | sudo tee /etc/apt/sources.list.d/wazuh.list

sudo apt update
sudo apt install -y filebeat
```
Verificar Versión
```xml
filebeat version
# Esperado: filebeat version 7.10.2
```
>Si ya tenías instalado el Filebeat de Elastic 8.x, desinstalalo primero con
>sudo apt remove -y filebeat y eliminá su repositorio
>(/etc/apt/sources.list.d/elastic-8.x.list) antes de instalar el de Wazuh.
---

## 3. Configurar Filebeat en el network-sensor
```xml
sudo nano /etc/filebeat/filebeat.yml
```
Contenido completo
```xml
filebeat.inputs:
  - type: log
    enabled: true
    paths:
      - /var/log/suricata/eve.json
    fields:
      source_sensor: suricata
      lab: soc-network-detection-lab
    fields_under_root: true

  - type: log
    enabled: true
    paths:
      - /opt/zeek/logs/current/conn.log
      - /opt/zeek/logs/current/dns.log
      - /opt/zeek/logs/current/notice.log
    fields:
      source_sensor: zeek
      lab: soc-network-detection-lab
    fields_under_root: true

output.elasticsearch:
  hosts: ["<IP-PRIVADA-WAZUH-SERVER>:9200"]
  protocol: https
  username: "admin"
  password: "<PASSWORD-WAZUH-INDEXER>"
  ssl.verification_mode: none
  index: "filebeat-network-sensor-%{+yyyy.MM.dd}"

setup.template.name: "filebeat-network-sensor"
setup.template.pattern: "filebeat-network-sensor-*"
setup.ilm.enabled: false

logging.level: info
logging.to_files: true
logging.files:
  path: /var/log/filebeat
  name: filebeat
  keepfiles: 7
```
> El campo source_sensor es lo que permite diferenciar y filtrar eventos de
> Suricata vs Zeek dentro del mismo índice — esencial para las reglas de
> correlación de la Semana 5.

### 3.1 Habilitar el puerto 9200 en el wazuh-server

Por defecto, Wazuh Indexer escucha solo en 127.0.0.1 — el network-sensor
no puede alcanzarlo. Dos ajustes necesarios:

a) Security Group del wazuh-server — permitir 9200 desde el sensor:
| Tipo | Puerto | Origen | Descripción | 
|---|---|---|---|
| Custom TCP| 9200 | IP privada del network-sensor/32 | Filebeat del network-sensor |

b) Bind del Wazuh Indexer a todas las interfaces:
En el wazuh-server:
```xml
sudo nano /etc/wazuh-indexer/opensearch.yml
```
```xml
network.host: 0.0.0.0
```
```xml
sudo systemctl restart wazuh-indexer
```
### 3.2 Probar la conexión antes de iniciar el servicio
```xml
sudo filebeat test output
```
Resultado esperado:
```xml
talk to server... OK
version: 7.10.2
```
### 3.3 Iniciar Filebeat
```xml
sudo systemctl enable filebeat
sudo systemctl start filebeat
```
---

## 4. Verificación de datos en Wazuh
### 4.1 Confirmar el índice por API
```xml
curl -k -u admin:'<password>' "https://<IP-wazuh-server>:9200/_cat/indices?v" | grep filebeat-network
```
Resultado esperado:
```xml
yellow open filebeat-network-sensor-2026.06.24   ...   17984   ...   43.1mb
```
### 4.2 Crear el index pattern en el Dashboard

Menú → Dashboards Management → Index Patterns → Create index pattern
| Campo | Valor |
|---|---|
| Name | filebeat-network-sensor-* |
| Time | field@timestamp |

### 4.3 Confirmar ambas fuentes en Discover

> Menú → Discover → seleccionar filebeat-network-sensor-*
> Filtrar por source_sensor: suricata — deberían aparecer campos event_type, alert.
> Filtrar por source_sensor: zeek — deberían aparecer campos id.orig_h, id.resp_h, proto.
---

## Troubleshooting
| Sintoma | Causa | Solución |
|---|---|---|
|dial tcp ...:9200: i/o timeout | Security group del wazuh-server bloqueando el puerto 9200 | Agregar regla de entrada TCP 9200 desde la IP del network-sensor |
| connection refused en el puerto 9200 | Wazuh Indexer escuchando solo en 127.0.0.1 | Cambiar network.host a 0.0.0.0 en opensearch.yml y reiniciar el servicio |
| invalid_index_name_exception [license] | Filebeat 8.x (Elastic) incompatible con OpenSearch — verifica un endpoint que no existe | Reinstalar con Filebeat 7.10.2 del repositorio de Wazuh |
| El filebeat.yml del wazuh-server quedó apuntando a IPs o configuración incorrecta | Edición accidental del archivo en la instancia equivocada (confundir SSH sessions) | Verificar siempre hostname antes de editar configuración; reconstruir con filebeat.inputs apuntando a alerts.json + wazuh-template.json |
| Scheduled task no genera evento aunque el canal de logging está enabled: true | Canal recién habilitado no estaba "caliente" — Windows requiere reinicio del servicio para empezar a loguear activamente | Restart-Service Schedule -Force antes de generar el evento de prueba |
| source_sensor: zeek no aparece en el Dashboard, solo Suricata | Proceso de Zeek crasheado (zeekctl status → crashed), conn.log congelado en el tiempo | zeekctl deploy para relevantar; crear unit file systemd con Restart=on-failure para que no vuelva a quedar caído sin supervisión |
| Zeek sin persistencia tras reinicios de la instancia | zeekctl no se integra con systemd por defecto | Crear /etc/systemd/system/zeek.service con ExecStart=/opt/zeek/bin/zeekctl deploy y systemctl enable zeek |

---
## Hallazgos clave de la semana
1. Filebeat de Elastic y OpenSearch no son intercambiables — aunque
ambos hablan el "protocolo Elasticsearch", Filebeat 8.x rechaza
activamente servidores OpenSearch por un chequeo de licencia. La solución
no es configuración, es usar la distribución correcta del software.

3. zeekctl deploy no es persistente por sí solo — un sensor de red en
producción necesita supervisión activa del proceso (systemd, supervisord,
o similar). Sin eso, un crash silencioso puede pasar desapercibido por
días, exactamente lo que ocurrió en este lab.

4. Los canales de logging de Windows pueden estar "enabled" sin estar
activos. wevtutil gl reportando enabled: true no garantiza que el
canal esté escribiendo eventos nuevos — un reinicio del servicio asociado
puede ser necesario para que el canal empiece a loguear tras habilitars

5. Trabajar con múltiples instancias SSH simultáneas requiere disciplina
de verificación. Confundir terminales causó la sobreescritura accidental
de la configuración de Filebeat del wazuh-server — el hábito de correr
hostname antes de cualquier cambio de configuración evita este tipo de
incidente.











