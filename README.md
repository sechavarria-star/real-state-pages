# Panel Real State — apprealstate.giwa-ia.com

Panel comercial multi-cliente del asistente de WhatsApp para desarrolladoras
inmobiliarias: resumen y embudo, **agenda de visitas por asesor** (reprogramar,
marcar, bloqueos, horarios, visitas manuales) y conversaciones (tomar / devolver
al asistente). Sitio estático; la API es la Edge Function `realstate-panel`.

```
apprealstate.giwa-ia.com  →  GitHub Pages (este index.html)
                                ↓ POST text/plain {"accion": …, "token": …}
                           Edge Function realstate-panel (proyecto ofhiogczijhzkuqdjwvz)
                                ↓ Postgres como realstate_panel
                           schema realstate: lee tablas + ejecuta SOLO realstate.panel_*
```

Código de la función y SQL: `../real-state/supabase/` (`functions/realstate-panel/`,
`sql/04-panel.sql`).

## Acceso

Usuarios por persona (`realstate.panel_usuarios`), rol `admin` (todo su cliente) o
`asesor` (ve todo, edita solo su agenda). Clave con bcrypt; 5 intentos fallidos →
15 min de bloqueo. El login devuelve un token firmado (HMAC) que dura 12 h y viaja
en el body. Cada consulta se filtra por la cuenta del token.

Alta, cambio de clave y baja (la clave se tipea, no queda en ningún archivo):

```bash
python3 ../real-state/scripts/usuario_panel.py alta argencons ana@empresa.com "Ana Pérez" admin
python3 ../real-state/scripts/usuario_panel.py alta argencons juan@empresa.com "Juan Gómez" asesor --asesor "Asesor Demo Belgrano"
python3 ../real-state/scripts/usuario_panel.py clave ana@empresa.com
python3 ../real-state/scripts/usuario_panel.py lista argencons
```

## Acceso a la base

La función se conecta como `realstate_panel`: lee las tablas de `realstate` (salvo
`panel_usuarios`), **no puede escribir ninguna tabla directo** y solo ejecuta las
funciones `realstate.panel_*`, que validan cuenta y rol. No ve otros schemas.

Secrets de la función (no hay copia local): `REALSTATE_PANEL_DB_PASSWORD` y
`REALSTATE_PANEL_TOKEN_SECRET`. Rotar la contraseña del rol:
`alter role realstate_panel password '<nueva>'` + `supabase secrets set
REALSTATE_PANEL_DB_PASSWORD=<nueva> --project-ref ofhiogczijhzkuqdjwvz`. Rotar el
secreto de tokens cierra todas las sesiones abiertas.

## CORS

La función solo responde a `https://apprealstate.giwa-ia.com` (constante `ORIGENES`).
Para probar en local, sumar **temporalmente** `http://localhost:8080`, deployar y
sacarlo al terminar.

## Deploy

- **Front:** `git push` a `main` → GitHub Pages. DNS en Squarespace:
  `CNAME apprealstate → sechavarria-star.github.io`.
- **Función** (desde `../real-state`):

```bash
supabase functions deploy realstate-panel --project-ref ofhiogczijhzkuqdjwvz --no-verify-jwt --use-api
```

## Reglas

- Repo público: nada de datos ni secretos acá.
- Todo lo que viene de la base pasa por `esc()` antes de ir al HTML.
- Antes de publicar: extraer el `<script>` y `node --check`.
- Si el panel reprograma o cancela una visita, **no se avisa al lead automáticamente**
  (fase 3: requiere plantilla aprobada fuera de las 24 h).
