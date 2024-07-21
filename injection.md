# INJECTION

## RECONOCIMIENTO DE RED

### conectividad y SO

Comprobamos al conectividad con la máquina: `ping -c 1 172.17.0.2`.
> `-c 1`: solo envía 1 paquete ICMP

![image](https://github.com/user-attachments/assets/53747492-e791-4a6c-9b42-ae246908f258)

Vemos que nos llega la respuesta de la máquina, con un TTL de 64 por lo que su SO es Linux.

### puertos y servicios

Hacemos un escaneo de todo el rango de puertos con nmap: `nmap -p- --open -sS --min-rate 5000 -vvv -n -Pn 172.17.0.2 -oG tcp_ports`.
> `-p-`: todo el rango de puertos

> `--open`: solo reportar los puertos abiertos

> `-sS`: usar el método de escaneo SYN, que es más sigiloso y rápido que el normal. Requiere privilegios a nivel de sistema para enviar raw packets

> `--min-rate 5000` enviar como mínimo 5000 paquetes por segundo

> `-vvv` triple verbose. Nada más descubrir un puerto lo muestra por pantalla, también muestra más información de lo normal sobre el escaneo

> `-n` no hacer resolución DNS

> `-Pn` No hacer host discovery

> `-oG tcp_ports` exportar el output en formato grepeable al archivo `tcp_ports`


#### puertos abiertos

**22 - TCP - SSH**
**80 - TCP - HTTP**

Hacemos un escaneo de servicios y versiones con nmap sobre los puertos abiertos: `nmap -p22,80 -sCV 172.17.0.2 -oN tcp_ports_targeted`.
> `-p22,80` hacer el escaneo sobre el puerto 22 y el 80.

> `-sCV` Escanear el servicio y la versión (`sV`) y lanzar los scripts de reconocimiento por defecto (`sC`)

> `-oN tcp_ports_targeted` Exportar el output en formato nmap al archivo `tcp_ports_targeted`

