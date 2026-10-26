# Laboratorio 13 — Hoja de trabajo: correo seguro (IMAPS, envío autenticado, SPF, DKIM y DMARC)

Nombre: Lizeth Rodríguez   Fecha: 26/10/2026

Requisito cumplido: Laboratorio 12 en 12/12 (Postfix en `mail` 10.10.10.25 y `mail2` 10.10.10.26).
Contraseña de los buzones de prueba: la del laboratorio (`Horizonte2026!`).

## Parte A — Instalación (en este orden)
1. mail: `instalar_correo_seguro.sh horizonte` → Dovecot, OpenDKIM y certificado de la CA de correo;
   escuchando en 25, 587 y 993 (el 143 no aparece).
2. ns1: `publicar_dns_correo.sh` → zona con serial 2026102202, `named-checkzone … OK` y `rndc reload` en la vista interna.
   Publicados: `DKIM: "v=DKIM1; h=sha256; k=rsa; " "p=MIIBIjAN…"`, `DMARC: "v=DMARC1; p=quarantine; …"` y `SPF: "v=spf1 mx -all"`.
3. mail2: `instalar_correo_seguro.sh familia` → verifica SPF (policyd-spf) y DKIM (OpenDKIM en modo `v`).
4. cliente: `verificar_lab13.sh` → primer resultado **10 de 15**. Fallaban las 5 pruebas que leen por IMAPS con `curl`,
   porque el cliente **no tenía `curl`**. Instalado `curl` → **15 de 15**.

Los TODO se completaron antes de instalar:

| Archivo | Valor | Para qué |
|---|---|---|
| `dovecot/99-horizonte.conf` | `ssl = required` | Con `yes` todavía se podía entrar sin cifrar; `required` exige TLS |
| `dovecot/99-horizonte.conf` | `inet_listener imap { port = 0 }` | Cierra el IMAP en claro (143); solo queda IMAPS 993 |
| `dovecot/99-horizonte.conf` | `service auth { unix_listener /var/spool/postfix/private/auth { mode = 0660, user/group postfix } }` | Postfix pregunta a Dovecot si usuario y contraseña son correctos (SASL) |
| `postfix/submission.txt` | `smtpd_tls_security_level=encrypt` | En el 587, TLS obligatorio antes de todo |
| `postfix/submission.txt` | `smtpd_relay_restrictions=permit_sasl_authenticated,reject` y `smtpd_recipient_restrictions=permit_sasl_authenticated,reject` | Solo usuarios autenticados envían |
| `postfix/submission.txt` | `smtpd_sender_restrictions=reject_sender_login_mismatch` | Cada usuario solo envía con SU dirección |
| `dns/dmarc.txt` | `"v=DMARC1; p=quarantine; rua=mailto:dmarc@horizonte.internal; adkim=s; aspf=s"` | De solo observar a cuarentena, con informes y alineación estricta |
| `opendkim/*.conf` | `Nameservers 10.10.10.4` | OpenDKIM consulta el resolutor del colegio y no el DNS interno de Docker |

## Parte B — Leer el correo con IMAPS (Dovecot)
**2.** `curl -s imaps://mail.horizonte.internal/INBOX --user 'secretaria:…' -X 'STATUS INBOX (MESSAGES)'` →
`* STATUS INBOX (MESSAGES 1)`: la secretaria se autenticó por TLS y Dovecot leyó su Maildir.

**3.** `openssl s_client -connect mail.horizonte.internal:993` → `subject=CN = mail.horizonte.internal`,
`issuer=O = Laboratorio de correo TW1, CN = CA Correo Laboratorio` y **`Verification: OK`**. Lo emitió la CA de correo del
laboratorio, que el cliente tiene instalada como confiable. Ya no es el certificado autofirmado del Lab 12.

**4.** `curl imap://mail.horizonte.internal/` (143) → sin respuesta (`143 cerrado`). Se cerró porque IMAP en claro
enviaría usuario y contraseña legibles, igual que TELNET en el Lab 10. Con `port = 0` el servicio ni escucha.

**5. Thunderbird con dos usuarios.** En Windows:
- Se agregó `127.0.0.1 mail.horizonte.internal` al archivo `hosts`, para que el nombre coincida con el certificado
  (`Test-NetConnection … -Port 9993` → `TcpTestSucceeded : True`).
- Se importó `ca-correo.crt` en **Autoridades**.
- Se crearon las cuentas **secretaria** y **rosario.vaca**: IMAP `mail.horizonte.internal:9993` SSL/TLS y SMTP
  `:9587` STARTTLS, con contraseña normal y el usuario sin @dominio.

Docker publica el 993 y el 587 del nodo `mail` en los puertos 9993 y 9587 de la PC.

Prueba: Secretaría envió **"Inscripciones 2027"** a la directora y llegó a la bandeja de rosario.vaca a las 09:09.
Su código fuente (Ctrl+U) muestra:

```
Return-Path: <secretaria@horizonte.internal>
DKIM-Signature: v=1; a=rsa-sha256; c=relaxed/simple; d=horizonte.internal; s=mail; t=1793020188;
    h=Date:To:From:Subject:From; …
Received: from [127.0.0.1] (unknown [10.10.10.1])
    by mail.horizonte.internal (Postfix) with ESMTPSA id 3C7ECB4B7 for <rosario.vaca@horizonte.internal>
User-Agent: Mozilla Thunderbird
From: Secretaría <secretaria@horizonte.internal>
Subject: Inscripciones 2027
```

- **`with ESMTPSA`**: SMTP extendido con **S**ecuridad (TLS) y **A**utenticación, es decir, el envío por 587.
- **`[10.10.10.1]`**: la puerta de enlace de Docker, por donde entra la PC con Windows.
- **`h=…From:Subject:From`**: el `From` aparece dos veces porque está *sobrefirmado* (`OversignHeaders From`), así
  nadie puede agregar un segundo `From:` sin romper la firma.

## Parte C — Enviar con usuario y contraseña (puerto 587)
| Prueba | Resultado | ¿Por qué? |
|---|---|---|
| 587 con TLS pero sin usuario | `<~* 553 5.7.1 <secretaria@horizonte.internal>: Sender address rejected: not logged in` | Solo usuarios autenticados pueden enviar (`reject_sender_login_mismatch` + `permit_sasl_authenticated,reject`) |
| 587 con usuario pero sin TLS | `*** Host did not advertise authentication` | `smtpd_tls_auth_only=yes` y `encrypt`: sin TLS el servidor ni ofrece AUTH, así la contraseña nunca viaja en claro |
| 587 con TLS y usuario secretaria | `235 2.7.0 Authentication successful` y `250 2.0.0 Ok: queued as B0F78B273` | Cifrado + credenciales correctas: envío aceptado |
| secretaria enviando como rosario.vaca@ | `<~* 553 5.7.1 <rosario.vaca@horizonte.internal>: Sender address rejected: not owned by user secretaria` | `smtpd_sender_login_maps` asocia cada dirección con su usuario |

## Parte D — SPF, DKIM y DMARC
**1. Registros publicados** (desde el cliente con `dig`):

| Registro | Valor | Qué dice |
|---|---|---|
| `horizonte.internal MX` | `10 mail.horizonte.internal.` | Quién recibe el correo del dominio |
| `horizonte.internal TXT` (SPF) | `"v=spf1 mx -all"` | Solo los servidores del MX pueden enviar como @horizonte.internal; el resto falla (`-all`) |
| `mail._domainkey.horizonte.internal TXT` (DKIM) | `"v=DKIM1; h=sha256; k=rsa; p=MIIBIjAN…"` | La clave pública RSA 2048 con la que se verifican las firmas del selector `mail` |
| `_dmarc.horizonte.internal TXT` | `"v=DMARC1; p=quarantine; rua=mailto:dmarc@horizonte.internal; adkim=s; aspf=s"` | Si no pasa SPF o DKIM **alineados**, a cuarentena; informes a dmarc@; alineación estricta |

**2. OpenDKIM y el DNS.** Se completó por adelantado `Nameservers 10.10.10.4`. `dig` funciona porque usa
`/etc/resolv.conf` (el DNS de Docker, que reenvía al resolutor), pero OpenDKIM usa su propio resolvedor. En el
contenedor, sin esa opción, no llegaría a la zona interna del colegio y diría `key not found in DNS`. Resultado:
**`dkim=pass`** en el destino.

**3.** `dns/dmarc.txt` completado y publicado: `p=quarantine` aparece con `dig`. Al publicar, BIND avisó
`dmarc-horizonte.inc:2: file does not end with newline`, un aviso de formato sin consecuencias (la zona cargó `OK`).

**4. Cabeceras del correo recibido por rosa.mamani (leído por IMAPS en mail2):**

```
Return-Path: <secretaria@horizonte.internal>
Authentication-Results: mail.familia.internal; dkim=pass … header.d=horizonte.internal
Received-SPF: Pass (mailfrom) identity=mailfrom; client-ip=10.10.10.25; helo=mail.horizonte.internal; envelope-from=secretaria@horizonte.internal; receiver=familia.internal
Received: from mail.horizonte.internal (red_dns-mail-1.red_dns_aula [10.10.10.25])
DKIM-Signature: v=1; a=rsa-sha256; c=relaxed/simple; d=horizonte.internal; s=mail; …
Received: from cliente (red_dns-cliente-1.red_dns_aula [10.10.10.50])
From: secretaria@horizonte.internal
```

- **`Received:`** (de abajo hacia arriba): el cliente 10.10.10.50 entregó a mail.horizonte.internal por el 587, y este
  lo pasó a mail.familia.internal desde 10.10.10.25.
- **`DKIM-Signature`**: firma de mail.horizonte.internal. `a=rsa-sha256` es el algoritmo, `c=relaxed/simple` la
  normalización, `d=horizonte.internal` el dominio que firma y `s=mail` el selector que dice dónde está la clave
  pública. También incluye la lista de cabeceras firmadas y el resumen del cuerpo.
- **`Received-SPF: Pass`**: la IP 10.10.10.25 está autorizada por el SPF del dominio del sobre.
- **`Authentication-Results`**: el resumen del receptor: DKIM pasó para `horizonte.internal`.

**5.** `spf.check2`: `10.10.10.25 -> ('pass', 'sender SPF authorized')` y
`10.10.10.26 -> ('fail', 'SPF fail - not authorized')`. Solo el MX del colegio puede enviar como
@horizonte.internal. Si mail2 (u otro) lo intentara, el SPF fallaría y DMARC lo mandaría a cuarentena.

**6.** `opendkim-testmsg` sobre el correo original → sin errores (**firma válida**). Sobre una copia con el asunto
cambiado (`Subject: ALTERADO …`) → **`dkim_eom(): Bad signature`**. DKIM protege la integridad: cualquier cambio en
las cabeceras firmadas o en el cuerpo rompe la firma.

**7. Alineación DMARC.** El `From:` visible es `secretaria@horizonte.internal`. La firma DKIM tiene
`d=horizonte.internal` y el SPF pasó para `envelope-from=secretaria@horizonte.internal`. Los dos dominios coinciden
**exactamente** con el del `From:`, que es lo que exige la alineación estricta (`adkim=s`, `aspf=s`). Un atacante
podría pasar SPF con su propio dominio en el sobre, pero no alinearlo con el `From:` del colegio. Por eso DMARC
protege la dirección que ve el usuario.

## Parte E — Verificación
`bash /lab/lab13/verificar_lab13.sh` → **Resultado: 15 de 15 comprobaciones.**

## Reflexión
Google Workspace o Microsoft 365 hacen por ti casi todo lo de hoy: certificados válidos, IMAPS y envío autenticado,
firma DKIM con rotación de claves, verificación de SPF, DKIM y DMARC del correo entrante, filtros de spam y virus,
y la reputación de sus IPs. Lo que **sigue siendo responsabilidad del administrador del dominio es el DNS**:
publicar el MX que apunta a ellos, el SPF que los autoriza (`include:`), la clave DKIM que ellos generan, la política
DMARC y leer sus informes antes de pasar de `p=none` a `quarantine` o `reject`. También la gestión de cuentas y
contraseñas (o MFA) de cada persona. Si el DNS está mal, ni el mejor proveedor logra que el correo del colegio llegue
o que no sea suplantado.
