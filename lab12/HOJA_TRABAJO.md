# Laboratorio 12 — Hoja de trabajo: el correo del colegio con Postfix

Nombre: Lizeth Rodríguez   Fecha: 26/10/2026

`red_dns` ampliada con los nodos `mail` (10.10.10.25, horizonte.internal) y `mail2` (10.10.10.26, familia.internal),
ambos usando el resolutor 10.10.10.4. Se reinstalaron ns1/ns2 (Lab 7), las vistas y DNSSEC (Lab 9) y el resolutor
Unbound (Lab 8). Buzones de prueba: `ti`, `secretaria`, `rosario.vaca` y `rosa.mamani` (contraseña de laboratorio).

## Parte A — El DNS del correo (en el nodo cliente)
| Comando | Resultado | ¿Qué información aporta? |
|---|---|---|
| `dig +short horizonte.internal MX` | `10 mail.horizonte.internal.` | A qué servidor se entrega el correo de @horizonte.internal (prioridad 10) |
| `dig +short familia.internal MX` | `10 mail.familia.internal.` | El servidor de correo del otro dominio |
| `dig +short mail.horizonte.internal A` | `10.10.10.25` | La IP a la que se conecta el remitente (puerto 25) |
| `dig +short -x 10.10.10.25` | `red_dns-mail-1.red_dns_aula.` | El PTR que inventa el DNS interno de Docker |
| `dig +short @10.10.10.4 -x 10.10.10.25` | `mail.horizonte.internal.` | El PTR real de la zona inversa del colegio |

**1. ¿Por qué los dos PTR son distintos?** El cliente usa como DNS `127.0.0.11`, el servidor DNS interno de Docker.
Docker responde él mismo las inversas de sus contenedores con el nombre del contenedor y la red
(`red_dns-mail-1.red_dns_aula.`), y solo reenvía el resto a 10.10.10.4. Al preguntar directo al resolutor del
colegio, este consulta la zona `10.10.10.in-addr.arpa` de ns1/ns2 (Lab 7) y responde `mail.horizonte.internal.`.

**2. ¿Para qué MX, A y PTR?** El **MX** dice qué servidor recibe el correo de un dominio. El **A** da la IP de ese
servidor. El **PTR** sirve para que quien recibe compruebe que la IP que se conecta tiene un nombre coherente con
el saludo (`EHLO`): muchos servidores rechazan o marcan como spam el correo de IPs sin PTR o con un PTR genérico.

## Parte B — Instalar y configurar (6 TODO)
1. Resolutor: `instalar_dns_correo.sh` → `MX de familia.internal: 10 mail.familia.internal.`
2. mail2: `instalar_correo.sh familia` → Postfix de familia.internal con `mynetworks = 127.0.0.0/8`.
3. mail, con los archivos tal como vienen: `mynetworks = 0.0.0.0/0`, `mydestination = $myhostname, localhost`.
   Verificador: **5 de 12**.
4. **Prueba de relay** (desde mail hacia 10.10.10.25, que no está en `mynetworks` correcta):

```
<-  220 mail.horizonte.internal ESMTP Postfix (Ubuntu)
<-  250-VRFY
 -> MAIL FROM:<prueba@familia.internal>
 -> RCPT TO:<rosa.mamani@familia.internal>
<-  250 2.0.0 Ok: queued as ABED2C724
```

Sí: el servidor del colegio aceptó reenviar un correo **de** familia.internal **a** familia.internal. Es un
**relay abierto**. Cualquiera en internet podría usarlo para mandar spam o phishing con la IP del colegio. Esa IP
terminaría en listas negras y el correo legítimo del colegio dejaría de llegar. Además, el saludo revelaba
`Postfix (Ubuntu)` y `VRFY` permitía averiguar qué usuarios existen.

5. TODO completados:

| Archivo | Valor | Para qué |
|---|---|---|
| `horizonte.cf` | `smtpd_banner = $myhostname ESMTP` | No revelar software ni sistema |
| `horizonte.cf` | `mydestination = $myhostname, $mydomain, localhost.$mydomain, localhost` | Aceptar @horizonte.internal como destino final |
| `horizonte.cf` | `mynetworks = 127.0.0.0/8 10.10.10.50/32` | Solo el servidor y la PC de secretaría pueden enviar a otros dominios |
| `horizonte.cf` | `home_mailbox = Maildir/` | Un archivo por correo (lo leerá Dovecot en el Lab 13) |
| `horizonte.cf` | `disable_vrfy_command = yes` | No revelar qué usuarios existen |
| `aliases` | `inscripciones: secretaria, rosario.vaca` | Una dirección que reparte a dos personas |

Al reinstalar, la misma prueba respondió **`<** 554 5.7.1 <rosa.mamani@familia.internal>: Relay access denied`**,
el saludo quedó `220 mail.horizonte.internal ESMTP` y `VRFY` desapareció del EHLO.
`verificar_lab12.sh` → **Resultado: 12 de 12 comprobaciones.**

6. **`mydestination` vs `mynetworks`.** `mydestination` dice **para qué dominios este servidor es el destino final**:
el correo para @horizonte.internal se entrega en los buzones locales. `mynetworks` dice **quién puede usar el
servidor para enviar a otros dominios** (relay) sin autenticarse. Uno controla lo que entra para nosotros; el otro,
a quién le hacemos de cartero hacia fuera. `smtpd_relay_restrictions` aplica esa regla: permite a `mynetworks` y a
los autenticados, y rechaza el resto (`reject_unauth_destination`).

## Parte C — SMTP "a mano"
Desde el cliente con `nc -C mail.horizonte.internal 25`:

| Envío | Respuesta | Significado |
|---|---|---|
| (conexión) | `220 mail.horizonte.internal ESMTP` | Servidor listo |
| `EHLO pc-secretaria.horizonte.internal` | `250-PIPELINING … 250-STARTTLS … 250 CHUNKING` | Saludo aceptado y lista de extensiones |
| `MAIL FROM:<secretaria@horizonte.internal>` | `250 2.1.0 Ok` | Remitente del sobre aceptado |
| `RCPT TO:<ti@horizonte.internal>` | `250 2.1.5 Ok` | Destinatario aceptado |
| `DATA` | `354 End data with <CR><LF>.<CR><LF>` | Puede enviar el mensaje; termina con una línea con un punto |
| (mensaje y `.`) | `250 2.0.0 Ok: queued as E26568BB3` | Mensaje en cola con ese ID |
| `QUIT` | `221 2.0.0 Bye` | Fin de la sesión |

**Extensiones:** PIPELINING, SIZE 26214400, ETRN, STARTTLS, ENHANCEDSTATUSCODES, 8BITMIME, DSN, SMTPUTF8 y CHUNKING.
**VRFY no aparece**: si estuviera, cualquiera podría preguntar `VRFY rosario.vaca` y saber qué cuentas existen, un
dato útil para fuerza bruta o phishing dirigido.

**Sobre vs cabecera:** `MAIL FROM` es el **sobre**: lo usan los servidores para entregar y para devolver los rebotes
(queda como `Return-Path`). La cabecera `From:` es lo que **ve el usuario** en su programa de correo, y puede ser
distinta. Por eso un atacante puede falsificar el `From:`. SPF revisa el sobre, y DKIM y DMARC (Lab 13) protegen el
`From:` visible.

## Parte D — Seguir un correo
**1.** En `mail.log`:

```
6F937C72C: to=<rosa.mamani@familia.internal>, relay=mail.familia.internal[10.10.10.26]:25, delay=0.62,
  delays=0.17/0.1/0.29/0.06, dsn=2.0.0, status=sent (250 2.0.0 Ok: queued as E795BC72D)
```

- `relay=` es el servidor al que se entregó (el MX de familia.internal, en el puerto 25).
- `delay=` es el tiempo total en segundos. `delays=a/b/c/d` lo divide en antes de la cola, en la cola, conexión y
  transmisión.
- `dsn=2.0.0` es el código de estado: 2 = éxito.
- `status=sent` indica que se entregó; el texto final es la respuesta del otro servidor, con **su** ID de cola.

**2. inscripciones@** llegó a **dos** buzones:

```
51183C272: to=<secretaria@horizonte.internal>, orig_to=<inscripciones@horizonte.internal>, relay=local, status=sent (delivered to maildir)
51183C272: to=<rosario.vaca@horizonte.internal>, orig_to=<inscripciones@horizonte.internal>, relay=local, status=sent (delivered to maildir)
```

`orig_to=` muestra la dirección original y `to=` el buzón real tras expandir el alias.

**3. Cabeceras en mail2** (`/home/rosa.mamani/Maildir/new`):

```
Return-Path: <ti@horizonte.internal>
Received: from mail.horizonte.internal (red_dns-mail-1.red_dns_aula [10.10.10.25])
Received: from mail.horizonte.internal (localhost [127.0.0.1])
```

Cada servidor que toca el correo agrega una línea `Received:` **arriba**, así que se leen de abajo hacia arriba. El
correo entró en mail.horizonte.internal por `localhost` (el envío de prueba) y luego mail.familia.internal lo recibió
desde 10.10.10.25. En el nombre entre paréntesis aparece el PTR del DNS de Docker. `Return-Path` es el `MAIL FROM`
del sobre.

**4.** `postqueue -p` → `Mail queue is empty`. Tendría correos si un destino no responde o da un error temporal
(4xx): Postfix los guarda y reintenta durante días antes de rebotarlos.

## Parte E — TLS
`openssl s_client -starttls smtp -connect mail.horizonte.internal:25` → **`New, TLSv1.3, Cipher is TLS_AES_256_GCM_SHA384`**,
`subject=CN = mail.horizonte.internal` e `issuer=CN = mail.horizonte.internal`. Es un certificado **autofirmado**: el
de prueba de Ubuntu (snakeoil). En el Lab 13 se reemplaza por uno de una CA del laboratorio.

**¿Por qué se acepta un certificado no verificable entre servidores?** Porque entre MTA el TLS es **oportunista**
(`smtp_tls_security_level = may`). Es mejor cifrar sin verificar que mandar en claro, y si se exigiera un
certificado válido muchos servidores no podrían entregarse correo. Esto protege contra quien solo escucha, pero no
contra un intermediario activo. Para exigir TLS verificado existen DANE y MTA-STS.

## Problema del entorno
Al instalar herramientas, `apt-get update` devolvió `403 Forbidden` desde `archive.ubuntu.com` y `security.ubuntu.com`
por HTTP en todos los nodos: la red bloqueaba los repositorios por el puerto 80. Se cambiaron las fuentes a un espejo
por **HTTPS** (`mirrors.edge.kernel.org/ubuntu`). Los paquetes siguen verificándose con la firma de Ubuntu.

## Reflexión
La mayoría de las instituciones usa correo gestionado (Google Workspace, Microsoft 365) porque operar correo propio
exige mucho: mantener el servidor actualizado, evitar que sea un relay abierto, cuidar la reputación de la IP para
no caer en listas negras, filtrar spam y virus, tener respaldos y alta disponibilidad, y configurar bien SPF, DKIM y
DMARC. Un error de configuración, como el `mynetworks = 0.0.0.0/0` del técnico anterior, puede dejar a todo el
colegio sin correo. Un servidor propio tiene sentido cuando se necesita control total de los datos (privacidad o
normas legales), una red aislada, integración con sistemas internos o fines educativos, como este laboratorio.
