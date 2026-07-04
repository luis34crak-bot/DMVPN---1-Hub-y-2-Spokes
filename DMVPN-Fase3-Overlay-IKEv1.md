# 🔐 DMVPN Fase 3 (Overlay) — IKEv1

## 1. Introducción

La **DMVPN Fase 3** es una mejora de la Fase 2. También usa **Hub and Spoke**, pero agrega una función llamada **redirección de rutas**.

Cuando un Spoke quiere hablar con otro:

1. Primero envía el tráfico al Hub
2. El Hub le indica la mejor ruta
3. Luego se crea una conexión directa entre los Spokes

> Esto hace que la red sea más eficiente y escalable.

### 📌 Para qué se usa

- Redes con muchas sucursales
- Optimizar el tráfico entre sedes
- Facilitar el uso de enrutamiento dinámico

---

## 2. Topología

```
                        [ISP]
        e0/0            e0/1            e0/2
     1.1.1.2/30      2.2.2.1/30      3.3.3.1/30
        │                 │                │
     e0/0 (WAN)       e0/0 (WAN)       e0/0 (WAN)
   1.1.1.1/30        2.2.2.2/30       3.3.3.3/30
   [R1 - HUB]      [R2 - SPOKE1]    [R3 - SPOKE2]
   e0/1 (LAN)        e0/1 (LAN)       e0/1 (LAN)
   7.84.0.1/24       7.84.1.1/24      7.84.2.1/24
      │                 │                │
    LAN HUB          LAN SPOKE1       LAN SPOKE2
  7.84.0.0/24       7.84.1.0/24      7.84.2.0/24

           Tunnel0 (DMVPN) — Red 10.0.0.0/24
   HUB: 10.0.0.1   SPOKE1: 10.0.0.2   SPOKE2: 10.0.0.3
```

### Router ISP

> Este router permite la comunicación entre las IPs públicas.

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

---

## 3. Configuración DMVPN Fase 3

### 🟢 HUB (R1 - IOU2)

> En la Fase 3, el comando `ip nhrp redirect` es el que permite que el Hub le avise a los Spokes que existe un camino más corto.

```
hostname R1
!
crypto isakmp policy 10
 encryption aes
 hash sha
 authentication pre-share
 group 2
crypto isakmp key cisco123 address 0.0.0.0
!
crypto ipsec transform-set TS esp-aes esp-sha-hmac
 mode transport
!
crypto ipsec profile VPN_PROFILE
 set transform-set TS
!
interface Ethernet0/0
 ip address 1.1.1.1 255.255.255.252
 no shutdown
ip route 0.0.0.0 0.0.0.0 1.1.1.2
!
interface Tunnel0
 ip address 10.0.0.1 255.255.255.0
 no ip redirects
 ip nhrp map multicast dynamic
 ip nhrp network-id 1
 ip nhrp redirect
 tunnel source Ethernet0/0
 tunnel mode gre multipoint
 tunnel protection ipsec profile VPN_PROFILE
!
router eigrp 100
 network 10.0.0.0 0.0.0.255
 network 7.84.0.0 0.0.0.255
 no auto-summary
 interface Tunnel0
  no ip split-horizon eigrp 100
  # En Fase 3, puedes dejar el 'ip next-hop-self' por defecto.
```

### 🔵 SPOKE 1 (R2 - IOU3)

> El comando `ip nhrp shortcut` permite que el Spoke aprenda y use el túnel directo.

```
hostname R2
!
# (Misma configuración de Crypto que el Hub)
!
interface Ethernet0/0
 ip address 2.2.2.2 255.255.255.252
 no shutdown
ip route 0.0.0.0 0.0.0.0 2.2.2.1
!
interface Tunnel0
 ip address 10.0.0.2 255.255.255.0
 ip nhrp map 10.0.0.1 1.1.1.1
 ip nhrp map multicast 1.1.1.1
 ip nhrp nhs 10.0.0.1
 ip nhrp network-id 1
 ip nhrp shortcut
 tunnel source Ethernet0/0
 tunnel mode gre multipoint
 tunnel protection ipsec profile VPN_PROFILE
!
router eigrp 100
 network 10.0.0.0 0.0.0.255
 network 7.84.1.0 0.0.0.255
```

### 🟣 SPOKE 2 (R3 - IOU4)

```
hostname R3
!
# (Misma configuración de Crypto que el Hub)
!
interface Ethernet0/0
 ip address 3.3.3.3 255.255.255.252
 no shutdown
ip route 0.0.0.0 0.0.0.0 3.3.3.1
!
interface Tunnel0
 ip address 10.0.0.3 255.255.255.0
 ip nhrp map 10.0.0.1 1.1.1.1
 ip nhrp map multicast 1.1.1.1
 ip nhrp nhs 10.0.0.1
 ip nhrp network-id 1
 ip nhrp shortcut
 tunnel source Ethernet0/0
 tunnel mode gre multipoint
 tunnel protection ipsec profile VPN_PROFILE
!
router eigrp 100
 network 10.0.0.0 0.0.0.255
 network 7.84.2.0 0.0.0.255
```
