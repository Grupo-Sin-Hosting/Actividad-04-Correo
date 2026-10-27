# Roles de la defensa â€” Actividad 04

| Integrante | GitHub | Rama | Parte | Debe poder explicar |
|---|---|---|---|---|
| Lucas Carrasco | @Lukitas2005 | `rol-dns-correo` | Entorno y DNS del correo | Nodos mail/mail2, MX, A y los dos PTR (Docker vs resolutor), por quÃ© un servidor de correo necesita MX/A/PTR |
| Jacqueline CÃ¡ceres | @jacquelinesubia321-spec | `rol-postfix` | Postfix: del relay abierto al servidor seguro (Lab 12 B) | `mynetworks` vs `mydestination`, prueba de relay 250 â†’ 554, banner, VRFY, alias inscripciones |
| JosÃ© MarÃ­a Orihuela | â€” | `rol-smtp` | SMTP a mano, recorrido y TLS (Lab 12 Câ€“E) | CÃ³digos 220/250/354/221, sobre vs `From:`, `mail.log` (relay, delay, dsn, status, orig_to), STARTTLS oportunista |
| Lizeth RodrÃ­guez | @lixinrodriguez1-eng | `rol-imaps-587` | Dovecot IMAPS y envÃ­o autenticado (Lab 13 Bâ€“C) + coordinaciÃ³n | `ssl = required`, 143 cerrado, SASL, las 4 pruebas del 587; repositorio |
| Juniors Parapaino | @ederjuniorparapaino | `rol-thunderbird` | Thunderbird con dos usuarios y anÃ¡lisis de cabeceras | ConfiguraciÃ³n 9993/9587, CA importada, correo secretaria â†’ direcciÃ³n, `Received`, `DKIM-Signature`, `Authentication-Results` |
| Limber Quispe | â€” | `rol-spf-dkim-dmarc` | SPF, DKIM y DMARC (Lab 13 D) | Registros con `dig`, SPF pass/fail (.25/.26), firma rota al alterar, alineaciÃ³n DMARC, `Nameservers` de OpenDKIM |

Cada integrante: crea su rama, agrega abajo un resumen de su parte con sus palabras y abre un Pull Request con `Closes #N`.

## Notas de cada integrante

### Lizeth RodrÃ­guez â€” Dovecot IMAPS y envÃ­o autenticado
Dovecot quedÃ³ con `ssl = required` y el puerto 143 cerrado, asÃ­ que el correo solo se lee por IMAPS (993) con un certificado de la CA del laboratorio (Verification: OK). Postfix valida usuarios con Dovecot (SASL). En el 587, sin TLS ni siquiera se ofrece la autenticaciÃ³n; sin usuario se rechaza con 553; con TLS y usuario se acepta (235); y nadie puede enviar con la direcciÃ³n de otra persona. TambiÃ©n coordinÃ© el repositorio.

### Jacqueline CÃ¡ceres â€” Postfix y relay
La configuraciÃ³n inicial funcionaba, pero era un relay abierto: aceptÃ³ con 250 un correo de un dominio ajeno a otro ajeno (5/12). `mydestination` dice quÃ© dominios se entregan aquÃ­ y `mynetworks` quiÃ©n puede usar el servidor para enviar a otros. Lo dejamos solo en 127.0.0.0/8 y la PC de secretarÃ­a, y la misma prueba respondiÃ³ `554 Relay access denied`. TambiÃ©n quitamos la versiÃ³n del saludo, desactivamos VRFY y el alias inscripciones llega a dos buzones: 12/12.

### Limber Quispe — SPF, DKIM y DMARC
Publicamos y verificamos con dig: SPF v=spf1 mx -all (solo el MX envía), la clave pública DKIM en mail._domainkey y DMARC con p=quarantine y alineación estricta. En el destino, SPF dio pass para 10.10.10.25 y fail para 10.10.10.26. Al cambiar el asunto de una copia, DKIM respondió Bad signature. OpenDKIM necesita Nameservers 10.10.10.4 porque usa su propio resolvedor.
### Juniors Parapaino — Thunderbird y cabeceras
Para Thunderbird agregamos mail.horizonte.internal al archivo hosts (es el nombre del certificado) e importamos la CA del laboratorio. Las cuentas usan IMAP 9993 con SSL/TLS y SMTP 9587 con STARTTLS, con el usuario sin el dominio. Secretaría envió Inscripciones 2027 a la directora. Las cabeceras se leen de abajo hacia arriba: with ESMTPSA significa TLS y autenticación, DKIM-Signature d=horizonte.internal es la firma y Authentication-Results muestra dkim=pass.
