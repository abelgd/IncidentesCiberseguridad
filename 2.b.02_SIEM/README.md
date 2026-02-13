# README.md —  Detección de incidentes IDS/IPS/SIEM

*Actividad:* 2.b.02 — Implementación y configuración de SIEM  
*Objetivo:* Desplegar un SIEM basado en ELK (Elasticsearch + Logstash + Kibana) y un agente (Nginx + Filebeat) para recolectar logs, integrarlos en Kibana y dejar el entorno listo para validar un caso de uso de detección (fuerza bruta SSH con Hydra).
*Autor:* Abel García Domínguez

## Estructura del proyecto

Este repositorio se entrega con *tres documentos* principales. Este README.md actúa como *índice* y resumen.

1) *Documentación de instalación y configuración*  
- Archivo: **[![Instalación](documentacion/instalacion_configuracion.md)]**
- Contenido: arquitectura (contenedores y red), despliegue ELK 7.16.3, construcción del agente nginx-filebeat, configuración de Filebeat y validación de ingesta en Kibana, troubleshooting (TLS/versiones, Filebeat detenido, etc.).

2) *Documento del caso de uso*  
- Archivo: **[![Caso de uso](documentacion/documentacion_casodeuso.md)]**
- Contenido: descripción del caso de uso “Fuerza bruta SSH”, objetivos de detección, activos, escenarios, fuentes de logs, reglas/indicadores esperados y criterios de verificación.

3) *Implementación del caso de uso*  
- Archivo: **[![Implementación caso de uso](documentacion/implementacion_casodeuso.md)]**
- Contenido: instalación de Snort 2.9.20 (desde fuente), configuración (snort.conf, local.rules), validación con ICMP y SSH, ejecución del ataque con scripts del profesorado (crear_claves, crear_usuarios, lanzar_hydra), problemas encontrados (regla dependiente de flow:established, registro de alertas) y solución final (ajuste de regla + ejecución de Snort con salida a fichero para ingesta con Filebeat).

## Resumen técnico

### Arquitectura
- *SIEM / SOC:* contenedor sebp/elk:7.16.3  
  - Kibana: 5601
  - Elasticsearch: 9200
  - Logstash (Beats input): 5044
- *Agente / Endpoint:* contenedor nginx-filebeat  
  - Nginx: 8080
  - Filebeat: envío de logs a Logstash
- *Atacante WSL Ubuntu*

### Flujo de datos
1. Nginx y Snort generan logs en ficheros (/var/log/nginx/*.log, /var/log/snort/alert).
2. Filebeat “harvestea” esos ficheros y envía eventos a Logstash (5044).
3. Logstash procesa y envía a Elasticsearch.
4. Kibana visualiza los eventos mediante un patrón filebeat-* en Discover.

## Evidencias
Las capturas/imagenes utilizadas en los documentos se encuentran en **[![Imágenes](documentacion/img)]** y se referencian desde cada documento.
