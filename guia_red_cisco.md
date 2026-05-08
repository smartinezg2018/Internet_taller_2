# Guía de Configuración — Red Empresarial Cisco

> **Topología:** VLANs → OSPF → BGP → NAT → DHCP → DNS → SSH → Correo → WiFi

---

## Tabla de Direccionamiento

| Subred | Hosts | Red | Prefijo / Máscara | Primera IP | Última IP | Broadcast | Gateway |
|---|---|---|---|---|---|---|---|
| VLAN60 (MARKETING) | 730 | 10.10.0.0 | /22 · 255.255.252.0 | 10.10.0.1 | 10.10.3.254 | 10.10.3.255 | 10.10.0.1 |
| VLAN50 (PRODUCTION) | 350 | 10.10.4.0 | /23 · 255.255.254.0 | 10.10.4.1 | 10.10.5.254 | 10.10.5.255 | 10.10.4.1 |
| LAN3 (Server2) | 80 | 172.20.0.0 | /25 · 255.255.255.128 | 172.20.0.1 | 172.20.0.126 | 172.20.0.127 | 172.20.0.1 |
| LAN4 (TOKYO→WiFi) | 38 | 172.20.0.128 | /26 · 255.255.255.192 | 172.20.0.129 | 172.20.0.190 | 172.20.0.191 | 172.20.0.129 |
| RED SERVER1 (DNS) | 25 | 172.20.0.192 | /27 · 255.255.255.224 | 172.20.0.193 | 172.20.0.222 | 172.20.0.223 | 172.20.0.193 |
| WAN CENTRAL↔LONDON | 4 | 199.0.0.0 | /29 · 255.255.255.248 | 199.0.0.1 | 199.0.0.6 | 199.0.0.7 | CENTRAL: 199.0.0.1 · LONDON: 199.0.0.2 |
| WAN LONDON↔NEW_YORK | 2 | 199.0.0.8 | /30 · 255.255.255.252 | 199.0.0.9 | 199.0.0.10 | 199.0.0.11 | LONDON: 199.0.0.9 · NEW_YORK: 199.0.0.10 |
| WAN NEW_YORK↔TOKYO | 2 | 199.0.0.12 | /30 · 255.255.255.252 | 199.0.0.13 | 199.0.0.14 | 199.0.0.15 | NEW_YORK: 199.0.0.13 · TOKYO: 199.0.0.14 |
| Red inalámbrica | 100 | 10.1.1.0 | /25 · 255.255.255.128 | 10.1.1.1 | 10.1.1.126 | 10.1.1.127 | 10.1.1.1 |
| Conexión CLOUD | 3 | 200.0.0.0 | /29 · 255.255.255.248 | 200.0.0.1 | 200.0.0.6 | 200.0.0.7 | 200.0.0.1 |
| VLAN100 (MANAGEMENT) | — | 192.168.100.0 | /29 · 255.255.255.248 | 192.168.100.1 | 192.168.100.6 | 192.168.100.7 | 192.168.100.1 |

---

## PARTE 1 — Switch de Acceso (SW_VLAN)

El switch 2960 conecta los equipos de usuario. Los puertos 1-10 pertenecen a VLAN 50 (Producción), los puertos 11-19 a VLAN 60 (Marketing) y el puerto 24 es el enlace troncal hacia CENTRAL.

```
enable
configure terminal

! Crear VLANs
vlan 50
 name PRODUCTION
 exit

vlan 60
 name MARKETING
 exit

vlan 100
 name MANAGEMENT
 exit

! Puertos de acceso VLAN 50 (PC0 y producción — Fa0/1 a Fa0/10)
interface range fastEthernet 0/1 - 10
 switchport mode access
 switchport access vlan 50
 exit

! Puertos de acceso VLAN 60 (PC1 y marketing — Fa0/11 a Fa0/19)
interface range fastEthernet 0/11 - 19
 switchport mode access
 switchport access vlan 60
 exit

! Enlace troncal hacia CENTRAL (Fa0/24)
interface fastEthernet 0/24
 switchport mode trunk
 switchport trunk native vlan 100
 switchport trunk allowed vlan 50,60,100
 exit

end
write memory
```

**Verificación:**

```
show vlan brief
show interfaces trunk
show running-config
```

---

## PARTE 2 — Switch Multilayer CENTRAL

CENTRAL actúa como router de capa 3. Crea una SVI por cada VLAN para ser el gateway de cada subred, y un puerto routed hacia LONDON.

```
enable
configure terminal

! Habilitar enrutamiento IP en el multilayer switch
ip routing

! SVI VLAN 60 — MARKETING (gateway para 730 hosts)
interface vlan 60
 ip address 10.10.0.1 255.255.252.0
 no shutdown
 exit

! SVI VLAN 50 — PRODUCTION (gateway para 350 hosts)
interface vlan 50
 ip address 10.10.4.1 255.255.254.0
 no shutdown
 exit

! SVI VLAN 100 — MANAGEMENT (VLAN nativa)
interface vlan 100
 ip address 192.168.100.1 255.255.255.248
 no shutdown
 exit

! Puerto routed hacia LONDON (Gig1/1/4) — eliminar modo switchport
interface GigabitEthernet1/1/4
 no switchport
 ip address 199.0.0.1 255.255.255.248
 no shutdown
 exit

! Puerto troncal hacia SW_VLAN (Gig1/0/1)
interface GigabitEthernet1/0/1
 switchport mode trunk
 switchport trunk native vlan 100
 switchport trunk allowed vlan 50,60,100
 no shutdown
 exit

end
write memory
```

---

## PARTE 3 — OSPF en CENTRAL

OSPF anuncia las redes internas hacia LONDON y redistribuye la conectividad de las VLANs.

```
enable
configure terminal

router ospf 1
 ! Red WAN hacia LONDON (/29)
 network 199.0.0.0 0.0.0.7 area 0
 ! SVIs — usar wildcard exacto (host específico) para mayor precisión
 network 10.10.0.1 0.0.0.0 area 0
 network 10.10.4.1 0.0.0.0 area 0
 network 192.168.100.1 0.0.0.0 area 0
 exit

! Relay DHCP: reenviar solicitudes de VLANs hacia LONDON (199.0.0.2)
interface vlan 50
 ip helper-address 199.0.0.2
 exit

interface vlan 60
 ip helper-address 199.0.0.2
 exit

end
write memory
```

---

## PARTE 4 — Router LONDON (AS 200)

LONDON es el punto de salida BGP de la red interna. También actúa como servidor DHCP para las VLANs y aloja o conecta con Server1 (DNS/Correo).

### 4a. Interfaces

```
enable
configure terminal

! Interfaz hacia CENTRAL (/29 — primera IP usable)
interface GigabitEthernet0/0/0
 ip address 199.0.0.2 255.255.255.248
 no shutdown
 exit

! Interfaz hacia NEW_YORK (enlace BGP /30)
interface GigabitEthernet0/1/0
 ip address 199.0.0.9 255.255.255.252
 no shutdown
 exit

! Interfaz hacia Server1 (Red DNS 172.20.0.192/27)
interface GigabitEthernet0/0
 ip address 172.20.0.193 255.255.255.224
 no shutdown
 exit

end
write memory
```

### 4b. OSPF en LONDON

```
enable
configure terminal

router ospf 1
 ! Red WAN hacia CENTRAL
 network 199.0.0.0 0.0.0.7 area 0
 ! Red WAN hacia NEW_YORK
 network 199.0.0.8 0.0.0.3 area 0
 ! Red de Server1 (DNS)
 network 172.20.0.192 0.0.0.31 area 0
 exit

end
write memory
```

### 4c. BGP en LONDON

```
enable
configure terminal

router bgp 200
 ! Vecino BGP: NEW_YORK (AS 300)
 neighbor 199.0.0.10 remote-as 300
 ! Anunciar red WAN LONDON↔NEW_YORK
 network 199.0.0.8 mask 255.255.255.252
 exit

end
write memory
```

### 4d. Rutas estáticas hacia las VLANs internas (para DHCP)

LONDON necesita saber cómo llegar a las VLANs para entregar IPs vía DHCP relay.

```
enable
configure terminal

ip route 10.10.4.0 255.255.254.0 199.0.0.1
ip route 10.10.0.0 255.255.252.0 199.0.0.1
ip route 192.168.100.0 255.255.255.248 199.0.0.1

end
write memory
```

### 4e. Servidor DHCP en LONDON

```
enable
configure terminal

! Excluir IPs reservadas (gateways, servidores, administración)
ip dhcp excluded-address 10.10.4.1 10.10.4.20
ip dhcp excluded-address 10.10.0.1 10.10.0.20

! Pool para VLAN 50 — PRODUCTION
ip dhcp pool VLAN50
 network 10.10.4.0 255.255.254.0
 default-router 10.10.4.1
 dns-server 172.20.0.194
 exit

! Pool para VLAN 60 — MARKETING
ip dhcp pool VLAN60
 network 10.10.0.0 255.255.252.0
 default-router 10.10.0.1
 dns-server 172.20.0.194
 exit

end
write memory
```

**Verificación:**

```
show ip dhcp pool
show ip dhcp binding
```

---

## PARTE 5 — Router NEW_YORK (AS 300)

NEW_YORK es el hub BGP entre LONDON (AS 200) y TOKYO (AS 400), y redistribuye entre BGP y OSPF.

### 5a. Interfaces

```
enable
configure terminal

! Interfaz hacia LONDON
interface GigabitEthernet0/0/0
 ip address 199.0.0.10 255.255.255.252
 no shutdown
 exit

! Interfaz hacia TOKYO
interface GigabitEthernet0/1/0
 ip address 199.0.0.13 255.255.255.252
 no shutdown
 exit

end
write memory
```

### 5b. BGP en NEW_YORK

```
enable
configure terminal

router bgp 300
 ! Vecinos BGP
 neighbor 199.0.0.9 remote-as 200
 neighbor 199.0.0.14 remote-as 400
 ! Anunciar ambos enlaces WAN
 network 199.0.0.8 mask 255.255.255.252
 network 199.0.0.12 mask 255.255.255.252
 ! Redistribuir rutas OSPF hacia BGP
 redistribute ospf 1
 exit

end
write memory
```

### 5c. OSPF en NEW_YORK + redistribución BGP

```
enable
configure terminal

router ospf 1
 network 199.0.0.8 0.0.0.3 area 0
 network 199.0.0.12 0.0.0.3 area 0
 ! Redistribuir rutas BGP hacia OSPF
 redistribute bgp 300 subnets
 exit

end
write memory
```

### 5d. SSH en NEW_YORK

```
enable
configure terminal

ip domain-name technology.com
username admin secret cisco123

! Generar clave RSA (ingresar 1024 cuando se solicite el tamaño)
crypto key generate rsa
! > 1024

ip ssh version 2

line vty 0 4
 transport input ssh
 login local
 exit

end
write memory
```

**Verificación:**

```
show ip ssh
```

**Conectar desde PC0:**
1. `Desktop → SSH`
2. IP Address: `199.0.0.10`
3. Username: `admin` / Password: `cisco123`
4. Click **Connect**

---

## PARTE 6 — Router TOKYO (AS 400)

TOKYO conecta NEW_YORK con la CLOUD y gestiona las redes LAN4 y WiFi. Implementa NAT con PAT.

### 6a. Interfaces

```
enable
configure terminal

! Interfaz hacia NEW_YORK
interface GigabitEthernet0/0/0
 ip address 199.0.0.14 255.255.255.252
 no shutdown
 exit

! Interfaz hacia CLOUD (salida a Internet)
interface GigabitEthernet0/1/0
 ip address 200.0.0.1 255.255.255.248
 no shutdown
 exit

! Interfaz hacia LAN4 (conecta Wireless Router)
interface GigabitEthernet0/0
 ip address 172.20.0.129 255.255.255.192
 no shutdown
 exit

end
write memory
```

### 6b. BGP en TOKYO

```
enable
configure terminal

router bgp 400
 ! Vecino BGP: NEW_YORK (AS 300)
 neighbor 199.0.0.13 remote-as 300
 ! Anunciar enlace WAN con NEW_YORK
 network 199.0.0.12 mask 255.255.255.252
 ! Anunciar red hacia CLOUD
 network 200.0.0.0 mask 255.255.255.248
 ! Anunciar ruta por defecto (para acceso a Internet)
 network 0.0.0.0 mask 0.0.0.0
 ! Redistribuir rutas estáticas (ruta por defecto)
 redistribute static
 exit

end
write memory
```

### 6c. OSPF en TOKYO

```
enable
configure terminal

router ospf 1
 network 172.20.0.128 0.0.0.63 area 0
 exit

end
write memory
```

### 6d. Ruta estática hacia la red WiFi

```
enable
configure terminal

! La red WiFi (10.1.1.0/25) es alcanzable a través del Wireless Router (172.20.0.133)
ip route 10.1.1.0 255.255.255.128 172.20.0.133

end
write memory
```

### 6e. NAT con Pool y PAT (sobrecarga) en TOKYO

```
enable
configure terminal

! ACL: identificar tráfico interno de VLANs a natearse
access-list 1 permit 10.10.0.0 0.0.3.255
access-list 1 permit 10.10.4.0 0.0.1.255

! Pool de IPs públicas (200.0.0.3 a 200.0.0.6 de la red 200.0.0.0/29)
ip nat pool NATPOOL 200.0.0.3 200.0.0.6 netmask 255.255.255.248

! Activar PAT: asociar ACL con pool + overload
ip nat inside source list 1 pool NATPOOL overload

! Marcar interfaz interna (hacia NEW_YORK / red interna)
interface GigabitEthernet0/0/0
 ip nat inside
 exit

! Marcar interfaz externa (hacia CLOUD / Internet)
interface GigabitEthernet0/1/0
 ip nat outside
 exit

end
write memory
```

**Verificación:**

```
show ip nat translations
show ip nat statistics
```

---

## PARTE 7 — CLOUD

Dispositivo final que simula Internet. Solo requiere configuración de interfaz.

```
enable
configure terminal

interface GigabitEthernet0/0/0
 ip address 200.0.0.2 255.255.255.248
 no shutdown
 exit

end
write memory
```

---

## PARTE 8 — LAN3 y Server2

LAN3 utiliza subinterfaces en su router para conectar dos redes (VLAN 50 con Server2 y LAN de Marketing).

### 8a. Router de LAN3 — subinterfaces

```
enable
configure terminal

! Limpiar IP de la interfaz física (sin IP en el puerto físico)
interface GigabitEthernet0/0
 no ip address
 no shutdown
 exit

! Subinterfaz para la red de Server2 (LAN3 — 172.20.0.0/25)
interface GigabitEthernet0/0.50
 encapsulation dot1Q 50
 ip address 172.20.0.1 255.255.255.128
 exit

! Subinterfaz para VLAN 60 — MARKETING (si aplica en este segmento)
interface GigabitEthernet0/0.60
 encapsulation dot1Q 60
 ip address 10.10.0.1 255.255.252.0
 exit

! OSPF para anunciar la red de Server2
router ospf 1
 network 172.20.0.0 0.0.0.127 area 0
 exit

end
write memory
```

### 8b. Configuración de Server2

| Campo | Valor |
|---|---|
| IP | 172.20.0.2 |
| Máscara | 255.255.255.128 |
| Gateway | 172.20.0.1 |

---

## PARTE 9 — Server1 (DNS y Correo)

Server1 se ubica en la red `172.20.0.192/27` conectada a LONDON.

### 9a. Configuración IP de Server1

| Campo | Valor |
|---|---|
| IP | 172.20.0.194 |
| Máscara | 255.255.255.224 |
| Gateway | 172.20.0.193 |

### 9b. Configurar DNS en Server1

1. Click en **Server1**
2. Pestaña **Services**
3. Click en **DNS**
4. DNS Service: **ON**
5. Agregar registro:
   - Name: `technology.com`
   - Type: `A Record`
   - Address: `172.20.0.194`
6. Click **Add** → **Save**

### 9c. Configurar DNS en los equipos de usuario

**PC0 y PC1:**
- `Desktop → IP Configuration`
- DNS Server: `172.20.0.194`

### 9d. Configurar servicio de correo (SMTP/POP3) en Server1

1. Click en **Server1** → Pestaña **Services** → **EMAIL**
2. Activar **SMTP: ON** y **POP3: ON**
3. Domain Name: `technology.com` → Click **Set**
4. Crear usuarios:
   - Username: `pc0` / Password: `cisco123` → Click **+**
   - Username: `laptop` / Password: `cisco123` → Click **+**
5. Click **Save**

### 9e. Configurar cliente de correo en PC0

1. `Desktop → Email`
2. Rellenar:
   - Your Name: `PC0`
   - Email Address: `pc0@technology.com`
   - Incoming Mail Server: `172.20.0.194`
   - Outgoing Mail Server: `172.20.0.194`
   - Username: `pc0` / Password: `cisco123`
3. Click **Save**

### 9f. Configurar cliente de correo en Laptop0

1. `Desktop → Email`
2. Rellenar:
   - Your Name: `Laptop`
   - Email Address: `laptop@technology.com`
   - Incoming Mail Server: `172.20.0.194`
   - Outgoing Mail Server: `172.20.0.194`
   - Username: `laptop` / Password: `cisco123`
3. Click **Save**

### 9g. Prueba de correo

**Enviar desde PC0:**
1. `Desktop → Email → Compose`
2. To: `laptop@technology.com`
3. Subject: `Prueba`
4. Body: `Hola desde PC0`
5. Click **Send**

**Recibir en Laptop0:**
1. `Desktop → Email → Receive`
2. Debe aparecer el correo de PC0.

---

## PARTE 10 — Red Inalámbrica (WiFi)

El Wireless Router conecta Laptop0 de forma inalámbrica a la red de TOKYO (LAN4).

### 10a. Configuración del Wireless Router

1. Click en **Wireless Router0** → Pestaña **GUI**

**Setup → Basic Setup:**

| Campo | Valor |
|---|---|
| Internet IP | 172.20.0.133 |
| Subnet Mask | 255.255.255.192 |
| Gateway | 172.20.0.129 |
| Router IP | 10.1.1.1 |
| Subnet Mask | 255.255.255.128 |
| DHCP Server | Enabled |
| Start IP | 10.1.1.2 |
| Max users | 100 |

Click **Save Settings**

### 10b. Cambiar SSID

1. **GUI → Wireless → Basic Wireless Settings**
2. Network Name (SSID): `TU_APELLIDO`
3. Click **Save Settings**

### 10c. Implementar seguridad WPA2

1. **GUI → Wireless → Wireless Security**
2. Security Mode: `WPA2 Personal`
3. Encryption: `AES`
4. Passphrase: `cisco123`
5. Click **Save Settings**

### 10d. Filtrado por MAC

1. **GUI → Wireless → Wireless MAC Filter**
2. Wireless MAC Filter: **Enable**
3. Modo: **Permit PCs listed below to access wireless network**
4. Obtener la MAC de Laptop0:
   - Click en **Laptop0** → **Config → Wireless0**
   - Copiar la MAC Address
5. Pegar la MAC en el filtro
6. Click **Save Settings**

### 10e. Conectar Laptop0

1. Click en **Laptop0** → **Config → Wireless0**
2. SSID: `TU_APELLIDO`
3. Authentication: `WPA2-PSK`
4. PSK Pass Phrase: `cisco123`

**Verificar:** Laptop0 debe obtener IP automáticamente en el rango `10.1.1.2 - 10.1.1.126`

---

## PARTE 11 — Configuración de PC0 y PC1

Los equipos de usuario obtienen IP por DHCP. Verificar que los campos sean correctos.

### PC0 — VLAN 50 (PRODUCTION)

| Campo | Valor |
|---|---|
| IP | (asignada por DHCP desde 10.10.4.21 en adelante) |
| Máscara | 255.255.254.0 |
| Gateway | 10.10.4.1 |
| DNS Server | 172.20.0.194 |

### PC1 — VLAN 60 (MARKETING)

| Campo | Valor |
|---|---|
| IP | (asignada por DHCP desde 10.10.0.21 en adelante) |
| Máscara | 255.255.252.0 |
| Gateway | 10.10.0.1 |
| DNS Server | 172.20.0.194 |

### PC2 — LAN4 (TOKYO)

| Campo | Valor |
|---|---|
| IP | 172.20.0.130 |
| Máscara | 255.255.255.192 |
| Gateway | 172.20.0.129 |

---

## PARTE 12 — Verificación General

### Verificar tabla de rutas en CENTRAL

```
CENTRAL# show ip route
```

Se deben ver rutas OSPF (`O`) hacia las redes WAN y rutas externas (`O E2`) redistribuidas desde BGP, por ejemplo:

```
O E2  10.10.0.0/21 via 199.0.0.2
```

### Verificar adyacencias OSPF

```
show ip ospf neighbor
```

### Verificar sesiones BGP

```
show ip bgp summary
show ip bgp
```

### Verificar VLANs y troncales

```
show vlan brief
show interfaces trunk
```

### Verificar NAT (en TOKYO)

```
show ip nat translations
show ip nat statistics
```

### Verificar SSH (en NEW_YORK)

```
show ip ssh
show users
```

---

## Resumen de IPs por dispositivo

| Dispositivo | Interfaz | IP | Máscara |
|---|---|---|---|
| CENTRAL | Vlan 50 | 10.10.4.1 | 255.255.254.0 |
| CENTRAL | Vlan 60 | 10.10.0.1 | 255.255.252.0 |
| CENTRAL | Vlan 100 | 192.168.100.1 | 255.255.255.248 |
| CENTRAL | Gig1/1/4 | 199.0.0.1 | 255.255.255.248 |
| LONDON | Gig0/0/0 | 199.0.0.2 | 255.255.255.248 |
| LONDON | Gig0/1/0 | 199.0.0.9 | 255.255.255.252 |
| LONDON | Gig0/0 | 172.20.0.193 | 255.255.255.224 |
| NEW_YORK | Gig0/0/0 | 199.0.0.10 | 255.255.255.252 |
| NEW_YORK | Gig0/1/0 | 199.0.0.13 | 255.255.255.252 |
| TOKYO | Gig0/0/0 | 199.0.0.14 | 255.255.255.252 |
| TOKYO | Gig0/1/0 | 200.0.0.1 | 255.255.255.248 |
| TOKYO | Gig0/0 | 172.20.0.129 | 255.255.255.192 |
| CLOUD | Gig0/0/0 | 200.0.0.2 | 255.255.255.248 |
| Server1 (DNS) | NIC | 172.20.0.194 | 255.255.255.224 |
| Server2 | NIC | 172.20.0.2 | 255.255.255.128 |
| Wireless Router | WAN | 172.20.0.133 | 255.255.255.192 |
| Wireless Router | LAN | 10.1.1.1 | 255.255.255.128 |
| PC2 | NIC | 172.20.0.130 | 255.255.255.192 |
