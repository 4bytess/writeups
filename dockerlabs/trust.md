# TRUST - DOCKERLABS

## INFORMACIÓN Y DESPLIEGUE DE LA MÁQUINA

Esta máquina es de [DockerLabs](https://dockerlabs.es)

Para descargarla vas a [DockerLabs](https://dockerlabs.es) y buscas la máquina Trust. Abres el enlace de mega, descargas el zip, lo descomprimes y ejecutas `bash ./auto_deploy.sh ./trust.tar`. De esta forma se ejecutará el contendor de docker de la máquina.

### descripción

- Dificultad: muy fácil
- Sistema Operativo: Linux
- Autor: El Pingüino de Mario

## RECONOCIMIENTO DE RED

### conectividad y SO

Comprobamos la conectividad con la máquina: `ping -c 1 172.18.0.2`.
> `-c 1`: solo enviar 1 paquete ICMP

![image](https://github.com/user-attachments/assets/f43fcbd4-961a-4924-947a-fc57db7d8364)

Vemos que nos devuelve respuesta por lo que tenemos conectividad. Como el TTL es 64 la máquina usa Linux.

### puertos y servicios

Hacemos un escaneo de todo el rango de puertos con nmap: `nmap -p- --open -sS --min-rate 5000 -vvv -n -Pn 172.18.0.2 -oG tcp_ports`.
> `-p-`: todo el rango de puertos

> `--open`: solo reportar los puertos abiertos

> `-sS`: usar el método de escaneo SYN, que es más sigiloso y rápido que el normal. Requiere privilegios a nivel de sistema para enviar raw packets

> `--min-rate 5000` enviar como mínimo 5000 paquetes por segundo

> `-vvv` triple verbose. Nada más descubrir un puerto lo muestra por pantalla, también muestra más información de lo normal sobre el escaneo

> `-n` no hacer resolución DNS

> `-Pn` no hacer host discovery

> `-oG tcp_ports` exportar el output en formato grepeable al archivo `tcp_ports`

#### puertos y servicios

**22 - TCP - SSH**

**80 - TCP - HTTP**

Hacemos un escaneo de servicios y versiones con nmap sobre los puertos abiertos: `nmap -p22,80 -sCV 172.18.0.2 -oN tcp_ports_targeted`.
> `-p22,80` hacer el escaneo sobre los puertos 22 y 80

> `-sCV` escanear el servicio y la versión (`sV`) y lanzar los scripts de reconocimiento por defecto (`sC`)

> `-oN tcp_ports_targeted` exportar el output en formato nmap al archivo `tcp_ports_targeted`

## RECONOCIMIENTO WEB

Primero lanzamos `whatweb`:

![image](https://github.com/user-attachments/assets/629d12ff-3a5d-426f-b61e-5892fb9a14ce)

No vemos ninguna información interesante a parte de la versión de Apache, así que pasamos a ver la web desde el navegador:

![image](https://github.com/user-attachments/assets/9d902b69-56df-41c3-aabd-56554101b5db)

Vemos la web estándar de Apache2 de Debian. Como no tiene nada interesante pasamos a hacer fuzzing con `gobuster dir -u http://172.18.0.2/ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x txt,php,py,html`:
> `dir` usar el modo de fuzzing de directorios

> `-u http://172.18.0.2/` la URL para fuzzear

> `-w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt` el diccionario para fuzzear

> `-x txt,php,py,html` para cada palabra del diccionario probamos las extensiones de archivos txt, php, py y html

![image](https://github.com/user-attachments/assets/01a69e0d-b511-44e9-bfa4-9d8cc438993c)

Vemos un archivo secret.php. Si lo cargamos vemos esto:

![image](https://github.com/user-attachments/assets/5ea94378-13c5-4c0b-8914-d00e5d61c0ea)

Esto nos puede dar información de un posible usuario en la máquina que se llame Mario, así que como el puerto 22 del ssh está abierto, probamos a hacer fuerza bruta con hydra con el rockyou.txt sobre el ssh: `hydra -l mario -P /usr/share/wordlists/rockyou.txt ssh://172.18.0.2`:
> `-l mario` indicamos el usuario

> `-P /usr/share/wordlists/rockyou.txt` el diccionario

![image](https://github.com/user-attachments/assets/5bf12628-0002-4ab9-b6e5-39f5d026e0ef)

Vemos que consigue conectarse con la contraseña chocolate. Así que nos conectamos: `ssh mario@172.18.0.2` y de contraseña chocolate.

## ESCALADA DE PRIVILEGIOS

Si nos conectamos como mario y hacemos `sudo -l` para ver nuestros privilegios especiales de sudo, vemos esto:

![image](https://github.com/user-attachments/assets/fea9e8ae-1a0e-443d-9b64-c16dab99613e)

Podemos ejecutar como cualquier usuario el binario `/usr/bin/vim`, así que para conseguir la consola como root hacemos `sudo /usr/bin/vim`:

![image](https://github.com/user-attachments/assets/1685f071-92fd-4f69-822f-7410aefe29e2)

Y ejecutamos en vim el comando `:shell` para ganar una consola como el usuario que esté ejecutando vim, que como es root, nos llega la consola como root:

![image](https://github.com/user-attachments/assets/bd0f460e-2552-4d4f-a2de-60398d126d08)

Y ya hemos ganado acceso a la máquina Trust como root.