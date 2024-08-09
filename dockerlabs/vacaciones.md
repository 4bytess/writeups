# VACACIONES - DOCKERLABS

## INFORMACIÓN Y DESPLIEGUE DE LA MÁQUINA

Esta máquina es de [DockerLabs](https://dockerlabs.es)

Para descargarla vas a [DockerLabs](https://dockerlabs.es) y buscas la máquina Vacaciones. Abres el enlace de mega, descargas el zip, lo descomprimes y ejecutas `bash ./auto_deploy.sh ./vacaciones.tar`. De esta forma se ejecutará el contendor de docker de la máquina.

### descripción

- Dificultad: muy fácil
- Sistema Operativo: Linux
- Autor: Romabri

## RECONOCIMIENTO DE RED

### conectividad y SO

Usamos ping para enviar un paquete ICMP a la máquina para ver si tenemos conectividad: `ping -c 1 172.17.0.2`.
> `-c 1`: solo enviar 1 paquete ICMP

![image](https://github.com/user-attachments/assets/0ca709d8-d520-41a4-8064-20c72591f930)

Vemos que tenemos conectividad ya que la máquina nos responde. Como tiene TTL 64 muy posiblemente use Linux.

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

Hacemos un escaneo de servicios y versiones con nmap sobre los puertos 22 y 80: `nmap -p22,80 -sCV 172.17.0.2 -oN tcp_ports_targeted`.
> `-p22,80` hacer el escaneo sobre los puertos 22 y 80.

> `-sCV` escanear el servicio y la versión (`sV`) y lanzar los scripts de reconocimiento por defecto (`sC`)

> `-oN tcp_ports_targeted` exportar el output en formato nmap al archivo `tcp_ports_targeted`

## RECONOCIMIENTO WEB

Antes de nada, hacemos `whatweb http://172.17.0.2`:

![image](https://github.com/user-attachments/assets/3be4bfa0-73b6-4059-977a-9a58df7a495d)

No vemos nada interesante a parte de la versión de Apache.

Pasamos a ver la web desde el navegador:

![image](https://github.com/user-attachments/assets/b842e88e-8209-47d8-945d-8921d949162f)

Y vemos que no hay absolutamente nada. Pero si insepccionamos el código de la web, está este comentario:

![image](https://github.com/user-attachments/assets/8edca81e-3d3d-4e19-88ef-33201927279d)

## EXPLOTACIÓN SSH

Esto nos da información de dos posibles usuario a nivel de sistema: Juan y Camilo. Podemos hacer fuerza bruta por ssh con el rockyou: `hydra -l camilo -P /usr/share/wordlists/rockyou.txt ssh://172.17.0.2` y `hydra -l juan -P /usr/share/wordlists/rockyou.txt ssh://172.17.0.2`.
> `-l camilo/juan` especificamos el usuario

> `-P /usr/share/wordlists/rockyou.txt ssh://172.17.0.2` especificamos el diccionario

Y vemos que para Juan no saca ninguna contraseña, pero para camilo si que encuentra:

![image](https://github.com/user-attachments/assets/2465fdab-7763-43ed-ae97-c5515bbc78c4)

Parece ser que la contraseña de camilo es password1. Nos conectamos por ssh `ssh camilo@172.17.0.2` y de contraseña password1.

## ESCALADA DE PRIVILEGIOS (JUAN)

Si hacemos `env` para ver nuestras variables de entorno, vemos una variable llamada MAIL que vale /var/mail/camilo. Si vamos a esa ruta vemos que hay un archivo llamado correo.txt, que tiene este contenido:

![image](https://github.com/user-attachments/assets/b9cda9ad-a7cd-48d6-8376-d21b0fb8ceab)

Nos dice que la contraseña de Juan es 2k84dicb, así que hacemos `su juan` y metemos la contraseña.

## ESCALADA DE PRIVILEGIOS (ROOT)

Siendo el usuario Juan, hacemos `sudo -l`, y vemos esto:

![image](https://github.com/user-attachments/assets/c45168ab-0561-4537-b198-37d824a12c75)

Vemos que podemos ejecutar como cualquier usuario, sin poner contraseña, el binario `/usr/bin/ruby`, así que nos spawneamos una bash como root: `sudo ruby -e 'exec "/bin/bash"'`:

![image](https://github.com/user-attachments/assets/5ab5c200-8a85-4648-b8e8-0ccd47796109)

Y ya somos root en la máquina Vacaciones.