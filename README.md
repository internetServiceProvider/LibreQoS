# LibreQoS:

**LibreQoS** es un sistema **de gestión de calidad de servicio (QoS)** pensado especialmente para **proveedores de servicios de Internet (ISP)**, aunque puede aplicarse a cualquier red donde se necesite controlar y optimizar el uso del ancho de banda por usuario. Es un software libre y basado en linux el cual está diseñado para correr en un servidor cerca del router principal, entre las funciones que tiene se encuentra:

- Limitar el ancho de banda según el cliente y los planes que tenga contratados.
- También prioriza el tipo de tráfico que se va a ofrecer como servicio.
- Evita que la red se congestione, usando factores de agregación

Para proceder con la instalacion e implementacion del servicio LibreQoS se requieren de los siguientes elementos:

- Un servidor con **Linux** (Ubuntu 20.04 o superior recomendado).
- **LibreQoS** instalado (usa Docker, así que es más fácil).
- Una base de datos de **clientes y planes de velocidad** (por IP/MAC o VLAN ID).
- Que tu red permita el ruteo **o bridging** del tráfico a través del servidor con LibreQoS.

## Diseño de red:

**LibreQoS debe estar en medio** del tráfico entre el router de borde (el que conecta con internet, con NAT y firewall usualmente) y el router o switch central de tu red (donde se distribuye el tráfico hacia clientes), por lo que que **todo el tráfico** que entra/sale de tu red **debe pasar por LibreQoS** para que este pueda aplicar sus políticas de control.
![image](https://github.com/user-attachments/assets/f8f3da9b-351f-45a8-af30-b9d9d885cfe0)


## Bridge:

Una vez definidas las conexiones físicas. Se procede a hacer la configuración inicial del puente de red que se va a encargar de conectar las dos interfaces que se designaron, lo que viene a ser la interfaz que va hacia el core y la que va hacia el router. (Este bridge es recomendado para la mayoría de instalaciones, esto se debe a que este tipo de puente sigue pasando datos incluso si el servicio lqosd de LibreQoS falla.)

```bash
4: br0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000
    link/ether 6e:cd:8c:af:c8:9b brd ff:ff:ff:ff:ff:ff
    inet 192.168.50.10/27 brd 192.168.50.31 scope global br0
       valid_lft forever preferred_lft forever
    inet6 2001:db8:a:c:6ccd:8cff:feaf:c89b/64 scope global dynamic mngtmpaddr noprefixroute 
       valid_lft 2591725sec preferred_lft 604525sec
    inet6 fe80::6ccd:8cff:feaf:c89b/64 scope link 
       valid_lft forever preferred_lft forever
       
##Summary 
4: br0: <BROADCAST,MULTICAST,UP,LOWER_UP>
    inet 192.168.50.10/27
    interfaces: eno1, eno2

```

Este *bridge* es la interfaz principal de red del servidor (Faraday), usada para comunicarse con otros dispositivos en tu red de gestión o con tus routers/OLT. En donde el tráfico que entra por `eno1` o `eno2` es manejado de forma conjunta como si fueran una sola red.

## Configuracion de la network:

El archivo `network.json` en LibreQoS se encarga de modelar la topología jerárquica de la red física del ISP, permitiendo al sistema comprender cómo se distribuye el ancho de banda desde el nodo principal (por ejemplo, un MikroTik o router principal) hacia los distintos puntos de acceso donde se conectan los usuarios finales, como OLTs, ONTs o Access Points. Establece los límites máximos de ancho de banda disponibles en cada nivel del árbol,  para asi calcular cómo distribuir los recursos de manera justa y evitar la sobreasignación en puntos congestionados.

```json
"OLT_Huawei": {
  "downloadBandwidthMbps": 500,
  "uploadBandwidthMbps": 500,
  "type": "Site",
  "children": {
    "ONT_Huawei": {
      "downloadBandwidthMbps": 300,
      "uploadBandwidthMbps": 300,
      "type": "AP"
    },
    "TVWS_Master": {
      "downloadBandwidthMbps": 200,
      "uploadBandwidthMbps": 200,
      "type": "AP"
    }
  }
}

```

Este bloque representa un sitio llamado `OLT_Huawei`, que es un equipo de acceso óptico que conecta a varios usuarios finales. El OLT tiene una capacidad máxima de 500 Mbps de bajada y 500 Mbps de subida. Dentro de él hay dos "puntos de acceso" (`ONT_Huawei` y `TVWS_Master`) que representan los equipos finales que distribuyen internet a los clientes (como ONTs o radios). `ONT_Huawei` puede servir hasta 300 Mbps, lo cual permitiría hasta 3 usuarios del plan de 100 Mbps o 1 de 300 Mbps. `TVWS_Master`, por su parte, puede manejar hasta 200 Mbps, permitiendo 2 clientes de 100 Mbps. LibreQoS usa esta jerarquía para asegurarse de que la suma del consumo de todos los usuarios conectados a un nodo nunca exceda lo que realmente puede ofrecer el equipo, evitando saturaciones o unfairness en la red.

## ShapedDevices

El archivo de **shaped devices** en LibreQoS es fundamental porque define los dispositivos o clientes finales que estarán sujetos a políticas de control de tráfico (shaping). Este archivo permite asignar velocidades máximas de descarga y carga para cada cliente, garantizando así una distribución equitativa y eficiente del ancho de banda disponible. Al especificar parámetros como el identificador del circuito, nombre del dispositivo, nodo de acceso (AP), y los límites de velocidad, LibreQoS puede aplicar reglas precisas que evitan la congestión.

```markdown
| Circuit ID | Circuit Name   | Device ID | Device Name   | Parent Node   | MAC | IPv4 | IPv6 | Download Min Mbps | Upload Min Mbps | Download Max Mbps | Upload Max Mbps |
|------------|----------------|-----------|---------------|----------------|-----|------|------|--------------------|------------------|--------------------|------------------|
| 3000       | ONT_Huawei     | 1         | ONT_Huawei     | OLT_Huawei     |     |      |      | 25                 | 5                | 300                | 50               |
| 3001       | TVWS_Master    | 2         | TVWS_Master    | OLT_Huawei     |     |      |      | 25                 | 5                | 200                | 30               |
| 3002       | ONT_Fiberhome  | 3         | ONT_Fiberhome  | OLT_Fiberhome  |     |      |      | 25                 | 5                | 250                | 40               |
| 3003       | Server_CJ      | 4         | Server_CJ      | Switch_Cisco   |     |      |      | 25                 | 5                | 100                | 20               |

```
## Libreqos.conf

```toml
toml
version = "1.5"
lqos_directory = "/opt/libreqos/src"
node_id = "6f74b9363618059aa512e459e3821059709b8cce0ee5c2152f4433496a5ca4ce"
node_name = "LibreQoS"

```

- **version**: Define la versión del formato de configuración
- **lqos_directory**: Ruta donde están los archivos fuente de LibreQoS
- **node_id**: Identificador único del nodo (hash SHA-256)
- **node_name**: Nombre descriptivo del nodo

## Sección [tuning] - Optimización de Red

Esta es una de las secciones más críticas:

```toml

toml
stop_irq_balance = true
netdev_budget_usecs = 8000
netdev_budget_packets = 300
disable_offload = ["gso", "tso", "lro", "sg", "gro"]

```

- **stop_irq_balance**: Detiene el balanceador automático de interrupciones para control manual
- **netdev_budget**: Controla cuánto tiempo/paquetes puede procesar el kernel por ciclo
- **disable_offload**: Desactiva optimizaciones de hardware que pueden interferir con QoS:
    - GSO/TSO: Segmentación de paquetes grandes
    - LRO/GRO: Agregación de paquetes recibidos
    - SG: Scatter-gather (fragmentación)

## Sección [bridge] - Configuración de Interfaces

```toml

toml
use_xdp_bridge = false
to_internet = "eno2"
to_network = "eno1"

```

- **to_internet**: Interfaz hacia el proveedor de internet
- **to_network**: Interfaz hacia la red local
- **use_xdp_bridge**: XDP (eXpress Data Path) para procesamiento de paquetes de alta velocidad

## Sección [queues] - Control de Ancho de Banda

```toml

toml
default_sqm = "cake diffserv4"
uplink_bandwidth_mbps = 1250
downlink_bandwidth_mbps = 2500

```

- **default_sqm**: Algoritmo de gestión de colas (CAKE con DiffServ)
- **uplink/downlink_bandwidth**: Límites de ancho de banda en Mbps
- **monitor_only**: Si está en true, solo monitorea sin aplicar límites

## Sección [ip_ranges] - Control de Acceso

```toml

toml
allow_subnets = [
    "172.16.0.0/12",
    "10.0.0.0/8",
    "100.64.0.0/10",
    "192.168.0.0/16"
]

```

Define las redes privadas RFC 1918 y CGNAT (100.64.0.0/10) que el sistema gestionará.
