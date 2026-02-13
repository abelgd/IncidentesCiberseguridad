# Documentación de caso de uso - Abel García Domínguez
Detección de ataques de fuerza bruta SSH e ICMP
> Esta plantilla está cogida del pdf adjunto a la tarea sobre los casos de uso.

## Control de Cambios

| Fecha       | Actualización                    | Realizado por        | Autorizado por | Naturaleza del cambio                                      |
|------------|----------------------------------|----------------------|----------------|-------------------------------------------------------------|
| 12/02/2026 | Creación del caso de uso         | Abel García Domínguez| Profesor | Documento nuevo                                             |

---

## CONTENIDO

1. OBJETIVO  
2. ALCANCE  
3. FUENTES DE EVENTOS  
4. TIPO DE DATOS  
5. FLUJO LÓGICO  
6. NOTIFICACIÓN  
7. SEVERIDAD  
8. RECOMENDACIÓN  

---

## OBJETIVO

Detectar actividades relacionadas con **ataques de fuerza bruta SSH** y **escaneos ICMP** en la red monitorizada por el SIEM, correlando múltiples eventos de un mismo origen para generar alertas cuando se superen los umbrales definidos.

---

## ALCANCE

Este caso de uso tiene el siguiente alcance:

- **Servicios monitorizados**:
  - Servidores o máquinas con **SSH** accesible (TCP/22).
  - Hosts que responden a **ICMP echo-request/echo-reply**.
- **Tipos de actividad**:
  - Múltiples intentos de autenticación SSH fallidos desde una misma IP origen en un intervalo corto.
  - Alto volumen de paquetes ICMP desde una misma IP origen hacia uno o varios destinos (ping sweep / escaneo).

No se consideran:

- Fallos de autenticación aislados.
- Tráfico ICMP de monitorización puntual (por ejemplo, un único `ping`).

---

## FUENTES DE EVENTOS

- **IDS (Snort) en modo IDS**  
  - Genera alertas cuando detecta patrones de fuerza bruta SSH y escaneos ICMP.

---

## TIPO DE DATOS

Para la creación de este caso de uso, se requiere el análisis de los siguientes tipos de información:

| Plataforma          | Tipo de log                                                                 |
|---------------------|-----------------------------------------------------------------------------|
| IDS (Snort)| Alertas de reglas de **SSH brute force** y de **ICMP scan/flood**          |

Campos relevantes en los eventos:

- `@timestamp` (fecha y hora del evento).  
- `src_ip` (IP origen).  
- `dest_ip` (IP destino).  
- `alert.signature` o mensaje de log (tipo de alerta).  
- `proto` (TCP/UDP/ICMP), `dest_port` (22 para SSH).

---

## FLUJO LÓGICO

1. El IDS analiza el tráfico de red y genera eventos cuando:
   - Detecta intentos sucesivos de autenticación SSH fallida desde una misma IP origen.
   - Detecta un número anómalo de paquetes ICMP (eco) desde una misma IP origen.

2. Los eventos del IDS se envían al SIEM (ELK) y se almacenan en un índice (`ids-*`).

3. El SIEM aplica reglas de correlación / consultas que implementan los umbrales:

   - **SSH fuerza bruta**  
     - Se gatilla una alerta cuando:
       - Para una misma `src_ip` se registran **≥ 5 alertas de SSH** en un intervalo de **5 minutos**, dirigidas al mismo host o a hosts del mismo segmento.
   
   - **Escaneo ICMP**  
     - Se gatilla una alerta cuando:
       - Una misma `src_ip` genera **≥ 50 eventos ICMP** en menos de **60 segundos**, hacia uno o varios destinos.

4. La alerta de caso de uso incluye:

   - IP origen, IP destino (o rango).  
   - Tipo de ataque (`SSH brute force` / `ICMP scan`).  
   - Número de eventos que han disparado la condición.  
   - Intervalo temporal en el que se ha detectado la actividad.

---

## NOTIFICACIÓN

- Las alertas generadas por este caso de uso se visualizarán en el **cuadro de mandos del SIEM** en un panel específico de “Fuerza Bruta SSH/ICMP”.
- En un entorno real, las notificaciones se enviarían por correo a la cuenta del equipo de seguridad, por ejemplo:

  - `soc@instituto.local`  
  - `operador@cliente.com`

- El formato estándar de notificación incluye:

  - Fecha y hora de detección.  
  - IP origen e IP destino.  
  - Tipo de ataque (SSH / ICMP).  
  - Número de eventos y umbral superado.  
  - Enlace a la vista detallada en el SIEM.

---

## SEVERIDAD

| Alerta / Reporte                                  | Severidad |
|---------------------------------------------------|-----------|
| Fuerza bruta SSH desde IP externa                 | Alta      |
| Fuerza bruta SSH desde IP interna                 | Crítica   |
| Escaneo ICMP de múltiples hosts                   | Media     |

---

## RECOMENDACIÓN

- **Endurecimiento de SSH**:
  - Restringir el acceso a SSH mediante listas blancas de IP o VPN.
  - Habilitar autenticación por claves públicas y deshabilitar contraseñas débiles.
  - Cambiar el puerto por defecto cuando sea posible (medida adicional, no principal).

- **Mecanismos de bloqueo automático**:
  - Implementar herramientas como `fail2ban` o reglas de firewall que bloqueen IPs que superen los umbrales definidos por este caso de uso.

- **Ajuste continuo del caso de uso**:
  - Revisar periódicamente las alertas generadas para:
    - Ajustar los umbrales (número de intentos y ventana temporal).
    - Añadir excepciones para sistemas de monitorización legítimos.
  - Documentar los incidentes reales para mejorar la correlación y reducir falsos positivos.

- **Buenas prácticas adicionales**:
  - Deshabilitar servicios remotos inseguros (Telnet, RSH).  
  - Mantener IDS, SIEM y sistemas operativos actualizados con los últimos parches.  
  - Integrar este caso de uso con otros (por ejemplo, detección de malware) para tener una visión más completa del ataque.

