# Informe — Implementación del Caso de Uso en el SIEM (ELK)

*Caso de uso:* Detección de fuerza bruta SSH e ICMP (Snort → Filebeat → ELK)  
*Entorno:* Laboratorio con contenedores Docker (ELK + agente con Nginx/Filebeat + Snort)

---

## Control de cambios

| Fecha | Actualización | Realizado por | Autorizado por | Naturaleza del cambio |
|---|---|---|---|---|
| 17/02/2026 | Implementación del caso de uso en el SIEM | Abel García Domínguez | Profesor | Documento nuevo |

---

## 1. Objetivo de la implementación

Implementar en el SIEM el caso de uso definido en [la documentación del caso de uso](documentacion_casodeuso.md), asegurando que:

1. Snort genera alertas para ICMP y SSH.
2. Las alertas se escriben a fichero.
3. Filebeat ingesta el fichero y envía a Logstash.
4. Los eventos quedan indexados y visibles en Kibana.
5. Se valida el comportamiento ante tráfico real (ping e intento de fuerza bruta SSH con Hydra).

---

## 2. Requisitos previos

- Contenedor *ELK 7.16.x* levantado con puertos expuestos: 5601 (Kibana), 9200 (Elasticsearch), 5044 (Logstash Beats input).
- Contenedor *agente* con:
  - Snort 2.9.20 instalado y con snort.conf funcional.
  - Filebeat configurado para leer /var/log/snort/alert y enviar a Logstash.
- Red Docker común (elk-red) para conectividad entre agente y ELK (resolución por IP o por nombre).

---

## 3. Implementación de la detección (Snort)

### 3.1 Reglas en local.rules

Se implementaron reglas locales para:

- *ICMP*: corroborar rápidamente que Snort ve tráfico ICMP hacia el endpoint.
- *SSH*: detectar tráfico TCP hacia puerto 22 y permitir correlación/validación del caso de fuerza bruta.

> *Nota:* inicialmente la regla SSH dependía de flow:established, lo que limitaba la detección a conexiones TCP ya establecidas y dificultó la corroboración de intentos; se modificó para validar también intentos (ver apartado 3.3).

### 3.2 Ejecución de Snort con salida a fichero

Para que Filebeat pudiera recolectar alertas, Snort se ejecutó en modo que escribe alertas en formato fast dentro del directorio de logs:

```bash
snort -A fast -q -c /etc/snort/snort.conf -i eth0 -k none -l /var/log/snort
```

- -A fast: genera alertas en formato "fast" (apto para ingesta desde fichero).
- -l /var/log/snort: fuerza el directorio donde se crea/actualiza el fichero alert.
- -k none: workaround de laboratorio para evitar problemas de checksum y asegurar visibilidad en entornos virtualizados.

### 3.3 Incidencia: la regla solo detectaba con flow:established

*Problema:* durante la prueba con Hydra se observó que la regla SSH solo disparaba si el flujo era "established", por lo que no se registraban algunos intentos como se esperaba.

*Acción:* se modificó la regla (relajando o eliminando la dependencia estricta de flow:established) para que los intentos quedaran reflejados y poder corroborar el caso de uso durante la PoC.

---

## 4. Implementación de la ingesta (Filebeat → Logstash → Elasticsearch)

### 4.1 Fuente de logs (input)

Se configuró Filebeat para seguir el fichero generado por Snort:

- Ruta: /var/log/snort/alert

La finalidad es que cada línea nueva escrita por Snort se convierta en un evento que Filebeat envíe a Logstash.

### 4.2 Envío al SIEM (output a Logstash)

Se configuró output.logstash apuntando al Beats input (5044) del contenedor ELK.

> *Recomendación:* mantener coherencia en la conectividad (usar o bien IP fija del ELK, o bien nombre del contenedor) para evitar errores de resolución.

### 4.3 Verificación de Filebeat (sanidad de config)

Antes de validar en Kibana, se realizaron comprobaciones para asegurar configuración válida y conectividad correcta hacia Logstash.

Comandos usados en laboratorio:

```bash
filebeat test config -e
filebeat test output -e
```

---

## 5. Implementación en Kibana (visualización y verificación)

### 5.1 Index pattern

En Kibana se creó/seleccionó el patrón de índices para explorar eventos:

filebeat-*


### 5.2 Validación en Discover

Se validó que el pipeline estaba operativo comprobando que, tras generar tráfico, aparecían eventos nuevos en Discover con timestamps recientes, y que el campo message contenía la alerta "fast" de Snort.

---

## 6. Pruebas de funcionamiento (PoC)

### 6.1 Prueba ICMP (visibilidad del IDS)

- Se generó tráfico ICMP (ping) hacia el endpoint.
- Se comprobó que Snort escribía en /var/log/snort/alert.
- Se verificó en Kibana que aparecían eventos nuevos asociados a ICMP.

### 6.2 Prueba fuerza bruta SSH (Hydra)

Se siguió el procedimiento de la guía mediante los scripts:

- crear_claves
- crear_usuarios
- lanzar_hydra

*Validación:*

- Snort genera alertas asociadas a actividad SSH repetida.
- Filebeat ingesta las alertas.
- Kibana muestra eventos nuevos y permite corroborar el caso de uso.

*Resultado:* tras el ajuste del comando de Snort (salida a fichero fast) y la modificación de la regla para no depender exclusivamente de flow:established, se consiguió que los intentos quedaran registrados y visibles en el SIEM.
