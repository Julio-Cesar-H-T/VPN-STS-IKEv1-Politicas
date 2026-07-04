# VPN-STS-IKEv1-Politicas# VPN Site to Site con Túnel GRE sobre IPSec (IKEv1)

## Información del Proyecto

| Campo           | Detalle                               |
| --------------- | ------------------------------------- |
| **Estudiante**  | Julio César Hernández Tibrey          |
| **Matrícula**   | 2025-0702                             |
| **Institución** | Instituto Tecnológico de Las Americas |
| **Asignatura**  | Seguridad de Redes                    |
| **Profesor**    | Jonathan Esteban Rondón Corniel       |

---

## Objetivo

Establecer un túnel VPN site-to-site punto a punto entre **R1** (Sitio A) y **R2** (Sitio B) utilizando un **túnel GRE** (Generic Routing Encapsulation) protegido por **IPSec con IKEv1**. En este modelo, GRE encapsula primero el tráfico IP original, generando un paquete que soporta tráfico multicast/broadcast y enrutamiento dinámico; posteriormente, IPSec protege específicamente ese tráfico GRE mediante un crypto map aplicado sobre la interfaz física.

El objetivo funcional es comunicar de forma segura las VLANs de Finanzas/RRHH (Sitio A) con las VLANs de IT/Admin (Sitio B), usando una arquitectura de doble encapsulación (GRE + IPSec) compatible con protocolos de enrutamiento dinámico y tráfico multicast, algo que un VTI IPSec puro no soporta de forma nativa.

---

## Contexto: IKEv1 (Internet Key Exchange version 1)

- **Lanzamiento:** 1998 (RFC 2409), fue la base de la seguridad en internet por décadas.
- **Vulnerabilidad:** Su mayor debilidad es el **"Aggressive Mode"**. Si se configura así, el router envía el hash de la clave pre-compartida (PSK) sin encriptar, lo que permite ataques de fuerza bruta fuera de línea (offline) para crackear la contraseña.
- **Rendimiento:** Es un protocolo "pesado". Para establecer una conexión, puede necesitar entre **6 y 9 mensajes** de ida y vuelta, lo que lo hace más lento y propenso a fallas en enlaces inestables.

---

## Descripción: GRE sobre IPSec

**GRE (Generic Routing Encapsulation)** es un protocolo de tunelización que "envuelve" cualquier tipo de paquete para que pueda viajar de un punto a otro como si estuvieran en la misma red local.

### ¿Por qué GRE sobre IPSec?

GRE por sí solo **no proporciona encriptación**. Al combinarlo con IPSec, obtenemos:
- **Flexibilidad de GRE:** Soporte para multicast, broadcast y protocolos de enrutamiento
- **Seguridad de IPSec:** Encriptación y autenticación del tráfico

### Características principales:
- Doble encapsulación: primero GRE, luego IPSec protege el GRE
- El crypto map se aplica sobre la interfaz física, no sobre Tunnel0
- Soporta protocolos de enrutamiento dinámico y tráfico multicast a través del túnel
- Requiere una ACL de tráfico interesante, pero esta vez basada en el protocolo GRE, no en las subredes LAN

---

## Modo Transport vs Tunnel

| Aspecto | Mode Transport | Mode Tunnel |
|---------|----------------|--------------|
| Overhead | Menor | Mayor |
| Uso con GRE | Recomendado (GRE ya encapsula) | Redundante (doble encapsulación IP) |

> En este escenario se usa **`mode transport`**, ya que GRE ya cumple la función de "tunneling"; IPSec en modo transporte solo añade su cabecera ESP sin re-encapsular el paquete completo.

---

## Topología

```
LAN A (10.7.2.0/27)                                                     LAN B (10.7.2.32/27)
   VLAN 10 / VLAN 20                                                       VLAN 30 / VLAN 40
          |                                                                        |
          R1 ──── Tunnel0 (172.16.12.1/30, GRE) ════ R2 ──── Tunnel0 (172.16.12.2/30, GRE)
          |   e0/0: 200.7.2.1/30 ── [crypto map] ── ISP ── [crypto map] ── 200.7.2.6/30: e0/1
```

### Tabla de Direccionamiento

| Dispositivo | Interfaz | Dirección IP | Máscara         | VLAN/Descripción                           |
| ----------- | -------- | ------------ | --------------- | ------------------------------------------ |
| ISP         | e0/0     | 200.7.2.2    | 255.255.255.252 | Enlace a R1                                |
| ISP         | e0/1     | 200.7.2.5    | 255.255.255.252 | Enlace a R2                                |
| R1          | e0/0     | 200.7.2.1    | 255.255.255.252 | WAN — interfaz con crypto map (no Tunnel0) |
| R1          | Tunnel0  | 172.16.12.1  | 255.255.255.252 | Túnel GRE                                  |
| R1          | e0/1.10  | 10.7.2.1     | 255.255.255.240 | VLAN 10 - FINANZAS                         |
| R1          | e0/2.20  | 10.7.2.17    | 255.255.255.240 | VLAN 20 - RRHH                             |
| R2          | e0/1     | 200.7.2.6    | 255.255.255.252 | WAN — interfaz con crypto map (no Tunnel0) |
| R2          | Tunnel0  | 172.16.12.2  | 255.255.255.252 | Túnel GRE                                  |
| R2          | e0/0.30  | 10.7.2.33    | 255.255.255.240 | VLAN 30 - IT                               |
| R2          | e0/0.40  | 10.7.2.49    | 255.255.255.240 | VLAN 40 - ADMIN                            |

### Pools DHCP

**Sitio 1 (R1):**

| Pool     | Red          | Gateway   | DNS     |
| -------- | ------------ | --------- | ------- |
| FINANZAS | 10.7.2.0/28  | 10.7.2.1  | 8.8.8.8 |
| RRHH     | 10.7.2.16/28 | 10.7.2.17 | 8.8.8.8 |

**Sitio 2 (R2):**

| Pool | Red | Gateway | DNS |
|------|-----|---------|-----|
| IT | 10.7.2.32/28 | 10.7.2.33 | 8.8.8.8 |
| ADMIN | 10.7.2.48/28 | 10.7.2.49 | 8.8.8.8 |

`[ESPACIO PARA CAPTURA: Topología completa en PNETLab, incluyendo interfaz Tunnel0 en modo GRE]`

---

## Configuración Base de la Infraestructura

### ISP

```
conf t
hostname ISP
int e0/0
 ip add 200.7.2.2 255.255.255.252
 no shut
int e0/1
 ip add 200.7.2.5 255.255.255.252
 no shut
exit
```

> El router ISP **no conoce las redes privadas** (10.7.2.0/24). Simula un proveedor de internet real que solo enruta entre sus dos interfaces de tránsito directamente conectadas.

### R1 (Router on a Stick + DHCP)

```
conf t
hostname R1
int e0/0
 ip add 200.7.2.1 255.255.255.252
 no shut
exit
ip route 0.0.0.0 0.0.0.0 200.7.2.2
ip dhcp excluded-address 10.7.2.1
ip dhcp excluded-address 10.7.2.17
ip dhcp pool FINANZAS
 network 10.7.2.0 255.255.255.240
 default-router 10.7.2.1
 dns-server 8.8.8.8
ip dhcp pool RRHH
 network 10.7.2.16 255.255.255.240
 default-router 10.7.2.17
 dns-server 8.8.8.8
int e0/1
 no shut
int e0/1.10
 encapsulation dot1Q 10
 ip add 10.7.2.1 255.255.255.240
 no shut
int e0/2
 no shut
int e0/2.20
 encapsulation dot1Q 20
 ip add 10.7.2.17 255.255.255.240
 no shut
exit
```

### SW-1 (Acceso PC1 - VLAN 10)

```
conf t
hostname SW-1
vlan 10
 name FINANZAS
exit
int e0/0
 switchport trunk encapsulation dot1q
 switchport mode trunk
 no shut
int e0/1
 switchport mode access
 switchport access vlan 10
 spanning-tree portfast
 no shut
exit
```

### SW-2 (Acceso PC2 - VLAN 20)

```
conf t
hostname SW-2
vlan 20
exit
int e0/0
 switchport trunk encapsulation dot1q
 switchport mode trunk
 no shut
int e0/1
 switchport mode access
 switchport access vlan 20
 spanning-tree portfast
 no shut
exit
```

### R2 (Router on a Stick + DHCP)

```
conf t
hostname R2
int e0/1
 ip add 200.7.2.6 255.255.255.252
 no shut
exit
ip route 0.0.0.0 0.0.0.0 200.7.2.5
ip dhcp excluded-address 10.7.2.33
ip dhcp excluded-address 10.7.2.49
ip dhcp pool IT
 network 10.7.2.32 255.255.255.240
 default-router 10.7.2.33
 dns-server 8.8.8.8
ip dhcp pool ADMIN
 network 10.7.2.48 255.255.255.240
 default-router 10.7.2.49
 dns-server 8.8.8.8
int e0/0
 no shut
int e0/0.30
 encapsulation dot1Q 30
 ip add 10.7.2.33 255.255.255.240
 no shut
int e0/0.40
 encapsulation dot1Q 40
 ip add 10.7.2.49 255.255.255.240
 no shut
exit
```

### SW-3 (Trunk + Acceso PC3/PC4)

```
conf t
hostname SW-3
vlan 30
 name IT
vlan 40
 name ADMIN
exit
int e0/0
 switchport trunk encapsulation dot1q
 switchport mode trunk
 no shut
int e0/1
 switchport mode access
 switchport access vlan 40
 spanning-tree portfast
 no shut
int e0/2
 switchport mode access
 switchport access vlan 30
 spanning-tree portfast
 no shut
exit
```

### Verificación de la base

```
show ip interface brief
show vlan brief
show interfaces trunk
```

- Las sub-interfaces `e0/1.10`, `e0/2.20` (R1) y `e0/0.30`, `e0/0.40` (R2) deben estar `up/up`.
- Los PCs deben recibir IP por DHCP dentro de su VLAN correspondiente.
- **Sin VPN aplicada**, el ping entre LAN A y LAN B debe fallar (el ISP no tiene ruta hacia 10.7.2.0/24). Este es el comportamiento esperado antes de configurar la VPN.

`[ESPACIO PARA CAPTURA: ping fallido entre LAN A y LAN B antes de aplicar la VPN]`

---

## Configuración GRE sobre IPSec con IKEv1

### Configuración R1

```
conf t
crypto isakmp policy 10
 encryption aes 256
 hash sha256
 authentication pre-share
 group 14
 lifetime 86400
exit
crypto isakmp key 2025-0702 address 200.7.2.6

crypto ipsec transform-set TS_GRE esp-aes 256 esp-sha256-hmac
 mode transport
exit

ip access-list extended GRE-TRAFFIC
 permit gre host 200.7.2.1 host 200.7.2.6
exit

crypto map CMAP-GRE 10 ipsec-isakmp
 set peer 200.7.2.6
 set transform-set TS_GRE
 match address GRE-TRAFFIC
exit

interface Tunnel0
 ip address 172.16.12.1 255.255.255.252
 tunnel source e0/0
 tunnel destination 200.7.2.6
 tunnel mode gre ip
 no shutdown
exit

interface e0/0
 crypto map CMAP-GRE
exit

ip route 10.7.2.32 255.255.255.224 Tunnel0
end
write memory
```

`[ESPACIO PARA CAPTURA: configuración completa aplicada en R1 — terminal]`

### Configuración R2

```
conf t
crypto isakmp policy 10
 encryption aes 256
 hash sha256
 authentication pre-share
 group 14
 lifetime 86400
exit
crypto isakmp key 2025-0702 address 200.7.2.1

crypto ipsec transform-set TS_GRE esp-aes 256 esp-sha256-hmac
 mode transport
exit

ip access-list extended GRE-TRAFFIC
 permit gre host 200.7.2.6 host 200.7.2.1
exit

crypto map CMAP-GRE 10 ipsec-isakmp
 set peer 200.7.2.1
 set transform-set TS_GRE
 match address GRE-TRAFFIC
exit

interface Tunnel0
 ip address 172.16.12.2 255.255.255.252
 tunnel source e0/1
 tunnel destination 200.7.2.1
 tunnel mode gre ip
 no shutdown
exit

interface e0/1
 crypto map CMAP-GRE
exit

ip route 10.7.2.0 255.255.255.224 Tunnel0
end
write memory
```

`[ESPACIO PARA CAPTURA: configuración completa aplicada en R2 — terminal]`

---

## Explicación de los Componentes

### Fase 1 (ISAKMP Policy)
| Parámetro | Valor | Descripción |
|-----------|-------|-------------|
| `encryption aes 256` | AES-256 | Algoritmo de encriptación |
| `hash sha256` | SHA-256 | Algoritmo de integridad |
| `authentication pre-share` | PSK | Método de autenticación |
| `group 14` | DH Group 14 (2048-bit) | Grupo Diffie-Hellman |
| `lifetime 86400` | 24 horas | Tiempo de vida de la SA de Fase 1 |
| `crypto isakmp key 2025-0702` | PSK personalizada | Debe coincidir exactamente en ambos peers |

### Fase 2 (Transform Set)
| Parámetro | Descripción |
|-----------|-------------|
| `esp-aes 256` | Encriptación ESP con AES-256 |
| `esp-sha256-hmac` | Autenticación/Integridad con SHA-256 |
| `mode transport` | GRE ya encapsula el paquete; IPSec solo añade su cabecera ESP, evitando doble overhead |

### Interfaz de túnel GRE
| Parámetro | Descripción |
|---|---|
| `tunnel source` / `tunnel destination` | IPs públicas que forman los extremos del túnel |
| `tunnel mode gre ip` | Define el túnel como GRE puro (sin cifrado propio) |

### Tráfico interesante (ACL — protocolo GRE)
| Router | ACL | Regla |
|---|---|---|
| R1 | `GRE-TRAFFIC` | `permit gre host 200.7.2.1 host 200.7.2.6` |
| R2 | `GRE-TRAFFIC` | `permit gre host 200.7.2.6 host 200.7.2.1` |

> A diferencia del modelo policy-based "puro", esta ACL no protege las subredes LAN directamente: protege el **protocolo GRE** entre las IPs públicas. El paquete GRE (que ya contiene el tráfico de usuario dentro) es lo que se cifra.

### Punto de aplicación del crypto map
| Router | Interfaz |
|---|---|
| R1 | e0/0 (física, no Tunnel0) |
| R2 | e0/1 (física, no Tunnel0) |

### Enrutamiento hacia el túnel
| Router | Ruta estática |
|---|---|
| R1 | `ip route 10.7.2.32 255.255.255.224 Tunnel0` |
| R2 | `ip route 10.7.2.0 255.255.255.224 Tunnel0` |

---

## Comparación entre los Tres Modelos

| Aspecto | Policy-Based | Route-Based (VTI) | GRE over IPSec |
|---|---|---|---|
| Tipo de túnel | Ninguno (crypto map directo) | `tunnel mode ipsec ipv4` | `tunnel mode gre ip` |
| Dónde se aplica IPSec | Crypto map en interfaz física | `tunnel protection` en Tunnel0 | Crypto map en interfaz física |
| Qué cifra IPSec | Tráfico IP de usuario directamente | Tráfico IP de usuario directamente | El paquete GRE ya encapsulado |
| Modo de transform-set | Tunnel | Tunnel | **Transport** |
| Tráfico interesante (ACL) | Subredes LAN específicas | No aplica (todo va a Tunnel0) | Protocolo GRE entre IPs públicas |
| Soporta multicast / enrutamiento dinámico | No | No | **Sí** |

---

## Verificación

**Estado del Túnel:**
```
show interface tunnel 0
```

**Verificar estado de ISAKMP (Fase 1):**
```
show crypto isakmp sa
```

**Sesión de Cifrado:**
```
show crypto session
```

**Tráfico Protegido:**
```
show crypto ipsec sa | include pkts
```

**Verificar el crypto map aplicado en la interfaz física:**
```
show crypto map
```

**Prueba de Salto:**
```
traceroute 10.7.2.34 source e0/1.10
```

### Resultado esperado:

```
IPv4 Crypto ISAKMP SA
dst             src             state          conn-id status
200.7.2.6       200.7.2.1       QM_IDLE           1001 ACTIVE
```

> En `show crypto ipsec sa`, el `local ident`/`remote ident` debe mostrar el protocolo **GRE** entre 200.7.2.1 y 200.7.2.6 (no subredes LAN ni IP genérico), confirmando que IPSec protege específicamente el tráfico GRE y no el tráfico de usuario directamente.

`[ESPACIO PARA CAPTURA: show interface tunnel 0 — estado up/up, encapsulación GRE]`

`[ESPACIO PARA CAPTURA: show crypto isakmp sa — estado QM_IDLE]`

`[ESPACIO PARA CAPTURA: show crypto ipsec sa — local/remote ident con protocolo GRE]`

`[ESPACIO PARA CAPTURA: show crypto map — crypto map aplicado en interfaz física]`

`[ESPACIO PARA CAPTURA: ping y/o traceroute exitoso entre LAN A y LAN B]`

---

## Notas y Consideraciones

- El uso de `mode transport` en lugar de `mode tunnel` es la diferencia técnica clave de este escenario: evita una segunda encapsulación de cabecera IP, ya que GRE cumple esa función.
- Esta arquitectura es la base típica para escenarios donde se requiere correr OSPF o EIGRP entre sitios remotos sobre una VPN, algo que un VTI IPSec puro no soporta de forma nativa sin protocolos adicionales.
- Nombres de objetos y clave precompartida personalizados con la matrícula del autor (0702) para trazabilidad del laboratorio.
