#  Proxmox_LAB (Gunter, David, Gerard)

##  Descripción general

Este proyecto consiste en la creación de un **laboratorio de redes en Proxmox VE**
utilizando **contenedores LXC con Alpine Linux**.  
Se implementa una **red interna aislada** donde un **cliente** accede a Internet
a través de un **router LXC**, usando **NAT y Port Forwarding con iptables**.

El laboratorio permite comprender de forma práctica:
- Redes internas en Proxmox
- Routing / IP Tables
- NAT / Red Interna
- Port Forwarding
- Contenedores LXC unprivileged

---

##  Arquitectura del laboratorio

- **Host Proxmox VE**
- **vmbr0** → red física / salida a Internet
- **vmbr1** → red interna
- **Router LXC (privileged)**
- **Cliente LXC (unprivileged)**

---

## 🌐 Topología de red
```
Internet
|
Router físico
|
vmbr0 (Proxmox)
|
Router LXC
├─ eth0 → WAN (vmbr0)
└─ eth1 → LAN (vmbr1)
|
vmbr1
|
Cliente LXC
└─ eth0 → LAN (vmbr1)
```

---

## 📡 Direccionamiento IP

| Equipo       | Interfaz | IP              | Método |
|-------------|----------|-----------------|--------|
| Router LXC  | eth0     | 192.168.109.100 | Estática |
| Router LXC  | eth1     | 10.10.10.1      | Estática |
| Cliente LXC | eth0     | 10.10.10.2      | Estática |

> Todas las direcciones IP se configuran **manualmente**.  
> **No se utiliza DHCP**.

---

## ⚙️ Configuración del Router LXC

### Tipo de contenedor
- **El contenedor LXC debe crearse como privileged desde el inicio para permitir el uso de iptables y funciones de routing.** (solo se configura en router y el cliente se deja)
### Configuracion LXCRouter
  <img width="740" height="440" alt="image" src="https://github.com/user-attachments/assets/bfa73e78-2fdc-48ef-a44c-771bf76a3819" />

  
### Configuracion LXCClient
  <img width="726" height="438" alt="image" src="https://github.com/user-attachments/assets/1118181d-1834-4651-8a76-c2734bd3e8dc" />

---

### 🔁 Activación del reenvío IP

```
sysctl -w net.ipv4.ip_forward=1
```
Configuración aplicada de forma permanente para que los cambios no se pierdan al reiniciar:
```
echo "net.ipv4.ip_forward=1" >> /etc/sysctl.conf
```
---
## 🔥 Configuración NAT (Salida a Internet)
```
iptables -t nat -A POSTROUTING -s 10.10.10.0/24 -o eth0 -j MASQUERADE
iptables -A FORWARD -i eth1 -o eth0 -j ACCEPT
iptables -A FORWARD -i eth0 -o eth1 -m state --state RELATED,ESTABLISHED -j ACCEPT
```
Esto permite que el cliente interno acceda a Internet a través del router.

---
## 🔀 Port Forwarding
Se implementa **port forwarding** para permitir el acceso desde la red externa
a un servicio alojado en la red interna.

---
## 📌 Escenario de ejemplo
- Router (WAN): 192.168.109.100

- Cliente interno: 10.10.10.2

- Servicio interno: HTTP (puerto 80)

- Puerto expuesto: 8080

---
## Configuración del Router

### Regla DNAT (Redirección de puerto)
```
iptables -t nat -A PREROUTING -i eth0 -p tcp --dport 8080 \
-j DNAT --to-destination 10.10.10.2:80
```
### Permitir tráfico reenviado
```
iptables -A FORWARD -p tcp -d 10.10.10.2 --dport 80 \
-m state --state NEW,ESTABLISHED,RELATED -j ACCEPT

iptables -A FORWARD -p tcp -s 10.10.10.2 --sport 80 \
-m state --state ESTABLISHED,RELATED -j ACCEPT
```
---
##  Configuración del Cliente LXC
Archivo /etc/network/interfaces:

```
auto lo
iface lo inet loopback

auto eth0
iface eth0 inet static
    address 10.10.10.2/24
    gateway 10.10.10.1
```

##  Pruebas de conectividad
Desde el cliente:
```
ping 10.10.10.1
ping 192.168.109.1
ping 8.8.8.8
```
Resultado:

Se verifico una comunicación correcta entre el cliente y el router de la red interna, permitiendo el acceso a la red externa y la salida a Internet mediante NAT.

---
## Objetivos alcanzados
- Creación de red interna en Proxmox VE

- Uso de bridges Linux (vmbr0, vmbr1)

- Router LXC funcional

- NAT con iptables

- Port forwarding operativo

- Cliente interno con acceso a Internet

## Conclusión
Este laboratorio demuestra cómo Proxmox VE puede utilizarse para simular
infraestructuras de red reales, permitiendo aprender y practicar conceptos
fundamentales de administración de sistemas y redes de forma segura y controlada.
