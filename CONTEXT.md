# CONTEXT.md — Transferencia de sesión Claude Code
> Este archivo fue generado para transferir el estado completo de la sesión de desarrollo a una nueva instancia de Claude Code. Está escrito para otra IA, no para un humano.

---

## 1. Resumen ejecutivo

**Proyecto:** fly_vision.cl — sitio web estático para un servicio de grabación FPV con DJI Avata 2 en Chile.  
**Repositorio:** `MarinF-Lab/flyvision` (GitHub Pages, dominio personalizado `fly-vision.cl` vía CNAME).  
**Objetivo general:** Sitio público de presentación del servicio + panel privado de administración para gestionar pedidos y editar planes/precios.  
**Estado actual:** Completamente funcional y en producción. PR #2 mergeado a `main`. Cambios desplegados.  
**Nivel de avance:** ~95% del alcance definido en esta sesión. No hay errores conocidos ni trabajo pendiente activo.

---

## 2. Objetivo de esta sesión

**Se pretendía lograr:**
1. Agregar un link de Instagram en la imagen principal del drone en `index.html`.
2. Crear una página privada de administración (`admin.html`) separada del sitio público.
3. Que el admin pueda ver pedidos de clientes y editar precios/características de los planes.
4. Que los datos sean compartidos en tiempo real entre todos los dispositivos (no localStorage).
5. Agregar botones "Aceptar" y "Terminar" en cada tarjeta de pedido del panel admin.

**Se alcanzó todo:**
- Link Instagram en imagen del drone (abre en nueva pestaña, `target="_blank"`).
- `admin.html` como página independiente, protegida con contraseña, sin link desde el sitio público.
- Firebase Firestore como backend en tiempo real: colecciones `pedidos` y `planes`.
- Botones Aceptar / Terminar con estados persistidos en Firestore.
- PR #2 creado y mergeado. Todo en producción.

**Quedó pendiente:** Nada en el scope de esta sesión. El usuario puede pedir nuevas features.

---

## 3. Estado actual del código

### Componentes implementados y completos
- `index.html` — sitio público completo con Firebase (real-time planes + guardar pedidos)
- `admin.html` — panel privado completo con login, pedidos con acciones, editor de planes
- `CNAME` — apunta a `fly-vision.cl`
- `imagen.jpg` — imagen del drone (existía desde antes, no fue modificada)

### Componentes que NO deben modificarse sin análisis
- La configuración de Firebase (apiKey, projectId, etc.) — es el proyecto real del usuario.
- La contraseña de admin (`fly2026`) — fue establecida por el usuario.
- El número de WhatsApp (`+56935476134`) y el handle de Instagram (`fly_vision.cl`).
- Las reglas de Firestore en Firebase Console (el usuario las configuró manualmente, no están en el repo).

### Funcionalidades críticas
- `type="module"` en los scripts es **obligatorio** para Firebase ES modules. No remover.
- Las funciones `window.setStatus`, `window.deleteOrder`, `window.savePlan` están expuestas globalmente a propósito (necesario para `onclick` en HTML generado dinámicamente dentro de módulos ES).
- `onSnapshot` en `index.html` hace que el catálogo de planes se actualice en tiempo real cuando el admin edita precios — no reemplazar con `getDocs`.

---

## 4. Arquitectura

### Tecnologías
- **HTML/CSS/JS puro** — sin framework, sin build step, sin npm, sin bundler.
- **Firebase Firestore v12.15.0** vía CDN (ES modules desde `gstatic.com`).
- **Font Awesome 6.0.0-beta3** vía cdnjs (iconos).
- **Google Fonts** — Inter (pesos 300/400/600/700/800).
- **GitHub Pages** — hosting estático, rama `main`, dominio `fly-vision.cl`.

### Organización del proyecto
```
/home/user/flyvision/
├── index.html     # Sitio público (tabs: Inicio + Pedidos/Planes)
├── admin.html     # Panel admin privado (login + dashboard)
├── imagen.jpg     # Foto del drone DJI Avata 2
└── CNAME          # fly-vision.cl
```

### Patrones de diseño
- **Dark theme** con acento naranja `#fb5d00` y fondo `#1a1d23` / `#13151a`.
- **Glassmorphism suave** — cards con `background: #1e2128`, `border: 1px solid rgba(255,255,255,0.05)`.
- **Responsive mobile-first** — breakpoint principal `@media (min-width: 768px)` en index, `600px` en admin.
- **Toast notifications** para feedback de acciones (guardar, errores).
- **Real-time first** — preferir `onSnapshot` sobre `getDocs` para datos que se actualizan.

### Firebase config (IDÉNTICA en ambos archivos)
```javascript
const firebaseConfig = {
    apiKey: "AIzaSyDezL0tZRXITU-93pVKUIXy3nTOsUAtyWM",
    authDomain: "fly-vision-e13e6.firebaseapp.com",
    projectId: "fly-vision-e13e6",
    storageBucket: "fly-vision-e13e6.firebasestorage.app",
    messagingSenderId: "819877097310",
    appId: "1:819877097310:web:b228cb3c0020623cdd29cf"
};
```

### Firestore collections
| Colección | Descripción | Campos clave |
|-----------|-------------|--------------|
| `pedidos` | Pedidos de clientes | `nombre`, `plan`, `detalle`, `descripcion`, `status` (`'new'`/`'aceptado'`/`'terminado'`), `via` (`'WhatsApp'`/`'Instagram'`), `fecha` (serverTimestamp) |
| `planes`  | Planes y precios | `id`, `nombre`, `emoji`, `badge`, `precio`, `features[]`, `icons[]`, `orden` |

### Firestore Security Rules (aplicadas por el usuario en Firebase Console, NO en repo)
```javascript
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    match /pedidos/{id} { allow read, write: if true; }
    match /planes/{id}  { allow read, write: if true; }
  }
}
```
El usuario explícitamente no quiere autenticación ni restricciones.

---

## 5. Decisiones importantes tomadas

### Alternativas descartadas
| Alternativa descartada | Por qué se descartó |
|------------------------|---------------------|
| Panel admin como tab oculto en `index.html` | El usuario quería página separada, completamente privada |
| `localStorage` para persistencia | No funciona entre dispositivos distintos |
| Firebase Authentication | El usuario explícitamente pidió que no requiera autenticación |
| Seeding de planes en `index.html` | Solo el admin debe seedear; el index usa fallback a DEFAULT_PLANS si Firestore está vacío |
| `cycleStatus` (toggle de estado) | Reemplazado por `setStatus(docId, status)` para control explícito de transición de estado |

### Restricciones existentes
- El repo NO tiene CI/CD propio — GitHub Actions no está configurado. El deploy es automático via GitHub Pages desde `main`.
- No hay `.gitignore` — no hay archivos sensibles en el repo (la apiKey de Firebase es pública por diseño en proyectos web).
- No hay archivo de configuración de Firestore en el repo (`firestore.rules`, `firestore.indexes.json`) — todo se configuró manualmente en la consola.

---

## 6. Convenciones del proyecto

### Estilo de código
- **Sin frameworks**, sin TypeScript. JS vanilla puro con `async/await`.
- CSS inline dentro del mismo HTML (no archivos externos `.css`).
- Clases CSS en kebab-case: `order-card`, `btn-accion`, `filter-tabs`.
- Paleta: fondo `#1a1d23` / `#13151a`, cards `#1e2128` / `#23272f`, acento `#fb5d00` (naranja).
- Bordes sutiles: `rgba(255,255,255,0.05)` a `rgba(255,255,255,0.1)`.

### Convenciones de nombres
- IDs de Firestore para planes: `'basico'`, `'estandar'`, `'pro'`, `'premium'` (sin tildes en los IDs, tildes permitidas en los nombres).
- Variables: camelCase JS estándar.
- Funciones expuestas globalmente prefijadas implícitamente con dominio (`setStatus`, `deleteOrder`, `savePlan`).

### Buenas prácticas que deben mantenerse
- Siempre `type="module"` en scripts que usen Firebase.
- Siempre exponer funciones a `window.*` si se usan en `onclick` de HTML generado dinámicamente.
- No agregar tabs al sitio público ni links al panel admin — el usuario quiere que sea completamente oculto.
- `meta name="robots" content="noindex, nofollow"` debe permanecer en `admin.html`.

---

## 7. Estado funcional

### Funciona correctamente
- Sitio público (`index.html`): tabs, hero con link Instagram, catálogo de planes desde Firestore, modal de cotización, envío de pedido por WhatsApp e Instagram, guardado en Firestore.
- Panel admin (`admin.html`): login con `sessionStorage`, logout, listener real-time de pedidos, filtros (Todos/Nuevos/Aceptados/Terminados), stats row, botones Aceptar/Terminar/Eliminar, limpiar historial, editor de planes con Guardar.
- Seeding automático de planes en primer login admin.
- Actualización en tiempo real del catálogo público cuando el admin cambia precios.

### Requiere pruebas manuales (no hay suite de tests)
- El flujo completo del modal en mobile.
- La experiencia de copiar mensaje al portapapeles en iOS Safari (puede requerir gesto del usuario).

### Errores conocidos
- Ninguno conocido.

### No desarrollado
- Sistema de notificaciones al admin cuando llega un nuevo pedido.
- Historial de cambios de precios.
- Exportar pedidos a CSV.
- Autenticación real (Firebase Auth) — descartada explícitamente por el usuario.

---

## 8. Próximos pasos recomendados

### Prioridad alta (si el usuario los pide)
- Notificación push o email al admin cuando llega un nuevo pedido (requiere Firebase Cloud Messaging o un servicio externo como EmailJS).

### Prioridad media
- Campo de notas internas en el admin para cada pedido.
- Filtro por rango de fechas en el panel de pedidos.
- Indicador visual de "nuevo" (badge con número) en el topbar del admin.

### Mejoras futuras
- Galería de trabajos anteriores (videos/fotos) en el sitio público.
- Testimonios de clientes.
- Exportar pedidos a CSV desde el admin.
- Integración con Google Calendar para agendar sesiones de filmación.

---

## 9. Archivos relevantes

| Archivo | Rol | Prioridad para revisar |
|---------|-----|----------------------|
| `index.html` | Sitio público completo | Alta — contiene toda la lógica pública |
| `admin.html` | Panel admin completo | Alta — contiene toda la lógica admin |
| `CNAME` | Dominio personalizado | Baja — no modificar |
| `imagen.jpg` | Imagen del drone en el hero | Baja — asset estático |

### Ramas git
- `main` — producción, desplegado en GitHub Pages.
- `claude/avatar-profile-admin-panel-219lx3` — rama de trabajo de esta sesión (PR #2 ya mergeado, la rama sigue existiendo pero su trabajo está en main).

---

## 10. Contexto importante que no debe perderse

### Sobre el usuario y el negocio
- El propietario es Isaac Marin Flores (`isaac.marin.flores@daemretiro.cl`).
- Es un servicio de grabación FPV con DJI Avata 2 en Chile.
- Los clientes contactan al negocio vía WhatsApp (`+56 9 3547 6134`) o Instagram DM (`@fly_vision.cl`).
- El precio de los planes actualmente va de $30.000 a $120.000 CLP.
- Nombre público: `fly_vision.cl` / Dominio: `fly-vision.cl`.

### Criterios de diseño fijados
- **Dark mode exclusivo** — no hay modo claro, no se pidió.
- **Naranja `#fb5d00`** como color de marca principal.
- **Minimalista** — sin exceso de elementos, cards limpias.
- **Mobile-first** — la mayoría de usuarios son clientes que ven desde el celular.

### Restricciones del proyecto (no negociables)
- Sin autenticación real para el admin — contraseña hardcodeada `fly2026` + `sessionStorage` es suficiente para el usuario.
- Sin backend propio — solo Firebase Firestore como servicio.
- Sin build step — todo debe funcionar abriendo el HTML directamente o sirviéndolo estático.
- Sin link público al panel admin — el usuario navega a `/admin.html` manualmente.
- Reglas de Firestore: `allow read, write: if true` para `pedidos` y `planes`. El usuario aceptó el riesgo.

### Información técnica no obvia
- Firebase SDK v12.15.0 via `https://www.gstatic.com/firebasejs/12.15.0/` — no actualizar sin probar.
- Los pedidos en Firestore incluyen un campo `status` que por defecto es `'new'` (no `'nuevo'` — es inglés en el campo, español solo en la UI).
- El índice compuesto de Firestore para `query(collection(db, 'pedidos'), orderBy('fecha', 'desc'))` puede requerir ser creado en Firebase Console si aparece un error de índice faltante.
- La imagen del drone (`imagen.jpg`) sirve desde la raíz del repo — si se agrega un directorio `/assets/`, la ruta en el HTML debe actualizarse.

---

## 11. Instrucciones para el próximo Claude Code

**Estado real del proyecto:** Completamente funcional en producción. PR mergeado. No hay trabajo en curso ni deuda técnica conocida. La próxima acción debe ser una nueva feature solicitada por el usuario.

**Antes de modificar código:**
1. Lee `index.html` completo y `admin.html` completo — son los únicos archivos con lógica.
2. Verifica el estado del repo con `git log --oneline -5` y `git status`.
3. Si el usuario pide features nuevas, trabaja en la rama `claude/avatar-profile-admin-panel-219lx3` o crea una nueva rama descriptiva.
4. Nunca toques directamente `main` — siempre PR.

**No debes asumir:**
- Que hay un `package.json`, `node_modules`, o sistema de build — no existen.
- Que Firebase Authentication está configurado — no lo está ni debe estarlo.
- Que hay tests automatizados — no los hay.
- Que el usuario quiere comentarios en el código — no los agrega, no los pide.
- Que puedes cambiar la contraseña admin o el número de WhatsApp sin que el usuario lo pida explícitamente.

**Decisiones que debes respetar:**
- `type="module"` en todos los scripts que usen Firebase. Obligatorio.
- Funciones globales (`window.*`) para handlers en HTML dinámico. No hay alternativa sin refactorizar el enfoque de renderizado.
- `sessionStorage` para la sesión admin (se limpia al cerrar el navegador — comportamiento deseado).
- Reglas de Firestore públicas — el usuario las aceptó explícitamente.
- No hay link al admin desde el sitio público — debe mantenerse así.

**Cómo continuar el desarrollo consistentemente:**
- Mantén el mismo dark theme (`#1a1d23`, `#fb5d00`, cards `#1e2128`).
- Usa Font Awesome para iconos (ya importado).
- Cualquier nueva colección en Firestore debe añadirse también a las reglas de seguridad que el usuario tiene en Firebase Console.
- Si agregas nuevos campos a los documentos `pedidos` o `planes`, asegúrate de que tanto `index.html` como `admin.html` los manejen consistentemente.
- Para funciones que se llamen desde `onclick` en HTML generado, siempre usa `window.nombreFuncion = async (...) => { ... }`.
- Al hacer commit, mensajes en español para consistencia con los commits anteriores.
- Siempre pushear a la rama de trabajo y crear PR hacia `main` — nunca push directo a `main`.
