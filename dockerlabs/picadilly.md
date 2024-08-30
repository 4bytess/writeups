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

```python
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

![image](https://github.com/user-attachments/assets/57808989-a8cf-4a58-8e5d-dfc8556aba8f)

Están abiertos el puerto 80 y 443. Ahora lanzamos un escaneo de versiones con nmap, y también lanzamos scripts de reconocimiento estándar:

```python
nmap -p80,443 -sCV 172.17.0.2 -oN targeted
```

> `-p80,443` hacer el escaneo solo sobre estos dos puertos

> `-sCV` lanzar escaneo de versiones y scripts estándar de reconocimiento

> `-oN targeted` exportar el output al archivo `targeted`, en formato nmap

![image](https://github.com/user-attachments/assets/f999fd7c-aabf-4757-9d07-d429fbf654c4)

Podemos ver varios datos, como las versiones de Apache o algún nombre de dominio. También vemos que nos ha encontrado un archivo `backup.txt` en el servidor web del puerto 80.

## RECONOCIMIENTO WEB

Empezamos lanzando desde consola el comando `whatweb` sobre el puerto 80:

```python
whatweb http://172.17.0.2
```

![image](https://github.com/user-attachments/assets/005cf3cf-fbd7-4556-901b-ea8a65c06a58)

Vemos que el título es `Index of /`, por lo que alomejor hay directory listing. Cargamos el servidor en el navegador:

![image](https://github.com/user-attachments/assets/90de0b8f-6620-4a3f-add7-66b641b5912a)

Sí que se aplica directory listing, también vemos el archivo `backup.txt` que encontró nmap:

![image](https://github.com/user-attachments/assets/8b3fd0fb-3661-45dc-81e3-83fcb1fe6661)

Nos da una contraseña cifrada que nos puede servir más adelante. Pero como la máquina no tiene servidor SSH no podemos usarla todavía.

Pasamos al puerto 443:

```python
whatweb https://172.17.0.2
```

