Quiero que me prepares un proyecto de Ansible para convertir un Debian 13 (trixie) limpio en un Proxmox VE 9 y meterle un contenedor LXC con WireGuard. Lo llamo "el pepino" (o "el Proxmox bebé"). Escribe todo en español: comentarios, nombres de tareas y README.



\## Contexto

\- Lanzaré los playbooks desde mi LXC `ansible` (10.0.0.230), que vive en mi Proxmox del Kimsufi. Usa ansible-core de Debian 13 y el mínimo de colecciones posible (ansible.builtin primero). Si necesitas alguna, ponla en `requirements.yml`.

\- Destino real: un bare metal de Vultr con la imagen oficial de Debian 13. Trae cloud-init y los dos NVMe en RAID 1 por mdadm. El almacenamiento NO se toca: nada de particionar, romper el RAID ni montar ZFS.

\- Destino de pruebas: una VM del Kimsufi con la imagen `genericcloud` de Debian 13 (qcow2, CPU tipo host, snapshot `limpio`). Probaré ahí, haciendo `qm rollback` entre intentos.

\- Ansible entra por SSH solo con clave. En Vultr como root; en la genericcloud como el usuario `debian` con sudo. Que `ansible\_user` y `become` salgan del inventario.

\- No uso IPv6. No lo configures.



\## Estructura

pepino/

\- inventario.ini con dos grupos: `pepino` (Vultr) y `pepino\_prueba` (VM del Kimsufi), y group\_vars para cada uno

\- pepino.yml, que llama a los dos roles, cada uno con su tag

\- roles/changachanga/ (módulo 1: Proxmox)

\- roles/wg\_lxc/ (módulo 2: el contenedorcito WireGuard)

\- README.md



Tiene que poder lanzarse cada rol por separado (`--tags wg\_lxc`). Todo idempotente: si lo lanzo dos veces, la segunda no debe cambiar nada ni reiniciar.



\## Rol 1: changachanga (Proxmox sobre Debian)

1\. cloud-init quieto: que no gestione la red (`network: {config: disabled}` en /etc/cloud/cloud.cfg.d/), ni /etc/hosts ni el hostname. Borra la config de red que deja cloud-init (p. ej. /etc/network/interfaces.d/50-cloud-init). Anoche, en un reinicio, cloud-init me pisó el vmbr1.

2\. Hostname desde una variable, y que /etc/hosts lo resuelva a la IP pública real, no a 127.0.1.1.

3\. Repo `pve-no-subscription` de trixie en formato deb822 (.sources) con su keyring. Comprueba la URL y el nombre de la clave en la wiki oficial ("Install Proxmox VE on Debian 13 Trixie"), que cambian. Quita o desactiva los repos enterprise de pve y ceph que añade el paquete.

4\. Primera ronda: `apt full-upgrade`, instalar `proxmox-default-kernel` y reiniciar con el módulo `reboot`, con timeouts generosos (un bare metal tarda en volver).

5\. Segunda ronda: preseed de postfix con debconf ("Local only"), instalar `proxmox-ve`, `open-iscsi` y `chrony`, quitar `linux-image-amd64` y los linux-image-6.x de Debian y `os-prober`, `update-grub`.

6\. Red: plantilla `interfaces.j2` con:

&#x20;  - `vmbr0` puenteando la interfaz pública, con IP, máscara y puerta de enlace sacadas de los facts. Añade un `assert` que pare si no se detectan bien.

&#x20;  - `vmbr1` interno sin puerto físico, con masquerade de salida en `post-up`/`post-down`.

&#x20;  La subred de vmbr1 va en una variable y no puede pisar las mías: 10.0.0.0/24 (LAN del Kimsufi) y 10.1.0.0/24 (malla WireGuard). Pon un valor de ejemplo y avísame en el README.

&#x20;  La red nueva se aplica con el segundo reinicio, no con ifreload en caliente.

7\. Reinicio final y comprobaciones:

&#x20;  - el kernel en marcha es de Proxmox

&#x20;  - no hay ningún `linux-image-amd64` instalado

&#x20;  - `pveversion` responde

&#x20;  - existen vmbr0 y vmbr1

8\. Almacenamiento: dejar `local` (directorio en /var/lib/vz) con contenido images, rootdir, vztmpl, iso y backup. Nada más.

9\. Cargar el módulo `wireguard` en el host y dejarlo en /etc/modules-load.d (el kernel de Proxmox ya lo trae).



\## Rol 2: wg\_lxc (el contenedorcito WireGuard)

1\. Bajar con `pveam` la plantilla Debian 13 standard y crear el LXC con `pct` en el host. VMID, nombre, IP en vmbr1, RAM y disco van en variables. Arranque automático activado.

2\. Dentro del LXC:

&#x20;  - `apt install --no-install-recommends wireguard-tools`. NUNCA el paquete `wireguard` a secas: es un metapaquete que arrastra el kernel de Debian.

&#x20;  - `net.ipv4.ip\_forward=1` persistente.

&#x20;  - `/etc/wireguard/wg0.conf` desde la plantilla `wg0.conf.j2`, con permisos 600.

&#x20;  - `wg-quick@wg0` habilitado y arrancado.

3\. Claves fijas, no generadas en la instalación, para que el pepino tenga siempre la misma identidad y el hub lo reconozca:

&#x20;  - La privada llega en la variable `wg\_privada\_pepino`, cifrada con ansible-vault.

&#x20;  - Las públicas (la del pepino y las de los peers) van en variables normales.

&#x20;  - NO inventes claves. Deja marcadores claros y explica en el README cómo generarlas: `wg genkey | tee pepino.key | wg pubkey > pepino.pub`, luego `ansible-vault encrypt\_string` y `shred -u pepino.key`.

4\. Peers en una lista de variables:

&#x20;  - Como mínimo, el hub del Kimsufi (endpoint `vpn.felpeto.es`, con PersistentKeepalive), que enrutará hacia la subred del pepino.

&#x20;  - Opcional: mi portátil directo (el "túnel exprés").

&#x20;  - La IP del pepino en la malla 10.1.0.0/24 y el puerto UDP van en variables.

5\. En el host:

&#x20;  - DNAT del puerto UDP de WireGuard desde la IP pública hacia la IP del LXC, con reglas `post-up`/`post-down` en interfaces.j2 o con nftables. Elige una y justifícala.

&#x20;  - Ruta de la red WireGuard por la IP del LXC, para que haya camino de vuelta.

6\. El hub del Kimsufi NO lo toques. Genera en el README (o en un fichero aparte) el bloque `\[Peer]` exacto que tengo que añadir yo a mano en el hub, con la AllowedIPs del pepino y de su subred.



\## Cortafuegos (el de Proxmox), al final de todo

\- Plantillas para /etc/pve/firewall/cluster.fw y host.fw.

\- Define a mano el ipset `management`: la IP pública del Kimsufi, la IP fija de mi casa y 10.1.0.0/24, todo en variables. No dejes que Proxmox adivine la "red local", porque en Vultr sería un rango compartido con otros clientes.

\- Reglas:

&#x20; - SSH (22) y la web (8006) solo desde `management`.

&#x20; - El puerto UDP de WireGuard abierto a todo el mundo.

&#x20; - Todo lo demás, cerrado.

\- El cortafuegos de Proxmox no hace NAT: el DNAT va aparte, como en el punto anterior.

\- Activa el cortafuegos en la ÚLTIMA tarea, con la regla de SSH ya puesta. Antes de activarlo, comprueba que la IP desde la que conecta Ansible está dentro de `management`, y si no, para con un error claro. Si me quedo fuera, la consola de Vultr es gráfica y JAWS no la lee.



\## Lo que quiero que me entregues

\- El proyecto completo. Si tienes un repo a mano, en una rama nueva con su PR; si no, como un zip que me puedas mandar.

\- Pásale `ansible-playbook --syntax-check` y `ansible-lint` si puedes instalarlo. No lo lances contra ninguna máquina real.

\- Un README pensado para leerlo con lector de pantalla (JAWS): listas y títulos, nada de tablas ni diagramas ASCII. Que incluya:

&#x20; - Cómo preparar la VM genericcloud de pruebas en el Kimsufi: importar en qcow2, unidad de cloud-init con mi clave, CPU host y snapshot `limpio`.

&#x20; - Qué variables tengo que rellenar.

&#x20; - Cómo generar y cifrar las claves.

&#x20; - El bloque para el hub.

&#x20; - El orden para lanzarlo.

&#x20; - Qué hacer si un reinicio no vuelve.

\- Al final, una lista corta de supuestos que hayas tomado y de cosas que no hayas podido verificar (URLs de repos, nombres de paquetes de trixie, etc.).

