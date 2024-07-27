# SHOW TIME - DOCKERLABS

## INFORMACIÓN Y DESPLIEGUE DE LA MÁQUINA

Esta máquina es de [DockerLabs](https://dockerlabs.es)

Para descargarla vas a [DockerLabs](https://dockerlabs.es) y buscas la máquina Show Time. Abres el enlace de mega, descargas el zip, lo descomprimes y ejecutas `bash ./auto_deploy.sh ./showtime.tar`. De esta forma se ejecutará el contendor de docker de la máquina.

### descripción

- Dificultad: fácil
- Sistema Operativo: linux
- Autor: maciiii___

## RECONOCIMIENTO DE RED

### conectividad y SO

Comprobamos nuestra conectividad con la máquina: `ping -c 1 172.17.0.2`
> `-c 1` Solo enviar 1 paquete ICMP

![image](https://github.com/user-attachments/assets/75d6b7e0-7f86-440e-877b-7a8a41733a10)

Vemos que nos llega la respuesta de vuelta de la máquina, con TTL de 64 por lo que su SO es Linux.

### puertos y servicios

Hacemos un escaneo de todo el rango de puertos con nmap: `nmap -p- --open -sS --min-rate 5000 -vvv -n -Pn 172.17.0.2 -oG tcp_ports`.
> `-p-`: todo el rango de puertos

> `--open`: solo reportar los puertos abiertos

> `-sS`: usar el método de escaneo SYN, que es más sigiloso y rápido que el normal. Requiere privilegios a nivel de sistema para enviar raw packets

> `--min-rate 5000` enviar como mínimo 5000 paquetes por segundo

> `-vvv` triple verbose. Nada más descubrir un puerto lo muestra por pantalla, también muestra más información de lo normal sobre el escaneo

> `-n` no hacer resolución DNS

> `-Pn` no hacer host discovery

> `-oG tcp_ports` exportar el output en formato grepeable al archivo `tcp_ports`

#### puertos abiertos

**22 - TCP - SSH**

**80 - TCP - HTTP**

Hacemos un escaneo de servicios y versiones con nmap sobre los puertos abiertos: `nmap -p22,80 -sCV 172.17.0.2 -oN tcp_ports_targeted`.
> `-p22,80` hacer el escaneo sobre el puerto 22 y el 80.

> `-sCV` escanear el servicio y la versión (`sV`) y lanzar los scripts de reconocimiento por defecto (`sC`)

> `-oN tcp_ports_targeted` exportar el output en formato nmap al archivo `tcp_ports_targeted`

## RECONOCIMIENTO WEB

Antes de ir a ver la web desde el navegador, lanzamos desde consola el comando `whatweb http://172.17.0.2`. Vemos que la web es un Apache 2.4.58 y que la distro de la máquina es Ubuntu.

Ahora vemos la raíz de la web desde el navegador, y vemos este contenido:

![image](https://github.com/user-attachments/assets/7dd79aeb-48eb-42b6-8cc4-3669027a8b84)