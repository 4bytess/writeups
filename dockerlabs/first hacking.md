# FIRST HACKING - DOCKERLABS

## INFORMACIÓN Y DESPLIEGUE DE LA MÁQUINA

Esta máquina es de [DockerLabs](https://dockerlabs.es)

Para descargarla vas a [DockerLabs](https://dockerlabs.es) y buscas la máquina First Hacking. Abres el enlace de mega, descargas el zip, lo descomprimes y ejecutas `bash ./auto_deploy.sh ./firsthacking.tar`. De esta forma se ejecutará el contendor de docker de la máquina.

### descripción

- Dificultad: muy fácil
- Sistema Operativo: Linux
- Autor: El Pingüino de Mario

## RECONOCIMIENTO DE RED 

### conectividad y SO

Comprobamos la conectividad con la máquina: `ping -c 1 172.17.0.2`.
> `-c 1`: solo enviar 1 paquete ICMP

![image](https://github.com/user-attachments/assets/af0f2642-5f9c-4f8b-9080-8f1e41c2272d)

Vemos que tenemos conectividad ya que recibimos el paquete de vuelta, y además con un TTL de 64 por lo que el sistema operativo de la máquina es Linux.

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

**21 - TCP - FTP**

Hacemos un escaneo de servicios y versiones con nmap sobre el puerto abierto: `nmap -p21 -sCV 172.17.0.2 -oN tcp_ports_targeted`.
> `-p21` hacer el escaneo sobre el puerto 21.

> `-sCV` escanear el servicio y la versión (`sV`) y lanzar los scripts de reconocimiento por defecto (`sC`)

> `-oN tcp_ports_targeted` exportar el output en formato nmap al archivo `tcp_ports_targeted`

![image](https://github.com/user-attachments/assets/2aebb551-a400-44fe-8461-9d788bb89b73)

Vemos que se usa un servidor vsftpd de versión 2.3.4.

## RECONOCIMIENTO/EXPLOTACIÓN SERVIDOR FTP

Si buscamos en Google vsftpd 2.3.4 exploit, veremos etsa versión tiene una vulnerabilidad conocida. En uno de los resultados podemos ver un exploit en python de github: https://github.com/Hellsender01/vsftpd_2.3.4_Exploit/blob/main/exploit.py. Si vemos el código fuente del exploit, podemos entender en que consiste más o menos la vulnerabilidad. Para ser que si te intentas loguear al servidor ftp y pones al final del usuario dos puntos, por ejemplo test:, y de contraseña cualquier cosa, se abre el puerto 6200 en el servidor, y si te conectas ganas una consola. Podríamos clonarnos directamente este script para explotar el ftp, o crearnos también nuestro propio exploit ahora que entendemos la vulnerabilidad: https://github.com/4bytess/dockerlabs-scripts/blob/main/first%20hacking/ftp_exploit.py.

Ahora ejecutamos el exploit:

![image](https://github.com/user-attachments/assets/14ae5b4e-04d8-4810-ae62-6d91ed5de458)

Y ganamos acceso directamente como root a la máquina. Ganamos acceso en una consola de python, si queremos podemos pasarnos nuestra clave pública al directorio `/root/.ssh/authorized_keys` para poder conectarnos por ssh a la máquina.