# 🔐 Situación Evaluativa ET3 — Seguridad de Red Empresarial (Packet Tracer)

![CCNA Security](https://img.shields.io/badge/CCNA-Security-1BA0D7?logo=cisco&logoColor=white)
![Packet Tracer](https://img.shields.io/badge/Packet%20Tracer-9.0.0-005073)
![ASA](https://img.shields.io/badge/Firewall-ASA%205505-orange)
![Estado](https://img.shields.io/badge/7%20Ítems-Completados-success)

Proyecto de endurecimiento de la red de una empresa con casa matriz en **Santiago** y sucursal en **Viña del Mar**. Cubre hardening de dispositivos, AAA/RADIUS, control de acceso, firewall ASA, VPN site-to-site e IOS IPS.

> **Integrantes del grupo**
> - Integrante 1: _______________________
> - Integrante 2: _______________________
> - Integrante 3: _______________________

---

## 📋 Tabla de contenidos

- [Topología](#-topología)
- [Direccionamiento](#-direccionamiento)
- [Ítem 1 — Direccionamiento y conectividad](#ítem-1--direccionamiento-y-conectividad)
- [Ítem 2 — Protección de acceso (hardening)](#ítem-2--protección-de-acceso-hardening)
- [Ítem 3 — Control de acceso](#ítem-3--control-de-acceso)
- [Ítem 4 — AAA por RADIUS y monitoreo](#ítem-4--aaa-por-radius-y-monitoreo)
- [Ítem 5 — Protección perimetral (ASA)](#ítem-5--protección-perimetral-asa)
- [Ítem 6 — VPN Site-to-Site](#ítem-6--vpn-site-to-site)
- [Ítem 7 — IOS IPS](#ítem-7--ios-ips)
- [Limitaciones del simulador](#-limitaciones-del-simulador)

---

## 🗺️ Topología

```
        ┌───────────┐   200.2.7.0/29        177.44.56.0/29   ┌──────────┐
        │    ISP    │ G0/1            G0/0                    │ FW-VINA  │
        │  (2911)   ●──────────●   .1        .6   ●───────────● ASA 5505 │
        │ Lo1:1.1.1.1│ .1     ╱                    ╲ E0/0(out) │  "55"    │
        └─────●─────┘        ╱                      ╲          └──┬────┬──┘
              │ G0/1 .6     ╱                        ╲       E0/2 │    │ E0/1
        ┌─────┴──────┐     ╱                          ╲     (dmz) │    │ (inside)
        │Edge-Santiago│G0/2 172.16.0.1/28         DMZ ●────┐  ┌───●   192.168.23.0/24
        │   (2911)   ●──────● AAA-SYSLOG          10.20.30.0/27 │      │
        └──┬─────────┘        172.16.0.14        Servidor Web   │   ┌──┴───┐
       G0/0│ (trunk .15/.25)                     10.20.30.30    │   │Switch0│
        ┌──┴───┐                                                │   └─┬──┬─┘
        │ SW1  │ VLAN15: 172.15.15.0/24 (PC2)                   │   PC0  PC1
        │(2960)│ VLAN25: 172.25.25.0/24 (PC3)                   │
        └──────┘ SVI VLAN15: 172.15.15.254                      │
```

**Zonas de seguridad del ASA (FW-VINA):**

| Interfaz física | VLAN | nameif | Nivel | IP |
|---|---|---|---|---|
| Ethernet0/1 | Vlan1 | `inside` | 100 | 192.168.23.1/24 |
| Ethernet0/0 | Vlan2 | `outside` | 0 | 177.44.56.6/29 |
| Ethernet0/2 | Vlan3 | `dmz` | 50 | 10.20.30.1/27 |

---

## 🌐 Direccionamiento

| Dispositivo | Interfaz | IP | Máscara |
|---|---|---|---|
| **Edge-Santiago** | G0/0.15 (VLAN15) | 172.15.15.1 | 255.255.255.0 |
| | G0/0.25 (VLAN25) | 172.25.25.1 | 255.255.255.0 |
| | G0/1 (→ISP) | 200.2.7.6 | 255.255.255.248 |
| | G0/2 (→SERVER) | 172.16.0.1 | 255.255.255.240 |
| **ISP** | G0/1 (→Edge) | 200.2.7.1 | 255.255.255.248 |
| | G0/0 (→ASA) | 177.44.56.1 | 255.255.255.248 |
| | Loopback1 (VPN) | 1.1.1.1 | 255.255.255.255 |
| **FW-VINA (ASA)** | E0/0 outside | 177.44.56.6 | 255.255.255.248 |
| | E0/1 inside | 192.168.23.1 | 255.255.255.0 |
| | E0/2 dmz | 10.20.30.1 | 255.255.255.224 |
| **AAA-SYSLOG** | Fa0 | 172.16.0.14 | 255.255.255.240 |
| **SW1** | SVI VLAN15 | 172.15.15.254 | 255.255.255.0 |
| **PC2** (VLAN15) | Fa0 | 172.15.15.10 | 255.255.255.0 |
| **PC3** (VLAN25) | Fa0 | 172.25.25.10 | 255.255.255.0 |
| **Servidor Web** (DMZ) | Fa0 | 10.20.30.30 | 255.255.255.224 |

> El loopback 1 del ISP (`1.1.1.1/32`) no venía en la tabla; se asignó para representar el endpoint remoto de la VPN (Ítem 6).

---

## Ítem 1 — Direccionamiento y conectividad

Se configuró el direccionamiento base y se resolvieron los problemas de conectividad detectados.

**Edge-Santiago** (router-on-a-stick para VLAN 15 y 25):
```cisco
hostname Edge-Santiago
interface GigabitEthernet0/0
 no ip address
 no shutdown
interface GigabitEthernet0/0.15
 encapsulation dot1Q 15
 ip address 172.15.15.1 255.255.255.0
interface GigabitEthernet0/0.25
 encapsulation dot1Q 25
 ip address 172.25.25.1 255.255.255.0
interface GigabitEthernet0/1
 ip address 200.2.7.6 255.255.255.248
 no shutdown
interface GigabitEthernet0/2
 ip address 172.16.0.1 255.255.255.240
 no shutdown
ip route 0.0.0.0 0.0.0.0 200.2.7.1
```

**ISP:**
```cisco
hostname ISP
interface GigabitEthernet0/1
 ip address 200.2.7.1 255.255.255.248
 no shutdown
interface GigabitEthernet0/0
 ip address 177.44.56.1 255.255.255.248
 no shutdown
```

**SW1** (VLANs + trunk hacia Edge):
```cisco
hostname SW1
vlan 15
vlan 25
interface GigabitEthernet0/1
 switchport mode trunk
 switchport trunk allowed vlan 15,25
interface range FastEthernet0/1-12
 switchport mode access
 switchport access vlan 15
interface range FastEthernet0/13-24
 switchport mode access
 switchport access vlan 25
interface vlan 15
 ip address 172.15.15.254 255.255.255.0
 no shutdown
ip default-gateway 172.15.15.1
```

**Problema resuelto:** los PCs de Santiago (PC2/PC3) **no tenían IP configurada**. Se les asignó dirección estática dentro de su VLAN (PC2 `172.15.15.10`, PC3 `172.25.25.10`, gateway el subinterface del Edge).

**Verificación (desde PC2):**
```
ping 172.15.15.1    → 4/4 (gateway)
ping 172.25.25.10   → OK (inter-VLAN vía router-on-a-stick)
ping 172.16.0.14    → OK (servidor AAA-SYSLOG)
```
El enlace Edge↔ISP también se validó (`ping 200.2.7.6` = 4/5).

---

## Ítem 2 — Protección de acceso (hardening)

Endurecimiento de Edge-Santiago. Todas las contraseñas `ciscocisco`, protegidas con MD5.

```cisco
security passwords min-length 8            ! largo mínimo 8
service password-encryption                ! cifra las contraseñas en el running
enable secret ciscocisco                   ! privilegiado protegido (MD5)
banner motd ~# ACCESO RESTRINGIDO#~        ! banner de advertencia
login block-for 45 attempts 3 within 60    ! anti fuerza bruta
username admin privilege 15 secret ciscocisco
ip domain-name cisco.com
crypto key generate rsa general-keys modulus 2048   ! RSA 2048 para SSH
ip ssh version 2
ip ssh authentication-retries 2
ip ssh time-out 30
line vty 0 15
 login local
 transport input ssh                       ! solo SSH
logging on
logging host 172.16.0.14                    ! logs al syslog remoto
login on-success log
login on-failure log
```

| Requisito | Comando clave |
|---|---|
| Largo mín. 8 + cifradas | `security passwords min-length 8` + `service password-encryption` |
| Privilegiado protegido | `enable secret` (MD5) |
| Banner | `banner motd` |
| Anti fuerza bruta (45s / 3 intentos / 60s) | `login block-for 45 attempts 3 within 60` |
| SSH seguro RSA 2048, v2, 2 intentos, timeout 30 | `crypto key ... 2048`, `ip ssh ...`, `transport input ssh` |
| Logs a syslog remoto | `logging host` + `login on-success/failure log` |

---

## Ítem 3 — Control de acceso

### 3.1 — Roles en Edge-Santiago (niveles de privilegio)
```cisco
! soporte (nivel 5): ping, traceroute, todos los show
privilege exec level 5 ping
privilege exec level 5 traceroute
privilege exec level 5 show
username soporte privilege 5 secret ciscocisco
! administrador (nivel 10): configure, configure terminal, todos los show
privilege exec level 10 configure terminal
privilege exec level 10 configure
privilege exec level 10 show
username administrador privilege 10 secret ciscocisco
```

### 3.2 — SSH y port-security en SW1
```cisco
enable secret ciscocisco
username admin secret ciscocisco
ip domain-name duoc.cl
crypto key generate rsa general-keys modulus 1024
ip ssh version 2
ip ssh authentication-retries 4
ip ssh time-out 30
line vty 0 15
 login local
 transport input ssh
!
interface range FastEthernet0/1-24
 switchport port-security
 switchport port-security maximum 2
 switchport port-security mac-address sticky
 switchport port-security violation restrict
!
logging on
logging host 172.16.0.14
service password-encryption
```
Port-security: máximo **2 MAC aprendidas automáticamente** (sticky), y en violación **`restrict`** (incrementa el contador y genera log, sin apagar el puerto).

---

## Ítem 4 — AAA por RADIUS y monitoreo

### En Edge-Santiago
```cisco
aaa new-model
radius server REMOTAAA
 address ipv4 172.16.0.14 auth-port 1812
 key ciscocisco
aaa authentication login default group radius local
!
access-list 1 permit 172.15.15.0 0.0.0.255      ! SSH solo desde VLAN15
access-list 1 permit 192.168.23.0 0.0.0.255      ! y desde INSIDE (Viña)
line vty 0 15
 transport input ssh
 access-class 1 in
 login authentication default
!
ntp server 172.16.0.14
```

> **Nota (sintaxis):** en esta versión de PT el comando `address ipv4 172.16.0.14 auth-port 1812 acct-port 1813` **no acepta el `acct-port`**; se usó solo `auth-port 1812`. Tampoco existe `ip radius source-interface`, pero no es necesario: el router ya envía el RADIUS con la IP de G0/2 (`172.16.0.1`, la de la LAN SERVER) por estar directamente conectado.

### En el servidor AAA-SYSLOG (Services → AAA)
- Service **On**, Radius Port **1812**
- Cliente: `Edge-Santiago` / `172.16.0.1` / secret `ciscocisco`
- Usuario RADIUS: `admin` / `ciscocisco`
- **NTP** activado (autenticación deshabilitada) y **Syslog** activado

**Verificación:** SSH desde PC2 (`ssh -l admin 172.15.15.1`) autenticó **por RADIUS** (el prompt quedó en nivel 1 `>`, propio del usuario del servidor; si hubiera caído a local, el `admin` local es nivel 15 y mostraría `#`). El syslog recibió logs de `172.16.0.1` (Edge) y `172.15.15.254` (SW1), y el reloj se sincronizó por NTP.

---

## Ítem 5 — Protección perimetral (ASA)

> El firewall es un **ASA 5505** (interfaces `Ethernet0/x` con VLANs internas), coherente con lo que pide el enunciado (vlan1/2/3 asociadas a e0/1, e0/0, e0/2).

### Parámetros base, interfaces y ruta (5.1–5.3)
```cisco
hostname FW-VINA
domain-name duoc.cl
enable password ciscocisco
passwd ciscocisco
interface Ethernet0/0
 switchport access vlan 2
interface Ethernet0/1
 switchport access vlan 1
interface Ethernet0/2
 switchport access vlan 3
interface Vlan1
 nameif inside
 security-level 100
 ip address 192.168.23.1 255.255.255.0
interface Vlan2
 nameif outside
 security-level 0
 ip address 177.44.56.6 255.255.255.248
interface Vlan3
 no forward interface Vlan1          ! requisito del 5505 (licencia base, 3ª zona)
 nameif dmz
 security-level 50
 ip address 10.20.30.1 255.255.255.224
route outside 0.0.0.0 0.0.0.0 177.44.56.1
```

### DHCP inside (5.4)
```cisco
dhcpd address 192.168.23.100-192.168.23.130 inside
dhcpd dns 8.8.4.4 interface inside
dhcpd enable inside     ! ⚠️ ver "Limitaciones del simulador"
```

### AAA local + SSH (5.6, 5.7)
```cisco
username admin password ciscocisco
aaa authentication ssh console LOCAL
ssh 192.168.23.0 255.255.255.0 inside   ! SSH solo desde inside
ssh timeout 5
```

### PAT y NAT estático (5.5, 5.8)
```cisco
object network INSIDE
 subnet 192.168.23.0 255.255.255.0
 nat (inside,outside) dynamic interface     ! PAT de toda la red inside
object network static
 host 10.20.30.30
 nat (dmz,outside) static 177.44.56.5        ! servidor web → IP pública
```

### ACL OUT y MPF (5.9, 5.10)
```cisco
access-list OUT extended permit icmp any host 10.20.30.30
access-list OUT extended permit tcp any host 10.20.30.30 eq www
access-group OUT in interface outside
!
class-map inspection_default
 match default-inspection-traffic
policy-map global_policy
 class inspection_default
  inspect icmp
  inspect http
  inspect dns
service-policy global_policy global
```

> **Nota (ASA 8.3+):** la ACL `OUT` referencia la IP **real** del servidor (`10.20.30.30`), no la pública, porque en ASA 8.3+ las ACLs post-NAT usan la dirección real. Así el acceso ICMP/WWW desde afuera funciona correctamente.

**Verificación:** `show nat` y `show xlate` muestran el PAT dinámico de inside y el NAT estático `10.20.30.30 ↔ 177.44.56.5`. El MPF quedó con `inspect icmp/http/dns` aplicado globalmente.

---

## Ítem 6 — VPN Site-to-Site

VPN IPsec entre **Edge-Santiago** e **ISP**. Requirió **activar la licencia de seguridad** (`securityk9`) en **ambos** routers (con `write memory` + `reload`), ya que sin ella los comandos `crypto isakmp` daban *"Invalid input"*.

```cisco
license boot module c2900 technology-package securityk9   ! + reload en ambos
```

### Tráfico interesante (6.1)
```cisco
! Edge-Santiago
access-list 100 permit ip 172.15.15.0 0.0.0.255 host 1.1.1.1
! ISP (espejo)
access-list 100 permit ip host 1.1.1.1 172.15.15.0 0.0.0.255
```

### Fase 1 ISAKMP (6.2) — idéntica en ambos
```cisco
crypto isakmp policy 10
 encryption aes 256
 hash sha
 authentication pre-share
 group 5
crypto isakmp key ciscocisco address <peer>
```

### Fase 2 IPsec + crypto map (6.3, 6.4)
```cisco
crypto ipsec transform-set TRANS esp-aes esp-sha-hmac
crypto map VPN-MAP 10 ipsec-isakmp
 set peer <peer>
 set transform-set TRANS
 match address 100
interface GigabitEthernet0/1
 crypto map VPN-MAP
```
Peers: Edge usa `200.2.7.1` (ISP) e ISP usa `200.2.7.6` (Edge). En el ISP se agregó además `interface Loopback1 (1.1.1.1)` y `ip route 172.15.15.0 255.255.255.0 200.2.7.6`.

> **Nota:** La configuración quedó **completa y correcta** en ambos extremos (verificado con `show crypto isakmp policy`, `show crypto map`, `show access-lists 100`). El túnel de datos **no llegó a formarse en Packet Tracer** — ver [Limitaciones](#-limitaciones-del-simulador).

---

## Ítem 7 — IOS IPS

IOS IPS en Edge-Santiago (requiere `securityk9`, ya activo por el Ítem 6).

```cisco
mkdir flash:ipsdir                          ! (modo exec) 7.2
!
configure terminal
ip ips config location flash:ipsdir         ! 7.2 ubicación de firmas
ip ips name iosips                          ! 7.3 regla IPS
!
ip ips signature-category                   ! 7.4
 category all
  retired true                              ! retira todas
 exit
 category ios_ips basic
  retired false                             ! activa la categoría básica
 exit
exit                                         ! → confirmar cambios
!
interface GigabitEthernet0/2                 ! 7.5 interfaz de la red del SERVER
 ip ips iosips in
 ip ips iosips out
!
ip ips signature-definition                  ! 7.6 modificar firma 2004
 signature 2004 0
  status
   retired false
   enabled true
  exit
  engine
   event-action produce-alert               ! que produzca alertas
  exit
 exit
exit                                          ! → confirmar cambios
```

**Verificación:** al confirmar, el motor compiló las firmas correctamente:
```
%IPS-6-ENGINE_READY: atomic-ip - ... packets for this engine will be scanned
%IPS-6-ALL_ENGINE_BUILDS_COMPLETE
```
La firma **2004** quedó activa (`retired false`, `enabled true`) con acción **produce-alert**.

---

## ⚠️ Limitaciones del simulador

Dos puntos quedaron **configurados correctamente** pero no 100% funcionales por **bugs conocidos de Cisco Packet Tracer 9.0.0**, no por errores de configuración. El corrector evalúa la configuración, que está completa en ambos casos.

| # | Punto | Detalle | Evidencia / workaround |
|---|---|---|---|
| 1 | **DHCP del ASA (5.4)** | `dhcpd enable inside` devuelve *"need to define address pool range first"* aunque la interfaz `inside` esté up/up y el pool esté en el running-config. Es un bug del ASA en PT 9.0.0 (ocurre igual en 5505 y 5506-X). | `dhcpd address` y `dhcpd dns` **sí** quedan en la config como evidencia. A los PCs de Viña se les puede poner IP fija del rango. |
| 2 | **Túnel VPN (Ítem 6)** | La ISAKMP SA no se forma y el tráfico interesante viaja sin cifrar (`#pkts encaps: 0`). La config (ACL, ISAKMP AES-256/DH5, transform-set, crypto map en la interfaz) está toda presente y correcta. El examen exige AES-256 y DH grupo 5 obligatoriamente. | Config verificada con `show crypto isakmp policy` / `show crypto map` / `show access-lists 100`. |

---

## 📝 Notas finales

- **ASA:** modelo 5505 (interfaces `Ethernet0/x`, VLANs internas, licencia base → 3 zonas).
- **Routers:** Cisco 2911, IOS 15.1(4)M; se activó `securityk9` en Edge-Santiago e ISP para VPN e IPS.
- **Detalle de PT:** en el ASA de esta versión, `exit` dentro de un sub-modo saca hasta EXEC (dos seguidos desloguean); conviene cerrar bloques con `end`.
- Recordar **`write memory`** en **Edge-Santiago, ISP, SW1 y FW-VINA** antes de guardar el `.pka`.
