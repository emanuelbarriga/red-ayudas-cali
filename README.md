# 🚨 Red de Ayudas Cali

Mapa operativo en tiempo real para coordinar puntos de ayuda durante emergencias en Cali. Los voluntarios reportan ubicaciones desde su celular; los coordinadores gestionan prioridades y despachos desde una tabla central. Sin cuentas ni contraseñas para el público.

**App en vivo:** https://red-ayudas-cali.web.app

---

## Cómo se usa (en 30 segundos)

1. Abre la app desde el celular — se pide **nombre y teléfono una sola vez por dispositivo**.
2. Mueve el mapa para ubicar el punto con el pin central 📍 y toca **➕ Reportar**.
3. Llena el formulario (nombre del lugar, coordinador, prioridad, foto opcional) y guarda.
4. El punto aparece al instante en el mapa **y en la tabla de coordinación** de todo el equipo.

## Acceso por roles

| Rol | Acceso | Qué puede hacer |
|-----|--------|-----------------|
| **Público / Voluntarios** | Sin clave — `https://red-ayudas-cali.web.app` | Ver el mapa, reportar puntos |
| **Solo lectura** | Clave: `ver2026` — `?pass=ver2026` | Ver la tabla completa, sin editar |
| **Editor / Coordinador** | Clave: `admin2026` — `?pass=admin2026` | Cambiar estados, ocultar puntos, cambiar rol |

> ⚠️ **Las claves viven en el código del cliente.** Son una separación de *interfaz* (velo de UI), no seguridad real: cualquiera con las herramientas de desarrollador puede leerlas. Si el proyecto escala, migrar a Firebase Auth + reglas por autenticación en el servidor.

---

## Funcionalidades

| Funcionalidad | Detalle |
|---------------|---------|
| **Tiempo real** | Firestore `onSnapshot`: los cambios aparecen en todos los dispositivos sin recargar |
| **Reporte con pin central** | Estilo Uber/Rappi: el punto se marca donde apunta el pin, no donde está el GPS |
| **Geocodificación inversa** | Autonombra el sector/barrio del punto vía Nominatim/OpenStreetMap (gratis) |
| **Compresión de imágenes** | Canvas reduce fotos a ~300 KB antes de subirlas a Storage |
| **Registro único por dispositivo** | Nombre y teléfono se piden una vez y quedan adjuntos a cada reporte |
| **Matriz operativa** | Tabla con prioridades ALTA→MEDIA→BAJA, estados y teléfono del reportero |
| **Cambio de estado en vivo** | ABIERTO → EN_VERIFICACION → ATENDIENDO → CERRADO |
| **Borrado suave** | "Borrar" solo oculta el punto (`eliminado: true`); nada se elimina de verdad |
| **Sesión persistente** | La clave de coordinador queda guardada en el dispositivo |
| **Responsive móvil** | Tabla como tarjetas apiladas, mapa a pantalla completa, sin zoom automático de iOS |
| **Actualización sin borrar caché** | `Cache-Control: no-cache` en `index.html` |

---

## Modelo de datos

### Colección `puntos_cali`

| Campo | Tipo | Descripción |
|-------|------|-------------|
| `nombrePunto` | string (1–150) | Nombre del lugar / sector |
| `coordinador` | string (1–100) | Coordinador en sitio (nombre y teléfono) |
| `tecnicoPresente` | `"SI"` \| `"NO"` | ¿Hay personal técnico/AV en sitio? |
| `prioridad` | `ALTA` \| `MEDIA` \| `BAJA` | Urgencia del punto (opcional en docs viejos) |
| `necesidades` | string | Necesidades específicas |
| `situacion` | string | Condiciones, riesgos, acceso |
| `fotoUrl` | string \| null | URL de la foto en Storage |
| `lat` / `lng` | number | Coordenadas **dentro del bounding box de Cali** |
| `estado` | `ABIERTO` \| `EN_VERIFICACION` \| `ATENDIENDO` \| `CERRADO` | Ciclo de vida del punto |
| `reporterNombre` / `reporterTelefono` | string \| null | Quién reportó (registro del dispositivo) |
| `timestamp` | timestamp | Fecha/hora del reporte |
| `eliminado` | boolean (opcional) | `true` = oculto (borrado suave) |

### Estados del punto

```
ABIERTO → EN_VERIFICACION → ATENDIENDO → CERRADO
🔴 Pendiente      🟡 En verificación    🔵 Atendiendo    🟢 Resuelto
```

---

## Seguridad (reglas del servidor)

Las reglas se despliegan como archivos versionados (`firestore.rules`, `storage.rules`) — no dependen de la consola.

### Firestore — `firestore.rules`

- **Validación completa en create y update** del esquema + bounding box geográfico de Cali
  (Lat 3.20–3.55, Lng −76.65 a −76.45).
- **Los reportes solo nacen `ABIERTO`**.
- **`delete` bloqueado al 100%** — nada se borra de Firestore.
- **`prioridad` validada** solo si existe (no rompe documentos antiguos).

### Storage — `storage.rules`

- Solo imágenes (`image/*`) de **máximo 5 MB**.
- **Sin sobrescritura ni borrado** de archivos.

### Blindajes del cliente

- **XSS**: todo texto de Firestore se escapa con `escapeHTML` antes de inyectarlo en popups.
- **Fotos**: solo se aceptan URLs `http(s)` como imagen; los nombres de archivo se sanitizan.
- **Coordenadas**: validación cliente del bounding box antes de enviar, con mensaje claro.

---

## Despliegue

### Requisitos

- Node.js + `npx` (se descarga firebase-tools automáticamente)
- Acceso al proyecto Firebase `red-ayudas-cali` (`firebase login`)

### Comandos

```bash
# Desplegar reglas + hosting (sin storage:rules — falla en firebase-tools 15.x)
npx firebase-tools deploy --only firestore:rules,storage,hosting

# Solo hosting
npx firebase-tools deploy --only hosting

# Emular localmente
npx firebase-tools emulators:start --only hosting
```

> **Gotcha conocido:** `deploy --only storage:rules` falla con *"Could not find rules for the following storage targets: rules"* si no hay target definido. Usa `storage` sin sufijo (despliega al bucket por defecto).

### Actualizaciones sin fricción

`firebase.json` envía `Cache-Control: no-cache` para `index.html`, así los celulares toman la versión nueva al abrir la app — sin pedir "borrar caché".

---

## Estructura del proyecto

```
.
├── firebase.json          # Configuración de hosting, reglas y headers
├── firestore.rules        # Reglas de Firestore (esquema + bounding box + no delete)
├── storage.rules          # Reglas de Storage (imágenes ≤ 5 MB, sin borrado)
├── public/
│   ├── index.html         # Toda la app (mapa, formulario, tabla, lógica)
│   └── 404.html
```

**Sin framework, sin build**: una sola página HTML con Leaflet + SDKs de Firebase por CDN. Es deliberado: se despliega con un comando y funciona en celulares de gama baja.

---

## Notas operativas y pendientes

| Tema | Estado |
|------|--------|
| **Anti-spam** | ⚠️ Pendiente: sin login, cualquiera con la URL puede crear puntos ilimitados (las reglas limitan *qué*, no *cuánto*) |
| **Compatibilidad de datos viejos** | ⚠️ Los documentos anteriores al esquema pueden fallar en updates de estado |
| **Seguridad real por roles** | ⚠️ Las claves son velo de UI; Firebase Auth + reglas sería el siguiente paso |
| **Geocodificación** | Nominatim es gratuito con límites (1 req/s); suficiente para este volumen |
| **Colaboradores del repo** | Invitar a Daniela/Jhina con acceso al repo GitHub si deben editar código |

---

## Roadmap sugerido

- [ ] Límite anti-spam (p. ej., tope de creaciones por dispositivo/día)
- [ ] Firebase Auth anónimo + reglas reales por autenticación
- [ ] Migrar claves de coordinador fuera del código (Firebase Remote Config o Functions)
- [ ] PWA / Service Worker para actualización automática sin recargar
- [ ] Exportar la tabla operativa (CSV) para reportes a autoridades
