# PICADILLY - DOCKERLABS

## INFORMACIÓN Y DESPLIEGUE DE LA MÁQUINA

Esta máquina es de [DockerLabs](https://dockerlabs.es)

Para descargarla vas a [DockerLabs](https://dockerlabs.es) y buscas la máquina Picadilly. Abres el enlace de mega, descargas el zip, lo descomprimes y ejecutas `bash ./auto_deploy.sh ./picadilly.tar`. De esta forma se ejecutará el contendor de docker de la máquina.

### descripción

- Dificultad: fácil
- Sistema Operativo: linux
- Autor: kaikoperez

## RECONOCIMIENTO DE RED

### conectividad y SO

Primero hacemos `ping -c 1 172.17.0.2` para enviar un paquete ICMP a la máquina.
> `-c 1` Solo enviar 1 paquete ICMP

![image](https://github.com/user-attachments/assets/12171f6c-fbb0-4bbf-8e74-f453568dc17b)

Y nos llega respuesta. Como el TTL es de 64 el sistema operativo de la máquina muy posiblemente sea Linux.

### puertos y servicios

Hacemos un escaneo de todo el rango de puertos con nmap:

```bash
nmap -p- --open -sS --min-rate 5000 -vvv -n -Pn 172.17.0.2 -oG tcp_ports
```

> `-p-`: todo el rango de puertos

> `--open`: solo reportar los puertos abiertos

> `-sS`: usar el método de escaneo SYN, que es más sigiloso y rápido que el normal. Requiere privilegios a nivel de sistema para enviar raw packets

> `--min-rate 5000` enviar como mínimo 5000 paquetes por segundo

> `-vvv` triple verbose. Nada más descubrir un puerto lo muestra por pantalla, también muestra más información de lo normal sobre el escaneo

> `-n` no hacer resolución DNS

> `-Pn` no hacer host discovery

> `-oG tcp_ports` exportar el output en formato grepeable al archivo `tcp_ports`

