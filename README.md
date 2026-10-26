# Actividad 04 — Correo institucional con Postfix y Dovecot

**Tecnología Web I (SIS-0150) · Universidad Privada Domingo Savio (UPDS) · Semestre 2/2026**
Docente: M.Sc. Ing. Jimmy Nataniel Requena Llorentty · Grupo **Sin Hosting**:
Jacqueline Cáceres · Lucas Carrasco · José María Orihuela · Juniors Parapaino · Limber Quispe · Lizeth Rodríguez

> Pon en marcha el correo institucional del dominio de laboratorio y demuestra el envío y la recepción entre buzones
> con conexiones cifradas.

| Laboratorio | Qué se hizo | Verificador |
|---|---|---|
| [Lab 12](lab12/HOJA_TRABAJO.md) | Postfix para `horizonte.internal`: de relay abierto (5/12) a configuración segura, Maildir, alias, envío por MX a `familia.internal`, SMTP a mano, `mail.log`, STARTTLS | **12 de 12** |
| [Lab 13](lab13/HOJA_TRABAJO.md) | Dovecot IMAPS (993), envío autenticado (587), SPF + DKIM (OpenDKIM) + DMARC publicados y verificados con `dig`, análisis de cabeceras, Thunderbird con dos usuarios | **15 de 15** |

- **Informe técnico grupal (APA 7):** [informe/](informe/)
- **Presentación:** [presentacion/](presentacion/)
- **Bitácoras de IA:** [lab12](lab12/BITACORA_IA.md) · [lab13](lab13/BITACORA_IA.md) (la consolidada está en el informe)
- **Roles de la defensa:** [ROLES.md](ROLES.md)

## Red de laboratorio (`red_dns`)

| Nodo | IP | Función |
|---|---|---|
| ns1 / ns2 | 10.10.10.2 / .3 | BIND 9.18: zona `horizonte.internal` (MX, A, PTR, SPF, DKIM, DMARC) con vistas y DNSSEC |
| resolutor | 10.10.10.4 | Unbound (Lab 8) + DNS simulado de `familia.internal` |
| cliente | 10.10.10.50 | PC de secretaría: `swaks`, `nc`, `curl`, verificadores |
| mail | 10.10.10.25 | `mail.horizonte.internal`: Postfix (25, 587), Dovecot (993), OpenDKIM (firma). Publicado en la PC como 9993/9587 |
| mail2 | 10.10.10.26 | `mail.familia.internal`: Postfix, Dovecot, OpenDKIM (verifica) y policyd-spf |

## Cómo reproducir

```powershell
cd red_dns
docker compose up -d --build
docker compose exec ns1 bash /lab/lab07/instalar_ns1.sh
docker compose exec ns2 bash /lab/lab07/instalar_ns2.sh
docker compose exec ns1 bash /lab/lab09/instalar_vistas_dnssec.sh
docker compose exec resolutor bash -c "systemctl disable --now named; apt-get update && apt-get install -y unbound unbound-anchor dns-root-data"
docker compose exec resolutor bash /lab/lab08/instalar_resolutor.sh
docker compose exec resolutor bash /lab/lab12/instalar_dns_correo.sh
docker compose exec mail2 bash /lab/lab12/instalar_correo.sh familia
docker compose exec mail  bash /lab/lab12/instalar_correo.sh horizonte
docker compose exec mail  bash /lab/lab12/verificar_lab12.sh          # 12 de 12
docker compose exec mail  bash /lab/lab13/instalar_correo_seguro.sh horizonte
docker compose exec ns1   bash /lab/lab13/publicar_dns_correo.sh
docker compose exec mail2 bash /lab/lab13/instalar_correo_seguro.sh familia
docker compose exec cliente bash /lab/lab13/verificar_lab13.sh        # 15 de 15 (el cliente necesita curl)
```

> Si `apt-get update` devuelve `403 Forbidden` (red que bloquea los repositorios por HTTP), cambia las fuentes a un
> espejo HTTPS: `sed -i -E 's#http://(archive|security).ubuntu.com/ubuntu#https://mirrors.edge.kernel.org/ubuntu#g' /etc/apt/sources.list.d/ubuntu.sources`

## Seguridad
- **Ninguna clave privada** está en el repositorio: ni la DKIM (vive en `/etc/opendkim/keys` del nodo `mail`), ni la
  de la CA de correo (`lab13/compartido/ca-correo.key`), ni DNSSEC ni TSIG. Ver `.gitignore`.
- `mynetworks` solo incluye `127.0.0.0/8` y la PC de secretaría; el relay desde fuera da `554 Relay access denied`.
- Contraseñas: solo las de los buzones de prueba del laboratorio.
