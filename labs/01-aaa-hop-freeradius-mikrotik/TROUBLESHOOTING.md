# Parte I — Bitácora completa de errores y troubleshooting (NOC AAA)

**Laboratorio:** EVE-NG + VMware Workstation + Debian HOP-SERVER + Debian FreeRADIUS 3.2.7 + RouterOS CHR 7.23.8. **Periodo:** octubre-noviembre 2026. Casos ordenados aproximadamente por la fase donde ocurrieron.

> Se incluyen los despistes humanos sin atribuir culpas. El objetivo es aprender a reconocer síntomas y diseñar una secuencia de diagnóstico. Los mensajes mostrados son fragmentos observados o expresamente señalados como ejemplos. Se han eliminado contraseñas, claves privadas y secretos RADIUS.

## Resumen de diagnóstico por capas

```text
EVE / VMware → LINK (UP/LOWER_UP/NO-CARRIER)
             → IP/DHCP/routes / ping
             → TCP SSH / UDP 1812, 1813
             → proceso escuchando / systemd
             → RADIUS cliente IP y shared secret
             → usuario + contraseña y protocolo MS-CHAPv2
             → Mikrotik-Group + permisos
             → Accounting / logs
```

## 1. EVE-NG sin IP de gestión: NO-CARRIER

**Síntoma:** bridge `pnet0` no obtenía IP; `eth0` y `pnet0` figuraban `DOWN`/`NO-CARRIER`. Habilitar la interfaz con `ip link set ... up` no bastaba.

**Diagnóstico:** revisar `ip -br addr`, `ip link show eth0`, tipo VMware VMnet8 NAT, enlace y DHCP.

**Corrección verificada:** desconectar y volver a conectar el adaptador virtual de EVE-NG en VMware Workstation sin reinstalar ni cambiar a Bridged. Reapareció `LOWER_UP` y `pnet0 192.168.19.128/24`.

**Lección:** `UP` administrativo no implica `LOWER_UP`; véase [Issue #1](../../../issues/1).

## 2. Redes EVE Cloud0/Cloud1 mezcladas con redes internas

**Síntoma:** confusión entre gestión NAT, acceso local desde Windows y segmentos internos aislados.

**Corrección:** distinguir `pnet0/Cloud0 → VMnet8 NAT 192.168.19.0/24`; `pnet1/Cloud1 → VMnet2 host-only 172.20.99.0/24`; `NET-MGMT-10 10.10.10.0/24`; `NET-SERVICES-30 10.10.30.0/24`.

**Lección:** un objeto Bridge de EVE etiquetado "VLAN 10" **no crea automáticamente una VLAN 802.1Q**. Documentar NIC↔interfaz y tipo de conexión.

## 3. Interfaces Linux `DOWN` / puente desconectado

**Síntoma:** nodos HOP y RADIUS no alcanzaban segmentos esperados.

**Diagnóstico:** `ip -br link`, `ip -br addr`, `ip route`, comprobar nodo EVE conectado al bridge correcto y habilitado.

**Corrección:** conectar las interfaces a sus bridges, levantarlas y asignar direcciones persistentes en `/etc/network/interfaces.d/`.

**Lección:** distinguir falta de IP, enlace virtual desconectado y fallo de routing.

## 4. La puerta de enlace MikroTik estaba apagada

**Síntoma:** `ping 10.10.30.1` desde Debian fallaba; parecía un problema de interfaces o ruta.

**Causa real comprobada:** el nodo CHR estaba apagado.

**Corrección:** encenderlo en EVE-NG y repetir la prueba bidireccional.

**Lección:** antes de depurar OSPF, VLAN o firewall, comprobar que el equipo remoto está encendido.

## 5. Error de teclado Debian en consola VNC/QEMU

**Síntoma:** dificultades para introducir `|`, `-`, `=`, `@` y comandos largos por diferencias de distribución de teclado; copiar y pegar no funcionaba como se esperaba.

**Corrección aplicada:** cambiar la distribución de teclado de Windows a **English (US)** con `Win + Space` para la consola en ese momento; después administrar Debian por SSH desde MobaXterm para pegar texto.

**Lección:** validar la entrada con símbolos simples; no atribuir un fallo de comandos a paquetes o sintaxis hasta verificar teclado real.

## 6. Confusión entre usuarios Linux `ilias`, `labops`, `root`

**Síntoma:** los intentos SSH fallaban o `su`/archivos inaccesibles, porque se usaba una cuenta distinta de la configurada.

**Corrección:** distinguir el usuario Linux `labops` del HOP, el administrador `root` para APT y servicios, y los usuarios remotos RADIUS del mismo nombre.

**Verificación:** comprobar el prompt de Debian (`labops@...$` versus `root@...#`) y el de RouterOS (`[admin@CHR] >`, `[labops@CHR] >`).

## 7. SSH inexistente: `ssh: command not found` o servicio ausente

**Síntoma:** no se podía conectar desde o hacia nodos Debian; `systemctl status ssh` no hallaba servicio.

**Causa:** faltaban los componentes `openssh-client` y/o `openssh-server` en instalaciones mínimas, según nodo.

**Corrección:** instalar el paquete pertinente por APT, verificar `systemctl is-active ssh`, escuchar en puerto 22, probar con usuario y destino explícitos.

**Lección:** cliente `ssh` y demonio `sshd` son componentes diferentes.

## 8. Licencia SecureCRT y migración a MobaXterm

**Síntoma:** la licencia de SecureCRT limitó la continuidad del laboratorio.

**Corrección:** migrar a MobaXterm y guardar sesiones SSH para HOP y RADIUS; se probaron funciones de macros (alternativa al Button Bar de SecureCRT).

**Lección de seguridad:** no guardar contraseñas reales como texto plano dentro de macros. Usar bóveda/cifrado y limitar permisos.

## 9. Claves SSH ED25519 y `authorized_keys`

**Síntoma:** dudas entre contraseña SSH, identidad `admin` del CHR y claves públicas del HOP; hubo necesidad de corregir archivo de claves autorizadas.

**Corrección:** generar/usar par ED25519, instalar **solo la clave pública** en `~/.ssh/authorized_keys`, verificar nombre de usuario, permisos y acceso; mantener un medio de recuperación alternativo. Se depuró el archivo de claves del usuario Linux `labops`.

**Lección:** nunca copiar una clave privada a repositorios o mensajes.

## 10. SSH bloqueado temporalmente después de muchos intentos

**Síntoma:** conexiones rechazadas o desconectadas pese a configuración aparentemente correcta; ocurrió tras numerosos intentos.

**Hipótesis compatible con registros:** penalización temporal `PerSourcePenalties` en `sshd`. Se recuperó después de esperar sin insistir aproximadamente 10 minutos.

**Lección:** evitar repetir conexiones masivamente; distinguir bloqueo temporal de credencial incorrecta. No desactivar mecanismos de seguridad apresuradamente.

## 11. Instalar FreeRADIUS sin repos APT — Candidate `(none)`

**Síntoma:** `apt-cache policy freeradius freeradius-utils` devolvía `Installed: (none)` y `Candidate: (none)`.

**Causa:** Debian 13 se instaló sin repositorios de red configurados.

**Corrección:** obtener Internet temporal por interfaz `ens4` → Cloud0 (NAT), usar el `dhcpcd` instalado, comprobar IP, gateway y DNS, configurar `debian.sources` oficial para Trixie, actualizar índices.

**Comprobaciones utilizadas:**

```bash
ip -br addr
ip route
getent ahostsv4 deb.debian.org
apt-cache policy freeradius freeradius-utils
```

**Resultado posterior:** FreeRADIUS 3.2.7 candidato disponible e instalado.

## 12. `dhclient: command not found` en Debian mínimo

**Síntoma:** intento de obtener DHCP con una utilidad que no estaba instalada.

**Comprobación:** existía `/usr/sbin/dhcpcd`, no `dhclient` ni `udhcpc`.

**Corrección aplicada:**

```bash
dhcpcd -4 -L -t 20 ens4
```

**Resultado:** `ens4=192.168.19.131/24`, ruta por defecto vía `192.168.19.2`, DNS funcional y ping externo correcto.

**Lección:** no asumir el cliente DHCP disponible por el nombre de la distribución.

## 13. Error de escritura en configuración estática: `netmack`

**Síntoma:** archivo `/etc/network/interfaces.d/radius-static` tenía un parámetro incorrecto.

**Causa:** se escribió `netmack` en lugar de `netmask`.

**Corrección:**

```text
auto ens3
iface ens3 inet static
    address 10.10.30.10
    netmask 255.255.255.0
```

**Lección:** revisar literalmente palabras clave y verificar IP tras la recarga de red.

## 14. `apt update` fallaba por repositorio CD-ROM obsoleto

**Error exacto observado:**

```text
The repository 'cdrom://[Debian GNU/Linux 13.7.0 ...] trixie Release' does not have a Release file.
```

**Diagnóstico:**

```bash
grep -Rni 'cdrom:' /etc/apt 2>/dev/null
```

**Resultado:** referencia en `/etc/apt/sources.list:1`.

**Corrección:** comentar solo la entrada `deb cdrom:`; preservar fuentes oficiales Debian de red.

**Evidencia:** `apt update` mostró `Hit` para `trixie`, `trixie-security`, `trixie-updates`, además de 13 paquetes actualizables.

## 15. Diferencia `apt update` / `apt upgrade` (recordatorio de Termux)

**Situación:** consulta sobre si había que ejecutar también `upgrade`.

**Aprendizaje:** `apt update` descarga índices; `apt upgrade` actualiza paquetes instalados; `apt full-upgrade` puede modificar de manera más amplia dependencias y retirar paquetes.

**Resultado:** instalaciones de `openssh-server`, `freeradius` y `freeradius-utils` completadas. Un aviso de `update-rc.d` no impidió el arranque del servicio.

## 16. FreeRADIUS `authorize`: muchos comentarios y no saber dónde añadir usuario

**Síntoma:** se abrió al final el archivo `mods-config/files/authorize`; parecía todo comentado.

**Corrección:** posicionarse al **principio**, tal como indica el comentario de ese archivo (`add test accounts to the TOP`), insertar entrada de `labops` sin `#`, guardar, validar con `freeradius -XC`.

**Lección:** comentarios de ejemplo `#` no son usuarios activos; la posición del bloque importa.

## 17. Interpretar `freeradius -XC`

**Síntoma:** salida extensa de 700+ líneas, difícil identificar si falló.

**Corrección:** buscar el resultado final `Configuration appears to be OK`. Esto prueba sintaxis/carga, **no** prueba contraseñas ni conectividad remota.

## 18. Contraseña vacía en `radtest`

**Resultado observado:**

```text
User-Password = ""
Received Access-Reject
```

**Causa:** se usó `read -rsp "admin" RADPASS; echo` entendiendo erróneamente que el argumento `"admin"` era la contraseña. En realidad era **el prompt**. La variable quedó vacía.

**Corrección:**

```bash
read -rsp "Password RADIUS: " RADPASS; echo
test -n "$RADPASS" && echo "Password OK" || echo "Password VACIA"
# Ejecutar radtest local sin compartir salida sin filtrar
unset RADPASS
```

**Resultado:** Access-Accept cuando se introdujo la contraseña correcta.

## 19. `Expected Access-Accept got Access-Reject` en contraseña incorrecta

**Síntoma:** parecía fallo del comando tras una prueba deliberadamente negativa.

**Explicación:** `radtest` espera aceptación por defecto; cuando se envía una clave falsa, `Access-Reject` **es el resultado correcto** del test negativo.

**Lección:** interpretar el objetivo de la prueba, no solo el código/advertencia genérica.

## 20. Seis entradas RADIUS duplicadas en CHR

**Síntoma observado:**

```text
# SERVICE ADDRESS
0 login 10.10.30.10
1 login 10.10.30.10
2 login 10.10.30.10
3 login 10.10.30.10
4 login 10.10.30.10
5 login 10.10.30.10
```

**Causa confirmada:** se repitió `/radius add` porque RouterOS volvía al prompt sin imprimir confirmación.

**Corrección:** `/radius print`, retirar duplicados `1..5` tras confirmar los IDs, conservar una entrada.

**Lección:** la ausencia de salida tras `add` no significa fallo. Verificar siempre antes de repetir.

## 21. `/radius get 0 src-address` no mostraba valor

**Síntoma:** comando ejecutado sin error y sin texto.

**Corrección:**

```routeros
:put [/radius get 0 src-address]
```

**Resultado:** `10.10.30.1`. En RouterOS, `:put` muestra el valor de la expresión.

## 22. `use-radius: no` interpretado como falta de usuarios

**Síntoma:** confusión conceptual al revisar `/user aaa print`.

**Explicación:** `use-radius=no` significa que RouterOS **no consulta RADIUS para login**, no que RADIUS carezca de usuarios.

**Prevención:** comprobar cuentas en FreeRADIUS, grupos en CHR y el parámetro AAA por separado.

## 23. Permisos `noc-auditor` y signos `!`

**Resultado confirmado:**

```text
policy=ssh,read,test,!local,!telnet,!ftp,!reboot,!write,!policy,...
```

**Explicación:** `!` indica privilegio **no concedido**. La restricción se probó intentando modificar la identidad del router y recibiendo `not enough permissions (9)`.

## 24. MikroTik `login failure`, sin Access-Accept/Reject

**Resultado de `/radius monitor 0 once`:**

```text
requests: 24
accepts: 0
rejects: 0
resends: 39
timeouts: 24
bad-replies: 0
```

**Hipótesis inicial:** problema UDP 1812, secreto compartido o respuestas descartadas. No se demostró un problema de contraseña en ese momento.

**Verificación de red:**

```bash
tcpdump -ni ens3 'udp port 1812'
```

**Resultado:** Access-Request desde `10.10.30.1` llegaban repetidamente a `10.10.30.10:1812`; no se veían respuestas. Por tanto **no** era una desconexión IP.

## 25. ¡Causa raíz del timeout! `invalid Message-Authenticator`

**FreeRADIUS `freeradius -X` mostró:**

```text
Dropping packet without response because of error:
Received packet from 10.10.30.1 with invalid Message-Authenticator!
(Shared secret is incorrect.) (from client chr-noc)
```

**Causa identificada:** el secreto RADIUS de `clients.conf` no coincidía con el de la única entrada `/radius` del CHR; era compatible con un error al introducir símbolos/comillas, pero **no se demostró** que RouterOS no admitiese símbolos.

**Corrección verificada:** generar un secreto largo y aleatorio con formato hexadecimal, establecer **idéntico** en ambos equipos, reiniciar el proceso FreeRADIUS apropiado y repetir SSH. Desapareció el error de integridad.

**Lección:** un secreto compartido incorrecto puede causar un **descarte sin respuesta**, no un Access-Reject.

## 26. Nano vacío: `Permission denied`

**Síntoma:** `nano /etc/freeradius/3.0/clients.conf` mostraba pantalla aparentemente vacía.

**Causa confirmada:** la sesión estaba como `radiusserver@radiusserver:~$`, no `root@radiusserver:~#`; el archivo era `freerad:freerad`, `-rw-r-----`.

**Comprobación sin divulgar secretos, una vez `su -`:**

```bash
ls -lh /etc/freeradius/3.0/clients.conf
wc -l /etc/freeradius/3.0/clients.conf
```

**Resultado:** ~12 KB, **390 líneas**. No estaba vacío; se evitó sobrescribir/restaurar indebidamente.

**Lección:** salir de Nano **sin guardar** ante una pantalla sospechosamente vacía; comprobar permisos primero.

## 27. `freeradius -X`: puerto UDP 1812 ocupado con servicio `inactive`

**Error exacto:**

```text
Failed binding to auth address * port 1812 bound to server default: Address already in use
```

**Diagnóstico:**

```bash
systemctl is-active freeradius
ps -ef | grep '[f]reeradius'
ss -lunp | grep ':1812'
```

**Resultado:** systemd inactivo, pero instancia **manual** de FreeRADIUS PID **37257** seguía en ejecución, escuchando `0.0.0.0:1812` y `[::]:1812`.

**Corrección aplicada:**

```bash
kill -TERM 37257
ss -lunp | grep ':1812'
freeradius -XC
freeradius -X
```

**Lección:** `systemctl inactive` no demuestra que no exista un proceso huérfano/ejecutado fuera de systemd. Nunca arrancar dos instancias en el mismo puerto.

## 28. Error posterior distinto: MS-CHAPv2 `Access-Reject`

**El secreto compartido ya era válido:** el servidor pudo verificar Message-Authenticator, identificar al cliente y responder.

**Registro observado:**

```text
files: users: Matched entry labops at line 202
mschap: Client is using MS-CHAPv2
mschap: ERROR: MS-CHAP2-Response is incorrect
Sent Access-Reject
```

**Causa compatible con el caso y resuelta en la práctica:** se introducía una contraseña diferente de la existente en `Cleartext-Password` para `labops`.

**Corrección:** consultar/actualizar la contraseña RADIUS correcta en `authorize`, recargar FreeRADIUS, reconectar desde HOP con esa contraseña. **No confundir contraseña Linux HOP, contraseña RADIUS usuario y secreto entre dispositivos.**

**Resultado:** `[labops@CHR] >`.

## 29. Prueba de autorización que tenía que fallar

**Comandos usados:**

```routeros
/user print
/system identity set name=CHR-PRUEBA
```

**Salida:**

```text
not enough permissions (9)
```

**Interpretación correcta:** el grupo `noc-auditor` permite lectura y diagnóstico, no `write`; **el rechazo es la prueba positiva de autorización**.

## 30. Los usuarios RADIUS no aparecen en `/user print`

**Síntoma:** sesión real `[labops@CHR] >` pero solo se mostraban `admin` y `auditor-local` en la tabla local.

**Explicación:** `/user print` muestra cuentas locales de RouterOS; `labops` era un usuario remoto RADIUS. No crearle una cuenta local por ese motivo.

## 31. Accounting UDP 1813: cómo verificarlo de verdad

**Captura:**

```text
10.10.30.1:39080 > 10.10.30.10:1813 Accounting-Request
10.10.30.10:1813 > 10.10.30.1:39080 Accounting-Response
10.10.30.1:43114 > 10.10.30.10:1813 Accounting-Request
10.10.30.10:1813 > 10.10.30.1:43114 Accounting-Response
4 packets captured, 0 packets dropped by kernel
```

**Registro en disco:** `/var/log/freeradius/radacct/10.10.30.1/detail-20261104`.

**Evidencia:**
- `Accounting-On` registrado en dos ocasiones.
- `labops` desde `10.10.10.20`, sesión `84000005`: `Start` 09:39:19 y `Stop` 09:39:42 (duración 23 s).
- Otra sesión `84000004` tenía `Start`, sin `Stop` en el extracto mostrado; **no afirmar** que fue cerrada correctamente.

## 32. Aviso BlastRADIUS `require_message_authenticator`

**Aviso observado:**

```text
BlastRADIUS check: Received packet with Message-Authenticator.
Please set "require_message_authenticator = true" for client chr-noc
```

**Acción:** activar `require_message_authenticator = true` dentro del cliente `chr-noc` en `clients.conf`; validar con `freeradius -XC`, reiniciar servicio y verificar una nueva sesión SSH.

**Resultado:** acceso remoto volvió a funcionar. **No desactivar** Message-Authenticator para solucionar otros fallos.

## 33. Alta de segundo operador con nombre distinto al pensado

**Síntoma:** se había propuesto el usuario `ilias-noc` pero el usuario creado finalmente era **`ilias`**; primer intento SSH con nombre equivocado falló.

**Corrección:** usar exactamente el nombre configurado en `authorize`. **Evidencia final:** `[ilias@CHR] >`.

**Lección:** los nombres de cuentas son datos exactos, no alias intercambiables.

## 34. MobaXterm, botón de contraseña y Button Bar de SecureCRT

**Necesidad:** un botón que envíe texto a la terminal, no un botón que abra automáticamente el gateway SSH.

**Alternativa encontrada:** macros de MobaXterm para enviar texto/atajos. Se completó una macro en la práctica.

**Advertencia:** las macros pueden almacenar contraseñas como texto; no publicar ni exportar esas macros. Preferir gestor de credenciales con contraseña maestra y SSH Gateway para gestionar el salto.

## 35. Incidentes operativos menores que merece la pena recordar

- **Confundir SSH de HOP con SSH de CHR:** `labops@HOP` es un usuario Linux, `labops@CHR` es un usuario RADIUS. Las contraseñas pueden ser distintas.
- **Pegar comandos largos en VNC:** usar SSH a Debian y mantener terminales etiquetadas para no ejecutar RouterOS en Bash o viceversa.
- **Olvidar que `freeradius -XC` no recarga el servicio:** validar y después reiniciar **la instancia realmente activa**.
- **Repetir comandos porque no imprimen OK:** comprobar antes con `print`, `ss`, `systemctl` o logs.
- **Compartir salida completa de `radtest`:** puede exponer `User-Password` y `Cleartext-Password`. Renovar credenciales reveladas.
- **Confiar solo en `ping`:** respuesta ICMP no equivale a comunicación RADIUS válida.
- **Confundir `Access-Reject` con `timeout`:** uno es rechazo deliberado del servidor; el otro indica ausencia de respuesta válida.
- **Acceso de emergencia:** mantener `admin` local funcionando y una sesión abierta durante cambios de AAA; no retirar cuentas de respaldo antes de probar.

---

## Cuadro de búsqueda rápida

| Mensaje o señal | Comprobación prioritaria |
|---|---|
| `NO-CARRIER` | NIC EVE/VMware y bridge, no DHCP primero |
| `ssh: command not found` | Cliente SSH instalado |
| `ssh.service not found` | OpenSSH Server instalado |
| `Candidate: (none)` | Fuentes APT, DNS e Internet |
| `cdrom ... no Release file` | Fuente CD-ROM en `sources.list` |
| `Permission denied` leyendo `clients.conf` | `whoami`, `su -`, permisos |
| `Address already in use` | `ss -lunp`, proceso manual y systemd |
| RADIUS `timeouts` | `tcpdump` UDP 1812 en servidor, `freeradius -X` |
| `invalid Message-Authenticator` | Secreto compartido cliente/servidor |
| `MS-CHAP2-Response is incorrect` | Contraseña RADIUS y tipo de auth |
| `not enough permissions (9)` | Grupo RouterOS; puede ser esperado |
| `Received Access-Reject` en test negativo | Comportamiento deseado |
| `/radius get` sin salida | `:put [/radius get ...]` |
| `/user print` no muestra usuario RADIUS | Tabla local ≠ directorio RADIUS |
| `Accounting-Request/Response` | Correlacionar `Acct-Session-Id`, Start y Stop |

**Nota:** esta bitácora reconstruye la secuencia a partir de salidas de terminal y conversaciones de práctica. No todos los errores disponen de un log textual íntegro ni una causa raíz concluyente; esos casos se identifican como hipótesis o lecciones, no como hechos confirmados.
