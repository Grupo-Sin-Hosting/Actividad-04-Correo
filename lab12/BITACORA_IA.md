# Bitácora de uso de IA — Laboratorio 12 (correo con Postfix)

Estudiante: Lizeth Rodríguez · Entorno: `red_dns` con `mail` (10.10.10.25) y `mail2` (10.10.10.26), Ubuntu 24.04,
Postfix 3.8 · Fecha: 25–26/10/2026

Reglas que respeté (sección 7 de la guía):
- **No** abrí `mynetworks` "para que funcione": la configuración final solo permite `127.0.0.0/8` y la PC de
  secretaría (`10.10.10.50/32`), y la prueba de relay da `554 Relay access denied`.
- No usé correos ni registros de personas reales: solo los buzones de prueba del laboratorio.
- No pedí la configuración completa sin entenderla: cada parámetro está explicado en la hoja de trabajo, y cada
  cambio se comprobó con el verificador.

| # | Qué pregunté | Qué respondió la IA | Cómo lo verifiqué | Qué aprendí |
|---|---|---|---|---|
| 1 | Cómo agregar `mail` y `mail2` a `red_dns`. | Dos nodos en 10.10.10.25 y .26 con `dns: 10.10.10.4` para que usen el resolutor del colegio. | `docker compose ps` con 7 nodos; `dig MX` desde el cliente → `10 mail.horizonte.internal.` | El DNS de un contenedor se puede dirigir a un resolutor propio. |
| 2 | Por qué falló `instalar_resolutor.sh` con `invalid user 'unbound'`. | En el Lab 8 Unbound se instaló a mano antes del script: apagar `named` e instalar `unbound`. | Después, `unbound-checkconf: no errors` y `active`. | Revisar los requisitos previos de cada instalador. |
| 3 | Por qué los dos PTR de 10.10.10.25 son distintos (prompt de la hoja). | El cliente usa el DNS interno de Docker (127.0.0.11), que responde las inversas de sus contenedores. | `-x` → `red_dns-mail-1.red_dns_aula.`; `@10.10.10.4 -x` → `mail.horizonte.internal.` | Saber a qué DNS se está preguntando. |
| 4 | Diferencia entre `mydestination`, `mynetworks` y `smtpd_relay_restrictions` (prompt 1). | Destino final, quién puede hacer relay, y la regla que lo aplica (`reject_unauth_destination`). | Relay: `250 queued` con 0.0.0.0/0 y `554 Relay access denied` con la red cerrada. | Qué es un relay abierto y por qué es peligroso. |
| 5 | Por qué seguía 5/12 después de editar. | Comprobar el archivo dentro del contenedor y si el instalador se volvió a ejecutar. | `grep` mostró los valores nuevos; faltaba reinstalar (se había pegado mal el comando: "horizontedocker…"). Reinstalado → 12/12. | Comprobar en el servidor, no solo en el editor. |
| 6 | Explicar cada campo de una línea de `mail.log` (prompt 2). | `relay=`, `delay=`/`delays=`, `dsn=` y `status=`; `orig_to=` en los alias. | Seguí los ID 6F937C72C (a familia) y 51183C272 (inscripciones → 2 buzones). | Seguir un correo con su ID de cola. |
| 7 | Por qué `apt-get update` daba `403 Forbidden`. | La red bloquea los repositorios por HTTP; usar un espejo por HTTPS. | Con `mirrors.edge.kernel.org` → `Fetched 33.6 MB` en los tres nodos. | Distinguir un bloqueo de red de un error de configuración. |
| 8 | Cómo hacer la conversación SMTP con `nc` sin pelear con las comillas de PowerShell. | Entrar al contenedor y ejecutar ahí el bloque con `sleep` entre comandos. | Códigos 220, 250, 354, `250 2.0.0 Ok: queued as E26568BB3` y 221. | PowerShell 5.1 altera las comillas al pasarlas a programas. |
| 9 | Por qué se acepta un certificado autofirmado entre servidores de correo. | El TLS entre MTA es oportunista; mejor cifrar que no; DANE/MTA-STS para exigirlo. | `TLSv1.3`, `TLS_AES_256_GCM_SHA384`, issuer = subject. | Límites del STARTTLS oportunista. |

## Conclusión
El servidor que "funcionaba" era un relay abierto: el verificador lo dejó en evidencia (5/12) y la prueba con swaks
lo confirmó (`250 queued` para dos dominios ajenos). Con los seis cambios quedó en 12/12, con relay rechazado,
saludo sin versión, sin VRFY, entrega en Maildir, el alias inscripciones repartiendo a dos personas y envío a otro
dominio por su MX. La IA me ayudó a entender los parámetros y a diagnosticar los tropiezos del entorno; cada
respuesta la verifiqué con `postconf`, `swaks`, `mail.log` o el verificador.
