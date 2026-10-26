# Roles de la defensa — Actividad 04

| Integrante | GitHub | Rama | Parte | Debe poder explicar |
|---|---|---|---|---|
| Lucas Carrasco | @Lukitas2005 | `rol-dns-correo` | Entorno y DNS del correo | Nodos mail/mail2, MX, A y los dos PTR (Docker vs resolutor), por qué un servidor de correo necesita MX/A/PTR |
| Jacqueline Cáceres | @jacquelinesubia321-spec | `rol-postfix` | Postfix: del relay abierto al servidor seguro (Lab 12 B) | `mynetworks` vs `mydestination`, prueba de relay 250 → 554, banner, VRFY, alias inscripciones |
| José María Orihuela | — | `rol-smtp` | SMTP a mano, recorrido y TLS (Lab 12 C–E) | Códigos 220/250/354/221, sobre vs `From:`, `mail.log` (relay, delay, dsn, status, orig_to), STARTTLS oportunista |
| Lizeth Rodríguez | @lixinrodriguez1-eng | `rol-imaps-587` | Dovecot IMAPS y envío autenticado (Lab 13 B–C) + coordinación | `ssl = required`, 143 cerrado, SASL, las 4 pruebas del 587; repositorio |
| Juniors Parapaino | @ederjuniorparapaino | `rol-thunderbird` | Thunderbird con dos usuarios y análisis de cabeceras | Configuración 9993/9587, CA importada, correo secretaria → dirección, `Received`, `DKIM-Signature`, `Authentication-Results` |
| Limber Quispe | — | `rol-spf-dkim-dmarc` | SPF, DKIM y DMARC (Lab 13 D) | Registros con `dig`, SPF pass/fail (.25/.26), firma rota al alterar, alineación DMARC, `Nameservers` de OpenDKIM |

Cada integrante: crea su rama, agrega abajo un resumen de su parte con sus palabras y abre un Pull Request con `Closes #N`.

## Notas de cada integrante

### Lizeth Rodríguez — Dovecot IMAPS y envío autenticado
Dovecot quedó con `ssl = required` y el puerto 143 cerrado, así que el correo solo se lee por IMAPS (993) con un certificado de la CA del laboratorio (Verification: OK). Postfix valida usuarios con Dovecot (SASL). En el 587, sin TLS ni siquiera se ofrece la autenticación; sin usuario se rechaza con 553; con TLS y usuario se acepta (235); y nadie puede enviar con la dirección de otra persona. También coordiné el repositorio.

### Jacqueline Cáceres — Postfix y relay
La configuración inicial funcionaba, pero era un relay abierto: aceptó con 250 un correo de un dominio ajeno a otro ajeno (5/12). `mydestination` dice qué dominios se entregan aquí y `mynetworks` quién puede usar el servidor para enviar a otros. Lo dejamos solo en 127.0.0.0/8 y la PC de secretaría, y la misma prueba respondió `554 Relay access denied`. También quitamos la versión del saludo, desactivamos VRFY y el alias inscripciones llega a dos buzones: 12/12.
