# Parte I — EVE-NG NOC Access & AAA: HOP + FreeRADIUS + MikroTik CHR

**Estado:** autenticación, autorización y accounting probados con resultados reales (noviembre de 2026). **No** se afirma que el endurecimiento operativo ni las copias de seguridad finales estén completados.

## 1. Objetivos y criterios de éxito

- Acceso a CHR exclusivamente a través del servidor de salto para los operadores de esta práctica.
- Usuarios RADIUS centralizados, sin necesidad de crearlos en `/user` de MikroTik.
- Autorización por grupos locales de RouterOS: `noc-auditor` con `ssh,read,test`, sin `write`, `reboot`, etc.
- Administrador `admin` local (`full`) preservado como emergencia y auditor `auditor-local` local de respaldo.
- Comprobar `Access-Accept`, `Access-Reject`, rechazo de cambios de configuración, y `Accounting-Request/Response` `Start/Stop`.

## 2. Entorno y topología

- EVE-NG dentro de **VMware Workstation**; EVE VM aproximadamente 8 GB RAM y 4 vCPU durante la práctica (puede limitar ampliaciones GPON).
- **VMnet8 NAT**: `192.168.19.0/24`, conectado a EVE `pnet0` mediante Cloud0 (gestión EVE, salida a Internet para instalar paquetes).
- **VMnet2 host-only**: `172.20.99.0/24`, host Windows `172.20.99.1`, Cloud1/`pnet1`; ofrece acceso al servidor de salto.
- EVE **NET-MGMT-10**: `10.10.10.0/24`. El sufijo 10 es un nombre lógico de red EVE, **no es evidencia de etiquetado 802.1Q**.
- EVE **NET-SERVICES-30**: `10.10.30.0/24`. Misma distinción sobre VLAN real.
- Un segundo enlace temporal desde RADIUS a Cloud0 (`ens4`) se utilizó para APT. Es temporal, **pendiente de retirada controlada**.

```text
Windows MobaXterm
  └── Cloud1 / VMnet2 / pnet1 (172.20.99.0/24)
       └── HOP-SERVER Debian
            ens4 = 172.20.99.20/24
            ens3 = 10.10.10.20/24
             └── SSH sobre NET-MGMT-10
                  └── CHR RouterOS 7.23.8
                       ether1 = 10.10.10.1/24
                       ether2 = 10.10.30.1/24
                        └── RADIUS UDP 1812 + Accounting UDP 1813
                             └── RADIUS-SERVER Debian 13
                                  ens3 = 10.10.30.10/24
                                  ens4 = DHCP 192.168.19.131/24 (solo durante provisión)
```

## 3. HOP-SERVER (Debian)

- SSH servidor activo; autenticación desde MobaXterm con el usuario Linux **`labops`** (no confundir con `ilias` ni con el homónimo RADIUS).
- Linux `ens3`: 10.10.10.20/24; `ens4`: 172.20.99.20/24, configurados de forma persistente durante el laboratorio.
- Se probaron claves SSH ED25519 y `authorized_keys`, además de autenticación por contraseña.
- En `/home/labops/.ssh/config` se definió un alias de acceso a CHR (host `10.10.10.1`, usuario `admin`, puerto 22). Se puede usar `ssh admin@10.10.10.1` explícitamente.
- Tras la instalación en Debian se solucionaron problemas de distribución de teclado, copia/pega y paquetes `openssh-server`. MobaXterm sustituyó a SecureCRT en la práctica.

## 4. RouterOS CHR

Identidad observada del router: `CHR`.

Direcciones de ejemplo para contrastar con el propio equipo:

```routeros
/ip address print
/radius print
/radius monitor 0 once
/user aaa print
/user group print where name="noc-auditor"
/user print
```

Grupo de auditor restringido, **si no existe** (ya fue creado en el laboratorio; no volver a duplicarlo):

```routeros
/user group add name=noc-auditor policy=ssh,read,test
```

Cuentas locales verificadas: `admin` (grupo `full`) y `auditor-local` (grupo `noc-auditor`, fuente `10.10.10.20/32`). `labops` e `ilias` **no aparecen como cuentas locales**: sus credenciales se consultan a RADIUS.

La entrada única de servidor (solo crear si aún no existe):

```routeros
/radius add service=login address=10.10.30.10 src-address=10.10.30.1 secret="<SECRETO_RADIUS_LARGO_Y_UNICO>"
```

> Importante: `/radius add` es una **mutación** y no imprime OK; no repetir por falta de salida. Comprobar con `/radius print`. Durante el laboratorio se crearon accidentalmente seis entradas y se eliminaron cinco duplicados.

Ajustes AAA finales observados:

```routeros
/user aaa set default-group=noc-auditor exclude-groups=full
/user aaa set use-radius=yes
/user aaa print
```

Observado antes de activar: `use-radius=no`, `accounting=yes`, `default-group=read`, `exclude-groups` vacío. Luego se cambió el default y se excluyó `full` antes de activar RADIUS. **Preservar sesión SSH `admin` mientras se prueba**.

Mostrar una propiedad en RouterOS:

```routeros
:put [/radius get 0 src-address]
```

`/radius get 0 src-address` sin `:put` **no imprimió nada**, aunque el valor estaba establecido en `10.10.30.1`.

## 5. FreeRADIUS en Debian 13

- FreeRADIUS **3.2.7**, servicio `freeradius`, instalado con `freeradius-utils`.
- Paquetes Debian vía repos oficiales `trixie`, `trixie-updates` y `trixie-security`, tras desactivar entrada `deb cdrom:` obsoleta.
- `/etc/freeradius/3.0/clients.conf`: se añadió un cliente `chr-noc` con `ipaddr=10.10.30.1` y un **secreto compartido idéntico** al del CHR. Se añadió posteriormente `require_message_authenticator = true` y el acceso SSH se volvió a comprobar exitosamente.
- `/etc/freeradius/3.0/mods-config/files/authorize`: usuarios RADIUS con `Cleartext-Password` y atributo de respuesta `Mikrotik-Group` (exclusivamente práctica aislada; no copiar contraseñas reales al repositorio).
- Se autenticó `labops` mediante `radtest` y CHR; posteriormente se creó otra cuenta RADIUS cuyo nombre final verificado en el prompt fue **`ilias`**, grupo `noc-auditor`.

**Fragmentos de referencia (marcadores, no copiar valores literales):**

```text
# En clients.conf, dentro del bloque del cliente CHR:
client mikrotik-chr {
    ipaddr = 10.10.30.1
    secret = <SECRETO_COMPARTIDO_GENERADO>
    shortname = chr-noc
    require_message_authenticator = true
}

# En mods-config/files/authorize; sustituir contraseñas de ejemplo:
labops Cleartext-Password := "<CONTRASENA_RADIUS_PROPIA>"
    Mikrotik-Group := "noc-auditor"

ilias Cleartext-Password := "<OTRA_CONTRASENA_RADIUS_PROPIA>"
    Mikrotik-Group := "noc-auditor"
```

Tener en cuenta: si el servidor está en modo interactivo `freeradius -X`, el cambio a disco requiere reiniciar esa instancia; si se administra con systemd, usar `systemctl restart freeradius`. **Nunca arrancar ambas simultáneamente sobre UDP 1812**.

Comprobaciones:

```bash
freeradius -XC
systemctl is-active freeradius
ss -lunp | grep ':1812'
```

Para una contraseña RADIUS introducida sin exponerla en el historial, en el servidor Debian:

```bash
read -rsp "Clave del usuario RADIUS: " RADPASS; echo
radtest -t pap labops "$RADPASS" 127.0.0.1 0 <SECRETO_LOCALHOST_CONFIGURADO>
unset RADPASS
```

En `radtest` **la salida puede revelar la contraseña**. No compartirla ni incluirla en capturas públicas. El secreto de `localhost` es distinto del secreto compartido con CHR.

## 6. Evidencias de funcionamiento (sin credenciales)

1. `freeradius -XC` → `Configuration appears to be OK`.
2. `systemctl is-active freeradius` → `active`.
3. `radtest` con credenciales correctas → `Received Access-Accept`; con contraseña errónea → `Received Access-Reject`.
4. Respuesta RADIUS del usuario `labops`: `Mikrotik-Group = "noc-auditor"`.
5. `/ping 10.10.30.10 count=3` desde CHR → 3/3, 0 % pérdidas, RTT medio ~1.915 ms.
6. SSH HOP→CHR como `labops`: prompt `[labops@CHR] >`; comando `/user print` permitido.
7. Modificación probada `/system identity set name=CHR-PRUEBA` → **`not enough permissions (9)`**; el nombre del router no cambió.
8. SSH HOP→CHR con el usuario adicional `ilias`: prompt `[ilias@CHR] >`; las fallas previas quedaron en el log.
9. `tcpdump -ni ens3 'udp port 1813'`: dos `Accounting-Request` y dos `Accounting-Response`, sin pérdidas.
10. Registros de FreeRADIUS: `/var/log/freeradius/radacct/10.10.30.1/detail-20261104`; sesión `84000005` con `Start` a 09:39:19 EST y `Stop` a 09:39:42 EST, duración ~23 s, `Calling-Station-Id=10.10.10.20`. Otros eventos `Accounting-On` y un `Start` anterior `84000004` sin Stop visible en el extracto.

## 7. Tres contraseñas distintas — no confundir

1. Contraseña Linux del usuario `labops` para HOP-SERVER.
2. Contraseña RADIUS de `labops` o `ilias` en `authorize`, para SSH en CHR.
3. Secreto compartido entre CHR y FreeRADIUS en `/radius` y `clients.conf`.

No son intercambiables. Una contraseña de prueba RADIUS fue revelada durante las pruebas; **debe renovarse y validarse de nuevo** si aún es válida.

## 8. Siguiente laboratorio — FTTH/GPON

Mantener este laboratorio AAA como **Parte I** estable. Parte II propuesta: VOLTHA + BBSim sobre máquinas separadas, simulación lógica de OLT y ONUs, control y gestión; validar recursos y compatibilidad antes de definir número real de puertos/ONUs. La integración de los planos GPON con MikroTik/AAA **no ha sido ejecutada ni probada**.

Para reproducir las incidencias exactas, ver [TROUBLESHOOTING.md](TROUBLESHOOTING.md). Para cierre y operaciones, ver [OPERACIONES.md](OPERACIONES.md).
