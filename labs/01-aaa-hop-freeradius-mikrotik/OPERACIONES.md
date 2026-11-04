# Parte I — Procedimientos de operación, cierre y futura integración GPON

Esta sección convierte las pruebas realizadas en un **runbook** reutilizable para tareas típicas de NOC. Los fragmentos contienen marcadores: **no** son volcados del router ni incluyen secretos.

## 1. Operación normal: estado de servicios

En RADIUS-SERVER, como root:

```bash
systemctl is-active freeradius
ss -lunp | grep ':1812'
freeradius -XC
```

- `active` indica que el servicio administrado por systemd está activo.
- `-XC` valida la configuración del disco sin demostrar tráfico real.
- Si `systemctl` dice `inactive` pero hay puerto ocupado, inspeccionar `ps -ef | grep '[f]reeradius'`: podría existir una instancia manual `freeradius -X`.

En MikroTik CHR:

```routeros
/radius print
/radius monitor 0 once
/user aaa print
/user group print where name="noc-auditor"
```

Ante un fallo de acceso: **mantener abierta una sesión local `admin`**, después diagnosticar `/radius monitor`, `tcpdump` y `freeradius -X` sin lanzar múltiples instancias.

## 2. Alta de un operador de solo lectura

**Objetivo:** operador gestionado por FreeRADIUS, sin crear usuario local en CHR, con grupo `noc-auditor`.

### Comprobaciones previas

- Grupo RouterOS existe: `/user group print where name="noc-auditor"`.
- Restricciones del grupo: `ssh,read,test`; `!write` y `!reboot`.
- Cliente RADIUS CHR registrado con `ipaddr=10.10.30.1`.
- Acceso de recuperación `admin` comprobado.
- No duplicar nombres de usuarios locales: `/user print`.

### 2.1. Crear usuario en el archivo de FreeRADIUS

```bash
cp -a /etc/freeradius/3.0/mods-config/files/authorize /root/authorize.before-new-user.bak
nano /etc/freeradius/3.0/mods-config/files/authorize
```

Añadir al **inicio del archivo**, sustituyendo todo marcador:

```text
OPERADOR_NUEVO Cleartext-Password := "<CONTRASENA_UNICA_SEGURA>"
    Mikrotik-Group := "noc-auditor"
```

**Precaución:** ejemplo didáctico con contraseña en texto claro, apropiado solo para entorno aislado. Para producción, integrar un backend central seguro, gestión de secretos y controles de acceso.

Validar y aplicar el cambio si se usa systemd:

```bash
freeradius -XC
systemctl restart freeradius
systemctl is-active freeradius
```

Si hay un `freeradius -X` manual activo, hay que detenerlo/reiniciarlo como corresponda **en lugar de ejecutar otra instancia** sobre el puerto 1812.

### 2.2. Validar autenticación y atributo de grupo

Para una prueba local con `radtest`, introducir contraseña por prompt y no publicar salida sin filtrar:

```bash
read -rsp "Clave RADIUS: " RADPASS; echo
radtest -t pap OPERADOR_NUEVO "$RADPASS" 127.0.0.1 0 <SECRETO_LOCALHOST_CONFIGURADO> 2>&1 | grep -E 'Received Access-|Mikrotik-Group'
unset RADPASS
```

Esperado: `Received Access-Accept` y `Mikrotik-Group = "noc-auditor"`. Para prueba negativa, una contraseña incorrecta debe producir `Access-Reject`.

**El secreto `localhost` NO es el secreto CHR↔RADIUS.**

### 2.3. Validar acceso real y autorización

Desde HOP-SERVER:

```bash
ssh -o PreferredAuthentications=password -o PubkeyAuthentication=no OPERADOR_NUEVO@10.10.10.1
```

En el CHR, verificar lectura:

```routeros
/interface print
/user print
```

Prueba negativa de autorización (sin cambiar identidad, si el grupo es correcto):

```routeros
/system identity set name=CHR-PRUEBA
```

Esperado: `not enough permissions (9)`; no ejecutar este test con `admin` porque sí podría modificar la identidad.

**Sesión del laboratorio confirmada:** el usuario adicional quedó finalmente como `ilias`, no `ilias-noc`. Se mostró `[ilias@CHR] >`; esto ilustra la importancia de verificar el nombre literal del usuario.

## 3. Baja temporal, revocación y modificación

> **Práctica pendiente de ejecutar en este laboratorio.** El procedimiento siguiente es un runbook propuesto, no una acción ya realizada.

- Registrar qué cuenta se quiere revocar y comprobar que **no** sea el administrador local de emergencia.
- Hacer copia del archivo `authorize`.
- Comentar la entrada del operador y su línea de grupo o retirarlas con cuidado, validando la sintaxis.
- Aplicar cambios a la instancia correcta y probar **una nueva conexión**: resultado esperado, `Access-Reject`.
- Comprobar que el resto de operadores sigue entrando correctamente.
- **Advertencia importante:** revocar credenciales RADIUS impide **nuevas autenticaciones**; no garantiza desconectar sesiones SSH ya establecidas. Para ello hay que gestionar las sesiones activas por separado, con la autoridad correspondiente.
- Para una modificación de permisos, cambiar el atributo `Mikrotik-Group` a un grupo previamente definido y verificar la política efectiva en CHR.

## 4. Accounting y auditoría

En CHR: `/user aaa print` mostró `accounting=yes` e `interim-update=0s`; eso no significa automáticamente que haya logs de inicio y cierre hasta probarlo.

Captura temporal:

```bash
tcpdump -ni ens3 'udp port 1813'
```

Buscar ficheros guardados:

```bash
find /var/log/freeradius/radacct -type f -name 'detail-*'
```

Extraer campos sin mostrar secretos:

```bash
grep -E 'Acct-Status-Type|User-Name|Calling-Station-Id|Acct-Session-Id|NAS-IP-Address|Timestamp' /var/log/freeradius/radacct/10.10.30.1/detail-20261104
```

**Evidencia observada:** usuario `labops`; IP de origen `10.10.10.20`; sesión `84000005`; `Start` a 09:39:19 EST y `Stop` a 09:39:42 EST. Otro `Start` carecía de `Stop` en el extracto; no atribuir cierre al registro incompleto.

Estas horas son las del reloj del servidor (se observaron etiquetas **EST**); documentar NTP y zona horaria antes de analizar incidencias entre varios equipos.

## 5. Seguridad — lo ya probado y lo pendiente

### Verificado

- Cliente CHR restringido por IP `10.10.30.1` en `clients.conf`.
- CHR envía `Message-Authenticator`.
- `require_message_authenticator = true` agregado al bloque de CHR; FreeRADIUS validó configuración y se probó nuevamente el SSH.
- `default-group=noc-auditor`, `exclude-groups=full` en CHR.
- `admin` local del CHR conservado y `auditor-local` como otra cuenta local.

### Pendiente de confirmar/realizar

- [ ] **Rotar todas las contraseñas de laboratorio que llegaron a mostrarse en texto claro**; no reutilizarlas en otros sistemas.
- [ ] Confirmar que el nuevo secreto compartido CHR↔RADIUS sigue siendo único, suficientemente largo y no expuesto en exports/historial/macros.
- [ ] Revisar la advertencia de clave corta para **localhost** (secreto independiente del cliente CHR), si aún aplica; al cambiarlo, actualizar las pruebas locales.
- [ ] Verificar el comportamiento de SSH ante la caída de RADIUS sin comprometer el acceso local de emergencia.
- [ ] Evaluar el uso de claves SSH y contraseñas protegidas por bóveda; no macros con secretos en texto plano.
- [ ] Implementar respaldo, política de retención, y permisos de lectura para logs.
- [ ] Validar sincronización NTP y zonas horarias.
- [ ] Revisar filtrado de puertos en Debian y CHR: permitir UDP 1812/1813 **solo** desde CHR y desde las redes de administración necesarias.
- [ ] Decidir arquitectura de producción: PAP/MS-CHAPv2 y secretos de laboratorio no equivalen a AAA endurecido a nivel empresarial.

## 6. Copias de seguridad y snapshots (antes de GPON)

**No se ha comprobado todavía la ejecución satisfactoria de todas estas operaciones.**

1. **RouterOS:** exportar una configuración revisada antes de publicarla. `/export file=AAA-NOC-PARTE1` genera un export pero no sustituye a una copia completa de respaldo. Ojo: algunos exports o cambios en consola pueden contener secretos; **no subirlos a GitHub**.
2. **Debian:** guardar de manera privada `/etc/freeradius/3.0`, los ficheros de interfaces, SSH y datos de prueba; el repositorio contiene solamente versiones saneadas.
3. **VMware/EVE-NG:** realizar snapshot de la VM EVE-NG con los nodos apagados cuando convenga y comprobar que se puede volver a un estado anterior.
4. **RADIUS-NIC temporal:** `ens4` tenía dirección DHCP `192.168.19.131` para instalar paquetes. **Antes de quitar el enlace Cloud0**, garantizar acceso SSH alternativo desde HOP o una consola VNC de recuperación; no cortar accidentalmente la única conexión de administración.
5. **MobaXterm:** guardar sesiones y macros sin credenciales publicables, proteger la bóveda, crear respaldos cifrados.

## 7. Checklist final de aceptación de la Parte I

- [x] SSH desde MobaXterm a HOP con `labops`.
- [x] SSH desde HOP a CHR con usuario local `admin`.
- [x] FreeRADIUS 3.2.7 operativo.
- [x] Usuario RADIUS `labops` autenticado, `Access-Accept`.
- [x] Contraseña incorrecta, `Access-Reject`.
- [x] `Mikrotik-Group = "noc-auditor"` devuelto por FreeRADIUS.
- [x] SSH HOP→CHR con usuario RADIUS `labops`.
- [x] Operador de solo lectura no puede cambiar identidad: `not enough permissions (9)`.
- [x] Accounting UDP 1813 con solicitudes y respuestas.
- [x] Accounting Start/Stop correlacionado por `Acct-Session-Id`.
- [x] Nuevo operador RADIUS identificado como `ilias` entra al CHR.
- [x] `require_message_authenticator = true` configurado y nueva prueba de SSH satisfactoria.
- [ ] Rotación final de todos los secretos expuestos confirmada.
- [ ] Backups y snapshot confirmados.
- [ ] Retirada de NIC temporal confirmada.

## 8. Parte II: futura integración FTTH/GPON

**Objetivo futuro, no ejecutado:** ampliar la arquitectura EVE-NG para conectar servicios NOC con VOLTHA/BBSim (OLT y ONUs simuladas), gestionar autorizaciones y registrar auditoría. Antes, revisar consumo de RAM/vCPU, capacidad de VM y conectividad de contenedores/simuladores en EVE-NG.

**Escenarios posteriores propuestos:** altas de ONUs, estado GPON, VLAN/DHCP y fallos de routing, alarmas de puertos PON, fallos de RADIUS/SSH, logs centralizados con Grafana/Zabbix. Diferenciar explícitamente una **simulación del plano de control GPON** de una red óptica física real.

---

**Principio operativo:** *el mejor runbook no oculta fallos: registra qué ocurrió, cómo se detectó, qué se cambió y qué evidencia demuestra la solución.*
