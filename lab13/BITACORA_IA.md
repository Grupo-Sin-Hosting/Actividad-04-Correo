# Bitácora de uso de IA — Laboratorio 13 (correo seguro)

Estudiante: Lizeth Rodríguez · Entorno: `red_dns` con `mail`, `mail2`, ns1 y cliente; Dovecot, Postfix (587),
OpenDKIM y policyd-spf; Thunderbird en Windows · Fecha: 26/10/2026

Reglas que respeté:
- La clave privada DKIM (`/etc/opendkim/keys/mail.private`) y la de la CA de correo (`compartido/ca-correo.key`) no se
  mostraron ni se suben al repositorio. Solo se publicó la clave pública DKIM en el DNS.
- Solo buzones de prueba del laboratorio; ningún correo real.
- No se desactivó el TLS ni la autenticación para "que funcione": el 143 quedó cerrado y el 587 exige TLS y usuario.

| # | Qué pregunté | Qué respondió la IA | Cómo lo verifiqué | Qué aprendí |
|---|---|---|---|---|
| 1 | Qué valor de `ssl` exige TLS en Dovecot. | `yes` permite conexiones sin cifrar; `required` lo exige. Cerrar el 143 con `port = 0`. | `ss` mostró solo 993; `curl imap://…` → sin respuesta; `openssl` → `Verification: OK`. | Diferencia entre ofrecer y exigir TLS. |
| 2 | Cómo conecta Postfix con Dovecot para validar contraseñas. | `service auth` con un `unix_listener` en `/var/spool/postfix/private/auth` (0660, postfix). | `235 2.7.0 Authentication successful` por el 587. | Qué es SASL. |
| 3 | Reglas del puerto 587. | `encrypt`, `permit_sasl_authenticated,reject` y `reject_sender_login_mismatch`. | Las 4 pruebas: `not logged in`, `did not advertise authentication`, `235 + queued`, `not owned by user secretaria`. | Envío autenticado y anti-suplantación interna. |
| 4 | Política DMARC de cuarentena con informes y alineación estricta. | `p=quarantine; rua=mailto:…; adkim=s; aspf=s`. | `dig _dmarc… TXT` y verificador ✅ DMARC con p=quarantine. | Qué significa la alineación. |
| 5 | Por qué OpenDKIM podría decir "key not found" si `dig` sí la encuentra. | OpenDKIM no usa el mismo camino que `dig`; se le indica `Nameservers 10.10.10.4`. | `dkim=pass header.d=horizonte.internal` en el destino. | Cada programa puede resolver DNS distinto. |
| 6 | Por qué el verificador dio 10/15. | Las 5 pruebas fallidas usan `curl` y el cliente no lo tenía; no era la configuración. | `curl --version` y luego **15/15**. | Leer el script antes de tocar la configuración. |
| 7 | Cómo explicar las cabeceras `Received-SPF`, `DKIM-Signature` y `Authentication-Results`. | Resultado de SPF para la IP; firma con `d=`, `s=`, `a=`, `c=`; resumen del receptor. | Cabeceras reales del correo "Verificador 1831" leído por IMAPS. | Análisis de cabeceras. |
| 8 | Cómo demostrar que DKIM protege la integridad. | Cambiar una cabecera firmada en una copia y volver a verificar. | Original: válida; con el asunto cambiado: `dkim_eom(): Bad signature`. | La firma cubre cabeceras y cuerpo. |
| 9 | Cómo conectar Thunderbird desde Windows a los contenedores. | Docker publica 993→9993 y 587→9587; entrada en `hosts` para el nombre del certificado; importar la CA. | Dos cuentas (secretaria y rosario.vaca) enviando y recibiendo. | El nombre del servidor debe coincidir con el certificado. |

## Conclusión
El correo del colegio quedó cifrado de punta a punta para los usuarios: se lee solo por IMAPS (993) y se envía solo
por 587 con TLS y usuario, y nadie puede usar la dirección de otra persona. Hacia fuera, el dominio publica SPF, DKIM y
DMARC: el destino confirmó `SPF Pass`, `dkim=pass` y la alineación con el `From:`, y una firma alterada se detecta.
Resultado: **15 de 15**.
