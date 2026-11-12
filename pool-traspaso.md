# Limpieza de Mesa — traspaso a Claude Code

Resumen de una conversación en claude.ai (11/11/2026) para continuarla en Claude Code.
Usuario: Lucas, Argentina. Hablarle en español rioplatense informal. No es programador:
los pasos tienen que ser detallados, uno por vez, con verificación en cada etapa y sin jerga.

## Qué es

App web de pool en un solo archivo HTML (`limpieza-de-mesa.html`, va junto a este archivo).
Hoy está publicada como artifact de Claude: https://claude.ai/artifact/U5mHku4P59PzrL1YsS8xJG
(compartida por enlace). Ahí el historial se guarda en la cuenta de Claude de Lucas o,
para visitantes sin sesión, solo en el navegador de cada uno.

## Qué hace hoy

**Pestaña Limpieza** (cronómetro para limpiar la mesa solo)
- Campo "¿Quién juega?" (nombre). Nombres iguales ignorando mayúsculas y espacios = misma persona.
- Dos modos: bola anunciada / bola no anunciada.
- Al parar: Fallos (+1 s c/u), Faltas (+5 s c/u), Cervezas encima.
  - Fallo: no anunciada = no entronerar ninguna; anunciada = no meter la bola anunciada.
  - Falta: no tocar ninguna bola o meter la blanca.
- Comparación mejor / actual / peor con día y hora, por jugador y por modo.
- Ranking por modo (mejor marca de cada jugador) e historial completo.
- Recuadro "Cómo se juega" arriba con las reglas.

**Pestaña Partidas** (marcador entre dos jugadores)
- Bola 8, bola 9, bola 10 (suman mesas ganadas) y straight pool (suma puntos).
- Meta configurable ("se juega a N"), 0 = sin meta.
- Straight pool: falta normal −1 y blanca entronerada −5, ambos valores configurables y
  guardados en cada partida. Botón "Deshacer última jugada".
- "Entre amigos" (marcador por rival y juego), "Jugadores" (ganó/perdió por juego), historial.

**General**: descarga del historial en CSV; borrar con dos toques; tema claro y oscuro.

## Modelo de datos (en el HTML)

- Limpieza: `{id:"r…", kind:"run", player, mode:"anun"|"libre", endedAt, rawMs, misses, faults, beers, totalMs}`
  con `totalMs = rawMs + misses*1000 + faults*5000`.
- Partida: `{id:"m…", kind:"match", game:"b8"|"b9"|"b10"|"sp", a, b, sa, sb, target, startedAt, endedAt,
  rules:{foul,scratch}|null, fouls:{a:{n,s},b:{n,s}}|null}`.
- Guardado actual: `claude.use("db")` en `data/users/<id>` + copia en `localStorage`.
  En el artifact el archivo no lleva `<!doctype>`, `<html>`, `<head>` ni `<body>`: los agrega Claude al publicar.

## El objetivo pendiente

Lucas quiere un **historial conjunto** para él y 5 amigos, cada uno desde su celu y
**sin iniciar sesión**. No le preocupa que cualquiera con el enlace pueda cargar datos.
Dentro de Claude no se puede (un visitante sin sesión no escribe datos compartidos y la
página no puede llamar a servicios externos). Solución acordada:

- **Página**: el mismo HTML en GitHub Pages (Lucas tiene cuenta de GitHub).
- **Datos**: planilla de Google Sheets en su Drive + Apps Script publicado como app web.

## Plan por etapas

1. **Planilla + script** (pendiente, lo hace Lucas desde la compu). Pasos abajo.
2. Lucas pasa la dirección que termina en `/exec`.
3. Adaptar el HTML para GitHub Pages (ver "Cambios al HTML").
4. Pasos detallados para crear el repositorio, subir `index.html` y activar Pages.
5. Probar desde el celu, y recién ahí repartir el enlace.
6. Opcional: migrar lo ya cargado en el artifact (se baja con el botón de CSV).

Decisiones que Lucas todavía no contestó: si sube él el archivo a GitHub o lo hace Claude;
si migra lo cargado o arrancan de cero.

## Etapa 1: pasos para Lucas

1. Entrar a https://sheets.new y ponerle de nombre **Historial Pool**.
2. Menú **Extensiones > Apps Script**. Borrar todo lo que hay en el cuadro de código.
3. Pegar el script de abajo y guardar (Ctrl+S).
4. **Implementar > Nueva implementación**. En la ruedita de "Seleccionar tipo" elegir **App web**.
5. Descripción: `pool`. Ejecutar como: **Yo**. Quién tiene acceso: **Cualquier usuario**.
6. **Implementar**, autorizar el acceso (si aparece "Google no verificó esta app":
   Configuración avanzada > Ir a (proyecto) > Permitir).
7. Copiar la dirección de "App web" que termina en `/exec`.
8. Verificación: abrir esa dirección en una pestaña nueva. Tiene que mostrar
   `{"ok":true,"items":[]}` y en la planilla tiene que aparecer la pestaña **Historial**
   con seis títulos.

Nota: los nombres de menús salen de la guía oficial
(https://developers.google.com/apps-script/guides/web?hl=es-419); el texto exacto de los
campos del paso 5 y los carteles del paso 6 no está confirmado y puede variar un poco.
Si después se edita el script, hay que crear una nueva versión de la implementación para
que el cambio valga en la dirección `/exec`.

### Script (Apps Script)

```javascript
const HOJA = 'Historial';

function hoja_() {
  const ss = SpreadsheetApp.getActiveSpreadsheet();
  let h = ss.getSheetByName(HOJA);
  if (!h) {
    h = ss.insertSheet(HOJA);
    h.appendRow(['id', 'tipo', 'fecha', 'quien', 'detalle', 'datos']);
    h.setFrozenRows(1);
  }
  return h;
}

function salida_(obj) {
  return ContentService.createTextOutput(JSON.stringify(obj))
    .setMimeType(ContentService.MimeType.JSON);
}

function texto_(v, max) {
  const t = String(v || '').slice(0, max);
  return /^[=+\-@]/.test(t) ? "'" + t : t;
}

function doGet() {
  const filas = hoja_().getDataRange().getValues().slice(1);
  const items = [];
  filas.forEach(function (f) {
    if (!f[0] || !f[5]) return;
    try {
      const it = JSON.parse(f[5]);
      it.id = String(f[0]);
      items.push(it);
    } catch (err) {}
  });
  return salida_({ ok: true, items: items });
}

function doPost(e) {
  const lock = LockService.getScriptLock();
  lock.waitLock(20000);
  try {
    const p = JSON.parse(e.postData.contents);
    const h = hoja_();
    const ids = h.getRange(1, 1, h.getLastRow(), 1).getValues()
      .map(function (r) { return String(r[0]); });

    if (p.action === 'add') {
      const it = p.item || {};
      const id = String(it.id || '');
      const datos = JSON.stringify(it);
      if (!/^[A-Za-z0-9_-]{5,60}$/.test(id) || datos.length > 5000) {
        return salida_({ ok: false, error: 'datos invalidos' });
      }
      if (ids.indexOf(id) === -1) {
        h.appendRow([id, texto_(it.kind, 20), new Date(Number(it.endedAt) || Date.now()),
          texto_(p.who, 60), texto_(p.summary, 200), datos]);
      }
      return salida_({ ok: true });
    }

    if (p.action === 'delete') {
      const i = ids.indexOf(String(p.id));
      if (i > 0) h.deleteRow(i + 1);
      return salida_({ ok: true });
    }

    return salida_({ ok: false, error: 'accion desconocida' });
  } catch (err) {
    return salida_({ ok: false, error: String(err) });
  } finally {
    lock.releaseLock();
  }
}
```

### Contrato del script

- `GET /exec` → `{ok:true, items:[...]}` con todas las limpiezas y partidas (el objeto guardado, con `id`).
- `POST /exec` con cuerpo de texto JSON (mandar `Content-Type: text/plain` para evitar el preflight):
  - `{action:"add", item:{...}, who:"nombre(s)", summary:"texto legible"}` (repetir el mismo `id` no duplica).
  - `{action:"delete", id:"..."}`.
- Sin probar todavía contra una implementación real: verificar GET y POST desde el navegador antes de dar la etapa por cerrada.

## Cambios al HTML para GitHub Pages (etapa 3)

- Envolverlo como documento completo: `<!doctype html>`, `<html lang="es">`, `<head>` con
  charset, viewport, `<title>` y el `<style>`; contenido en `<body>`. Guardarlo como `index.html`.
- Reemplazar el guardado con `claude.use("db")` / `claude.use("user")` por `fetch` a la dirección `/exec`:
  cargar con GET al abrir y cada tanto; al guardar, POST `add`; al borrar, POST `delete`.
  Mantener `localStorage` como copia y la cola de pendientes (`unsynced`) para cuando no hay señal.
- Quitar `claude.use("downloads")`: fuera de Claude la descarga de CSV se hace con un
  enlace `blob:` y atributo `download`.
- El historial pasa a ser de todos: en Limpieza ya se filtra por nombre y modo; en Partidas se ve todo.
- Sacar el botón "Pasar las limpiezas sin nombre" o dejarlo solo para datos locales.
- Borrar: cualquiera con el enlace puede borrar filas ajenas. Lucas lo acepta, pero conviene avisarle.
