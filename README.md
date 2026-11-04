# EVE-NG — Laboratorios NOC y bitácora de troubleshooting

Repositorio de prácticas reales de infraestructura de operador de telecomunicaciones: administración de redes, Linux, SSH, AAA y, más adelante, FTTH/GPON. **Los errores también se documentan**, incluidos los pequeños despistes que ayudaron a entender cómo funcionan las herramientas.

## Índice

| Contenido | Qué incluye |
|---|---|
| [01 · HOP + FreeRADIUS + MikroTik AAA](labs/01-aaa-hop-freeradius-mikrotik/README.md) | Topología, direcciones, configuraciones de referencia y pruebas verificadas |
| [Incidencias y errores de la Parte I](labs/01-aaa-hop-freeradius-mikrotik/TROUBLESHOOTING.md) | Diagnóstico, mensajes exactos y correcciones, sin ocultar los errores humanos |
| [Operación y cierre de la Parte I](labs/01-aaa-hop-freeradius-mikrotik/OPERACIONES.md) | Altas y bajas de operadores, logs, seguridad y tareas pendientes |
| [Issue #1 — EVE-NG / VMware NO-CARRIER](../../issues/1) | Incidencia inicial de enlace de gestión en VMware Workstation |

## Estado de los laboratorios

| Fase | Estado | Evidencia |
|---|---|---|
| I · NOC Access & AAA | **Funcional y probado**, cierre administrativo parcial | SSH HOP→CHR, FreeRADIUS Access-Accept/Reject, grupo restringido, Accounting Start/Stop |
| II · FTTH/GPON con VOLTHA/BBSim | **Planificado para integración posterior** | No se documenta como desplegado dentro de este laboratorio AAA |
| III · Integración AAA + acceso GPON | Pendiente | Reutilizar HOP, control de permisos, registro y monitorización |

## Laboratorio AAA — visión general

```text
Windows + MobaXterm
    |
    | VMnet2 / Cloud1: 172.20.99.0/24
    v
HOP-SERVER (Debian)       ens4 172.20.99.20 / ens3 10.10.10.20
    |
    | NET-MGMT-10: 10.10.10.0/24
    v
MikroTik CHR (RouterOS)   ether1 10.10.10.1 / ether2 10.10.30.1
    |
    | NET-SERVICES-30: 10.10.30.0/24
    v
RADIUS-SERVER (Debian)    ens3 10.10.30.10
    └─ FreeRADIUS 3.2.7: Authentication · Authorization · Accounting
```

**Demostrado:** un operador autenticado centralmente puede entrar por SSH como usuario restringido, leer información y ejecutar diagnósticos, pero no modificar la identidad del router. Los eventos Accounting Start/Stop se guardan en Debian.

## Seguridad y alcance

- **Nunca se publican contraseñas reales, claves SSH privadas ni secretos compartidos.** Todos los fragmentos de configuración usan marcadores de sustitución.
- Las direcciones IP son privadas y pertenecen al laboratorio virtual. No son endpoints públicos.
- Los fragmentos son reconstrucciones de la configuración **observada durante las pruebas**; no sustituyen un respaldo íntegro de dispositivos.
- **No introducir contraseñas en macros de MobaXterm o archivos de comandos sin protección.** Preferir bóveda y autenticación por claves cuando sea viable.
- La Parte I aún requiere snapshots, copias de seguridad definitivas y retirada controlada de la NIC temporal de Internet de RADIUS-SERVER.
- Nota de seguridad pendiente: comprobar que la contraseña de prueba expuesta anteriormente se haya revocado en todos los lugares donde se utilizó.

## Método de trabajo

1. Definir objetivo y topología.
2. Comprobar cada capa: enlace, IP, servicio, protocolo, autenticación y permisos.
3. Registrar el comando, el resultado, la hipótesis y la corrección.
4. Mantener una vía de administración local para recuperación.
5. Guardar estado estable antes de añadir componentes.

**Historial:** consolidado a partir de sesiones prácticas realizadas en EVE-NG y pruebas de noviembre de 2026. Se distinguen los resultados confirmados de los pasos pendientes.

> Repositorio técnico de aprendizaje: errores de teclado, comandos repetidos, permisos Linux, servicios duplicados y contraseñas confundidas incluidos, porque también son troubleshooting.
