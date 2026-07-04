# 🔐 DMVPN Fase 2 (IKEv1 + EIGRP)

## 1. Configuración del ISP e Infraestructura (Común)

### 🎯 Objetivo de la VPN

La **DMVPN Fase 2** se utiliza para conectar varias sucursales usando una topología **Hub and Spoke**, pero permitiendo que los Spokes se comuniquen directamente entre ellos.

Primero los Spokes se registran en el Hub usando **NHRP**. Luego, cuando un Spoke quiere comunicarse con otro, el Hub le indica la dirección del otro Spoke y se crea un túnel directo entre ellos.

### 📌 Para qué se usa

- Conectar muchas sucursales a una red central
- Reducir el tráfico que pasa por el Hub
- Permitir comunicación directa entre sucursales

> Por ejemplo, si una sucursal quiere comunicarse con otra, no tiene que pasar siempre por el Hub.

---

## 2. Topología

```
                        [ISP]
        e0/0            e0/1            e0/2
     1.1.1.2/30      2.2.2.1/30      3.3.3.1/30
        │                 │                │
     e0/0 (WAN)       e0/0 (WAN)       e0/0 (WAN)
   1.1.1.1/30        2.2.2.2/30       3.3.3.2/30
   [R1 - HUB]      [R2 - SPOKE1]    [R3 - SPOKE2]
   e0/1 (LAN)        e0/1 (LAN)       e0/1 (LAN)
   7.84.0.1/24       7.84.1.1/24      7.84.2.1/24
      │                 │                │
    LAN HUB          LAN SPOKE1       LAN SPOKE2
  7.84.0.0/24       7.84.1.0/24      7.84.2.0/24

           Tunnel0 (DMVPN) — Red 10.0.0.0/24
   HUB: 10.0.0.1   SPOKE1: 10.0.0.2   SPOKE2: 10.0.0.3
```

### Router ISP (El Transporte)

```
hostname ISP
!
interface Ethernet0/0
 ip address 1.1.1.2 255.255.255.252
 no shutdown
interface Ethernet0/1
 ip address 2.2.2.1 255.255.255.252
 no shutdown
interface Ethernet0/2
 ip address 3.3.3.1 255.255.255.252
 no shutdown
```

### PCs (VPCS)

| PC | Comando |
|----|---------|
| **PC1** | `ip 7.84.0.10 255.255.255.0 7.84.0.1` |
| **PC2** | `ip 7.84.1.10 255.255.255.0 7.84.1.1` |
| **PC3** | `ip 7.84.2.10 255.255.255.0 7.84.2.1` |

### Topología y Direccionamiento

**WAN (/30):**

| Router | Interfaz | IP | Conecta con |
|--------|----------|-----|--------------|
| R1 (HUB) | e0/0 | `1.1.1.1/30` | ISP e0/0 `1.1.1.2/30` |
| R2 (SPOKE1) | e0/0 | `2.2.2.2/30` | ISP e0/1 `2.2.2.1/30` |
| R3 (SPOKE2) | e0/0 | `3.3.3.2/30` | ISP e0/2 `3.3.3.1/30` |

**LAN:**

| Sede | Red | Gateway |
|------|-----|---------|
| HUB | `7.84.0.0/24` | `7.84.0.1` |
| SPOKE1 | `7.84.1.0/24` | `7.84.1.1` |
| SPOKE2 | `7.84.2.0/24` | `7.84.2.1` |

**Tunnel (DMVPN):**

| Router | Tunnel0 IP |
|--------|------------|
| HUB | `10.0.0.1` |
| SPOKE1 | `10.0.0.2` |
| SPOKE2 | `10.0.0.3` |

Red del túnel: `10.0.0.0/24`

---

## 3. DMVPN Fase 2 (IKEv1 + EIGRP)

### 🎯 Objetivo

Conectividad segura HUB-Spoke y capacidad spoke-to-spoke. En Fase 2, los spokes aprenden rutas con siguiente salto real.

> ⚠️ En el HUB, sobre Tunnel0, es esencial:
> ```
> no ip split-horizon eigrp 100
> no ip next-hop-self eigrp 100
> ```

### 🟢 Configuración HUB (R1)

```
hostname R1-HUB
!
interface Ethernet0/0
 ip address 1.1.1.1 255.255.255.252
 no shutdown
!
interface Ethernet0/1
 ip address 7.84.0.1 255.255.255.0
 no shutdown
!
crypto isakmp policy 10
 encr aes
 hash sha
 authentication pre-share
 group 2
 lifetime 86400
!
crypto isakmp key cisco123 address 0.0.0.0 0.0.0.0
!
crypto ipsec transform-set TS esp-aes esp-sha-hmac
 mode transport
!
crypto ipsec profile VPN_PROFILE
 set transform-set TS
!
interface Tunnel0
 ip address 10.0.0.1 255.255.255.0
 ip nhrp map multicast dynamic
 ip nhrp network-id 1
 tunnel source Ethernet0/0
 tunnel mode gre multipoint
 tunnel protection ipsec profile VPN_PROFILE
!
router eigrp 100
 network 10.0.0.0 0.0.0.255
 network 7.84.0.0 0.0.0.255
 no auto-summary
!
interface Tunnel0
 no ip split-horizon eigrp 100
 no ip next-hop-self eigrp 100
```

### 🔵 Configuración SPOKE1 (R2)

```
hostname R2-SPOKE1
!
interface Ethernet0/0
 ip address 2.2.2.2 255.255.255.252
 no shutdown
!
interface Ethernet0/1
 ip address 7.84.1.1 255.255.255.0
 no shutdown
!
crypto isakmp policy 10
 encr aes
 hash sha
 authentication pre-share
 group 2
 lifetime 86400
!
crypto isakmp key cisco123 address 1.1.1.1
!
crypto ipsec transform-set TS esp-aes esp-sha-hmac
 mode transport
!
crypto ipsec profile VPN_PROFILE
 set transform-set TS
!
interface Tunnel0
 ip address 10.0.0.2 255.255.255.0
 ip nhrp map 10.0.0.1 1.1.1.1
 ip nhrp map multicast 1.1.1.1
 ip nhrp nhs 10.0.0.1
 ip nhrp network-id 1
 tunnel source Ethernet0/0
 tunnel mode gre multipoint
 tunnel protection ipsec profile VPN_PROFILE
!
router eigrp 100
 network 10.0.0.0 0.0.0.255
 network 7.84.1.0 0.0.0.255
 no auto-summary
```

### 🟣 Configuración SPOKE2 (R3)

```
hostname R3-SPOKE2
!
interface Ethernet0/0
 ip address 3.3.3.2 255.255.255.252
 no shutdown
!
interface Ethernet0/1
 ip address 7.84.2.1 255.255.255.0
 no shutdown
!
crypto isakmp policy 10
 encr aes
 hash sha
 authentication pre-share
 group 2
 lifetime 86400
!
crypto isakmp key cisco123 address 1.1.1.1
!
crypto ipsec transform-set TS esp-aes esp-sha-hmac
 mode transport
!
crypto ipsec profile VPN_PROFILE
 set transform-set TS
!
interface Tunnel0
 ip address 10.0.0.3 255.255.255.0
 ip nhrp map 10.0.0.1 1.1.1.1
 ip nhrp map multicast 1.1.1.1
 ip nhrp nhs 10.0.0.1
 ip nhrp network-id 1
 tunnel source Ethernet0/0
 tunnel mode gre multipoint
 tunnel protection ipsec profile VPN_PROFILE
!
router eigrp 100
 network 10.0.0.0 0.0.0.255
 network 7.84.2.0 0.0.0.255
 no auto-summary
```

---

## 4. Verificación (Comandos)

```
show ip int brief
show dmvpn
show ip nhrp
show crypto isakmp sa
show crypto ipsec sa
show ip route eigrp
show ip eigrp neighbors
```
