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

### Lizeth Rodríguez — Dovecot IMAPS y envío autenticado
Dovecot quedó con ssl = required y el puerto 143 cerrado, así que el correo solo se lee por IMAPS (993) con un certificado de la CA del laboratorio (Verification: OK). Postfix valida usuarios con Dovecot (SASL). En el 587, sin TLS ni siquiera se ofrece la autenticación; sin usuario se rechaza con 553; con TLS y usuario se acepta (235); y nadie puede enviar con la dirección de otra persona. También coordiné el repositorio.

### Jacqueline Caceres — Postfix y relay 
La configuración inicial funcionaba, pero era un relay abierto: aceptó con 250 un correo de un dominio ajeno a otro ajeno (5/12). mydestination dice qué dominios se entregan aquí y mynetworks quién puede usar el servidor para enviar a otros. Lo dejamos solo en 127.0.0.0/8 y la PC de secretaría, y la misma prueba respondió 554 Relay access denied. También quitamos la versión del saludo, desactivamos VRFY y el alias inscripciones llega a dos buzones: 12/12.

### Jacqueline C�ceres � Postfix y relay
La configuraci�n inicial funcionaba, pero era un relay abierto: acept� con 250 un correo de un dominio ajeno a otro ajeno (5/12). mydestination dice qu� dominios se entregan aqu� y mynetworks qui�n puede usar el servidor para enviar a otros. Lo dejamos solo en 127.0.0.0/8 y la PC de secretar�a, y la misma prueba respondi� 554 Relay access denied. Tambi�n quitamos la versi�n del saludo, desactivamos VRFY y el alias inscripciones llega a dos buzones: 12/12.
