# VPN Site to Site Basada en Políticas (IKEv1)

## Información del Proyecto

| Campo           | Detalle                               |
| --------------- | ------------------------------------- |
| **Estudiante**  | Julio César Hernández Tibrey          |
| **Matrícula**   | 2025-0702                             |
| **Institución** | Instituto Tecnológico de Las Americas |
| **Asignatura**  | Seguridad de Redes                    |
| **Profesor**    | Jonathan Esteban Rondón Corniel       |

---



---

## Objetivo

Establecer un túnel VPN site-to-site punto a punto entre los routers **R1** (Sitio A) y **R2** (Sitio B) utilizando **IPSec con IKEv1**, donde el tráfico que debe cifrarse se determina mediante una **lista de acceso (ACL) de tráfico interesante**, aplicada a través de un **crypto map** sobre la interfaz pública de cada router.

El objetivo funcional es que los hosts de la **LAN A** (VLAN 10 — Finanzas, VLAN 20 — RRHH) puedan comunicarse de forma cifrada con los hosts de la **LAN B** (VLAN 30 — IT, VLAN 40 — Admin) a través de la red de tránsito simulada por el router ISP, el cual no tiene visibilidad de las redes privadas.

---

## Contexto: IKEv1 (Internet Key Exchange version 1)

- **Lanzamiento:** 1998 (RFC 2409), fue la base de la seguridad en internet por décadas.
- **Vulnerabilidad:** Su mayor debilidad es el **"Aggressive Mode"**. Si se configura así, el router envía el hash de la clave pre-compartida (PSK) sin encriptar, lo que permite ataques de fuerza bruta fuera de línea (offline) para crackear la contraseña.
- **Rendimiento:** Es un protocolo "pesado". Para establecer una conexión, puede necesitar entre **6 y 9 mensajes** de ida y vuelta, lo que lo hace más lento y propenso a fallas en enlaces inestables.

---

## Descripción

Una VPN basada en políticas (Policy-Based VPN) utiliza Access Control Lists (ACLs) para definir qué tráfico debe ser encriptado y enviado a través del túnel IPSec. Este método es el más tradicional y utiliza **Crypto Maps** para vincular las políticas de seguridad con las interfaces físicas.

### Características principales:
- El tráfico "interesante" se define mediante ACLs
- Cada política de seguridad requiere su propia entrada en el Crypto Map
- Es el método más compatible con equipos legacy
- Ideal para topologías simples punto a punto
- No existe interfaz de túnel: el cifrado se activa al vuelo cuando un paquete coincide con la ACL

---

## Topología

![Topología VPN Site to Site Basada en Políticas](img/topologia-vpn-policy-based.png)

```
LAN A (10.7.2.0/27)        crypto map           crypto map        LAN B (10.7.2.32/27)
   VLAN 10 / VLAN 20  ──── R1 (e0/0) ──── ISP ──── R2 (e0/1) ────  VLAN 30 / VLAN 40
                        200.7.2.1/30          200.7.2.6/30
```

### Tabla de Direccionamiento

| Dispositivo | Interfaz | Dirección IP | Máscara | VLAN/Descripción |
|-------------|----------|--------------|---------|-------------------|
| ISP | e0/0 | 200.7.2.2 | 255.255.255.252 | Enlace a R1 |
| ISP | e0/1 | 200.7.2.5 | 255.255.255.252 | Enlace a R2 |
| R1 | e0/0 | 200.7.2.1 | 255.255.255.252 | WAN (hacia ISP) — interfaz con crypto map |
| R1 | e0/1.10 | 10.7.2.1 | 255.255.255.240 | VLAN 10 - FINANZAS |
| R1 | e0/2.20 | 10.7.2.17 | 255.255.255.240 | VLAN 20 - RRHH |
| R2 | e0/1 | 200.7.2.6 | 255.255.255.252 | WAN (hacia ISP) — interfaz con crypto map |
| R2 | e0/0.30 | 10.7.2.33 | 255.255.255.240 | VLAN 30 - IT |
| R2 | e0/0.40 | 10.7.2.49 | 255.255.255.240 | VLAN 40 - ADMIN |

### Pools DHCP

**Sitio 1 (R1):**

| Pool | Red | Gateway | DNS |
|------|-----|---------|-----|
| FINANZAS | 10.7.2.0/28 | 10.7.2.1 | 8.8.8.8 |
| RRHH | 10.7.2.16/28 | 10.7.2.17 | 8.8.8.8 |

**Sitio 2 (R2):**

| Pool | Red | Gateway | DNS |
|------|-----|---------|-----|
| IT | 10.7.2.32/28 | 10.7.2.33 | 8.8.8.8 |
| ADMIN | 10.7.2.48/28 | 10.7.2.49 | 8.8.8.8 |

> Resumen de tráfico interesante usado en la VPN: LAN A = `10.7.2.0/27` (cubre VLAN 10 y 20) · LAN B = `10.7.2.32/27` (cubre VLAN 30 y 40)

`[ESPACIO PARA CAPTURA: Topología completa en PNETLab]`

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

## Configuración VPN IPSec con IKEv1

### Paso 1: Definición del Tráfico Interesante (ACL)

**En R1:**

```
conf t
ip access-list extended VPN-TRAFFIC
 permit ip 10.7.2.0 0.0.0.31 10.7.2.32 0.0.0.31
exit
```

**En R2:**

```
conf t
ip access-list extended VPN-TRAFFIC
 permit ip 10.7.2.32 0.0.0.31 10.7.2.0 0.0.0.31
exit
```

> **Nota:** Las ACLs son espejo entre sí. Lo que es origen en R1, es destino en R2 y viceversa.

`[ESPACIO PARA CAPTURA: configuración de la ACL VPN-TRAFFIC en R1]`

### Paso 2: Configuración IKEv1 en R1

```
conf t
! Fase 1: ISAKMP Policy (Negociación IKE)
crypto isakmp policy 10
 encryption aes 256
 hash sha256
 authentication pre-share
 group 14
 lifetime 86400
exit

! Clave pre-compartida (PSK) apuntando al peer remoto
crypto isakmp key 2025-0702 address 200.7.2.6

! Fase 2: IPsec Transform Set
crypto ipsec transform-set TS_R1-R2 esp-aes 256 esp-sha256-hmac
 mode tunnel
exit

! Crypto Map (vincula todo)
crypto map CMAP-R1 10 ipsec-isakmp
 set peer 200.7.2.6
 set transform-set TS_R1-R2
 match address VPN-TRAFFIC
exit

! Aplicar Crypto Map a la interfaz WAN
int e0/0
 crypto map CMAP-R1
exit
end
write memory
```

`[ESPACIO PARA CAPTURA: configuración completa aplicada en R1 — terminal]`

### Paso 3: Configuración IKEv1 en R2

```
conf t
! Fase 1: ISAKMP Policy (Negociación IKE)
crypto isakmp policy 10
 encryption aes 256
 hash sha256
 authentication pre-share
 group 14
 lifetime 86400
exit

! Clave pre-compartida (PSK) apuntando al peer remoto
crypto isakmp key 2025-0702 address 200.7.2.1

! Fase 2: IPsec Transform Set
crypto ipsec transform-set TS_R2-R1 esp-aes 256 esp-sha256-hmac
 mode tunnel
exit

! Crypto Map (vincula todo)
crypto map CMAP-R2 10 ipsec-isakmp
 set peer 200.7.2.1
 set transform-set TS_R2-R1
 match address VPN-TRAFFIC
exit

! Aplicar Crypto Map a la interfaz WAN
int e0/1
 crypto map CMAP-R2
exit
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
| `mode tunnel` | Modo túnel (encripta todo el paquete IP original) |

### Tráfico interesante (ACL)
| Router | ACL | Regla |
|---|---|---|
| R1 | `VPN-TRAFFIC` | `permit ip 10.7.2.0 0.0.0.31 10.7.2.32 0.0.0.31` |
| R2 | `VPN-TRAFFIC` | `permit ip 10.7.2.32 0.0.0.31 10.7.2.0 0.0.0.31` |

### Crypto Map
| Parámetro | Descripción |
|---|---|
| `set peer` | IP pública del router remoto |
| `set transform-set` | Vincula el conjunto de algoritmos de Fase 2 |
| `match address` | Vincula la ACL de tráfico interesante |
| Aplicación | Sobre la interfaz física pública (e0/0 en R1, e0/1 en R2), no sobre las sub-interfaces LAN |

---

## Verificación

**Verificar estado de ISAKMP (Fase 1):**
```
show crypto isakmp sa
```

**Verificar estado de IPSec (Fase 2):**
```
show crypto ipsec sa
```

**Verificar contadores de paquetes encriptados:**
```
show crypto ipsec sa | include pkts
```

**Verificar el crypto map aplicado:**
```
show crypto map
```

### Resultado esperado:

```
IPv4 Crypto ISAKMP SA
dst             src             state          conn-id status
200.7.2.6       200.7.2.1       QM_IDLE           1001 ACTIVE
```

> En `show crypto ipsec sa`, el `local ident`/`remote ident` debe mostrar exactamente las subredes definidas en la ACL (10.7.2.0/27 y 10.7.2.32/27) — característica propia del modelo policy-based: el cifrado está acotado a lo que indica la ACL, no a todo el tráfico del túnel.

`[ESPACIO PARA CAPTURA: show crypto isakmp sa — estado QM_IDLE]`

`[ESPACIO PARA CAPTURA: show crypto ipsec sa — local/remote ident y contadores de pkts]`

`[ESPACIO PARA CAPTURA: show crypto map en R1 y R2]`

`[ESPACIO PARA CAPTURA: ping exitoso entre un host de LAN A y un host de LAN B]`

---

## Notas y Consideraciones

- Si se modifican las VLANs o se agregan nuevas redes a alguna LAN, la ACL `VPN-TRAFFIC` debe actualizarse manualmente en ambos extremos — este es el principal punto débil del modelo policy-based.
- El ISP no requiere ninguna configuración relacionada con la VPN: solo observa paquetes ESP (protocolo IP 50) entre las IPs públicas 200.7.2.1 y 200.7.2.6.
- Nombres de objetos y clave precompartida personalizados con la matrícula del autor (0702) para trazabilidad del laboratorio.
