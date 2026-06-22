# Prompt — Funcionalidades de Mi Racha Espiritual (MRNT Class)

> Pegá este documento en Claude Design para reconstruir la lógica.
> Es un resumen exhaustivo de cómo opera la app real (`index.html`).
> Stack: una sola página `index.html` con React (Babel standalone) + Firebase
> (Auth, Realtime Database, Storage). No hay build: todo se sirve estático.

---

## 1. Visión general

**Mi Racha Espiritual** es una sala virtual sincronizada para una clase de
escuela sabática de jóvenes (Gteen / JA — Adventistas). Hay **un anfitrión (admin)**
que conduce el sábado, y **N invitados** que entran desde sus teléfonos.
Todos comparten una sola sala (SID = `maranata-YYYY-MM-DD`); la clase del
usuario (Gteen / JA) sólo cambia el badge/logo del avatar, no la sala.

El sábado está organizado según el **Plan MRNT**:

| Letra | Significado | Sala (key) | Color  |
|-------|-------------|-----------|--------|
| **M** | Misión      | `mision`  | azul `#3b82f6` |
| **R** | Relación    | `confra`  | amarillo `#eab308` |
| **N** | Nutrición   | `repaso`  | naranja `#fb923c` |
| **N** | Nutrición   | `juegos`  | naranja claro `#f97316` |
| **T** | Templo      | `templo`  | violeta `#7c3aed` |

Además hay un banner "Entrada" (navy) arriba donde aterrizan los avatares al
entrar. El layout mobile-first es: Entrada en banner superior + 4 columnas
verticales debajo (M · R · N · T), donde la columna N contiene Repaso (mitad
superior) y Juegos (mitad inferior).

`detectZone(x,y)` simplificado: si `y<=14` → `entrada`; si no → `sala`. La
fase activa (confra/repaso/juegos/mision/templo) la dispara el admin desde
su panel; todos los que estén en `sala` participan.

---

## 2. Roles y autenticación

### Acceso
- Login con Firebase Auth: **email+password** o **Google popup**.
- Persistencia `LOCAL`. Email de bienvenida vía EmailJS (`service_yczb7rn`,
  template `template_t8izci5`, public key `YBvr8UJciOIrsFqMI`).
- Tras login → pantalla **Setup**: el usuario elige `name`, `avatar` (emoji),
  `color`, foto opcional (comprimida a max 200px), y `clase` (`gteen` | `ja`).
- Al guardar perfil → entra a la sala con `zone:'entrada'`, `x:49, y:9`.
- `rooms/<SID>/users/<uid>` con `onDisconnect().remove()` para limpieza.
- **Asistencia automática**: si `new Date().getDay()===6` (sábado) al entrar,
  se escribe en `attendance/<uid>/<SID>` (nombre, avatar, color, ts).
  Muestra toast "✓ Asistencia registrada".

### Admin
- Lista hard-coded `ADMIN_EMAILS = ['sromeromolina10@gmail.com']`.
- Al iniciar sesión por primera vez con un correo de la lista, se persiste
  `profiles/<uid>/isAdmin = true`.
- **Visualmente**: el admin ve un **botón flotante ⚙️ abajo a la derecha**
  que abre el **AdminPanel**. Los invitados no lo ven.
- Solo el admin controla las fases (Iniciar/Finalizar Confra, Repaso,
  Juegos, Misión, Templo), las presentaciones, los videos, la nube de
  palabras, la encuesta, la descarga de datos y el reset.
- **Vista de proyección (`ScreenView`)**: el admin puede entrar a una vista
  pantalla completa con QR de invitación, conteo por zona y el chat de
  repaso en pantalla. Para proyectar en TV/pantalla.

### Invitados (usuarios normales)
- Ven la sala, el header con su avatar y la fase actual, el chat del
  repaso, y los modales que el admin va abriendo (confra speaker,
  presentación, video, nube, juegos, misión, templo/encuesta).
- Pueden mover su avatar entre Entrada y Sala (drag).
- Pueden enviar reacciones, mensajes y participar en cada actividad.
- Pueden editar su perfil (`EditProfileModal`) y cambiar de clase
  (Gteen ↔ JA) — sólo cambia el logo/color del avatar.
- Pueden ver su asistencia personal y juegos ganados (`AttendanceModal`).

---

## 3. Estructura del canvas (sala virtual)

Componente: `RoomCanvas(users, myId, session, onDragMove, roomRef, full,
reactions, classKey)`.

- Fondo **crema** `#f8f5ef`.
- Renderiza al centro un **diagrama MRNT** (`MRNTDiagram`), que es un SVG
  con 4 pétalos coloreados (M azul, R amarillo, N naranja, T violeta). El
  pétalo correspondiente a la fase activa se resalta.
- Alrededor del diagrama: 6 "cards" de zona (Entrada arriba; columnas M, R,
  N(Repaso), N(Juegos), T).
- Avatares (`AvatarBubble`) se posicionan en x/y porcentuales sobre el
  contenedor. Se arrastran con `pointermove`; al soltar se calcula la
  zona destino y se persiste en `rooms/<SID>/users/<uid>` con `{x,y,zone}`.
- Cada avatar muestra el emoji+color del usuario y, opcional, badge con el
  logo de su clase (Gteen rojo / JA azul).
- Reacciones flotantes (`reactionFloat` keyframe): un emoji sube y se
  desvanece.
- Conteos por zona se calculan en vivo (`entradaCount`, `salaCount`).
- En modo `full={true}` (proyección), el canvas ocupa toda la pantalla y
  muestra QR para invitar (`https://api.qrserver.com/v1/create-qr-code/...`
  apuntando a `window.location.origin+pathname`), panel de fase, timer.

---

## 4. AdminPanel (botón ⚙️)

Mini-modal flotante. Muestra:

- Chip de fase actual (color según fase).
- Grid 5 columnas con conteo de usuarios en cada zona MRNT.
- **Botones según `phase`**:
  - `waiting`:
    - M · Compromiso · `onStartMision(users,'commitments')`
    - M · Proyectos · `onStartMision(users,'projects')`
    - R · Confraternización · `onStartConfra(users)`
    - N · Repaso Bíblico · `onStartRepaso(users)`
    - N · Juegos · `onStartJuegos()`
    - T · Templo · `onStartTemplo()` (abre encuesta semanal)
  - `confra`: Siguiente persona / Finalizar confra
  - `repaso`: Iniciar/Cerrar presentación · 🎬 Videos · ☁ Nube de palabras
    (con input de pregunta) · Finalizar repaso
  - `juegos`: Finalizar juegos
  - `mision`: Finalizar misión
  - `templo`: Finalizar templo
- Botones siempre visibles:
  - 📥 Descargar datos (`DataDownloadModal`)
  - 📊 Ver estadísticas globales (`AdminStatsModal`)
  - 🔄 Reiniciar sesión (`resetSession`)
  - Si hay encuesta activa: 📋 Encuesta · ver respuestas (con contador
    `respondidos/total`, verde cuando todos respondieron).

---

## 5. Fases / Salas

Todas las fases viven en `sessions/<SID>` en Realtime Database. La fase
activa es `sessions/<SID>/phase`. Duraciones sugeridas:
```js
PHASE_DURATIONS = { confra:600, repaso:1800, juegos:900, mision:1200, templo:1200 }
```

### 5.1 Entrada (Llegada)
- Banner navy arriba. Apenas el usuario entra, queda aquí.
- Sin actividad propia. Es sala de espera mientras llega gente.
- El admin ve cuántos están en Entrada vs cuántos en Sala.

### 5.2 R · Confraternización
- Admin lanza `startConfra(confraUsers)` → arma una **cola aleatoria**
  (`Math.random()` shuffle) con los UIDs de quienes están en sala.
- Se guarda en `sessions/<SID>/confra = {active, queue, currentIdx:0, startTime}`.
- Pantalla `ConfraSpeakerPanel` se abre para **todos** mostrando al
  speaker actual: avatar grande, nombre, frase "está compartiendo",
  pregunta "🙏 ¿Quieres contarnos, pedir o agradecer algo?".
- El **speaker actual** ve "¡Es tu turno!" y un botón "Terminé →
  Siguiente" (o "Listo, todos compartieron ✨" si es el último).
- Los **demás** ven 6 botones de reacciones (`❤️ 🙏 👍 😊 😢 🎉`). Se
  pueden seleccionar varias. Cuentas acumuladas visibles para todos en
  chips. Las reacciones se guardan en `sessions/<SID>/reactions/<speakerUid>/<uid>/<reactionId>` y se borran al pasar al siguiente.
- El admin ve "⏭ Saltar" y "✓ Finalizar confra".
- Indicador de progreso con puntos (uno por persona).
- Timer de 10 min (`confra`).

### 5.3 N · Repaso Bíblico
- Admin lanza `startRepaso(repasoUsers)` → `phase:'repaso'`.
- Se activan en paralelo varias subherramientas:
  - **Chat de repaso** (`RepasoChat`): mensajes en `rooms/<SID>/chat`. Cada
    mensaje tiene `{uid, name, avatar, color, text, ts, tag?, img?}`.
    Soporta:
    - Texto + emojis rápidos (`QUICK_EMOJIS`).
    - Imágenes (comprimidas).
    - **Tags especiales** (`TAG_TYPES`): el usuario puede taggear un
      mensaje como pregunta, idea, oración, etc. Cada tag abre un
      `TagMessageModal`.
    - Reacciones a mensajes (`chatReactions`).
    - En proyección (`RepasoChatScreen`), el chat aparece a un costado
      para que todos vean lo que se comenta.
  - **Presentaciones** (ver §6).
  - **Videos sincronizados de YouTube** (ver §7).
  - **Nube de palabras** (`WordCloudActivity`, ver §8).
- Botón "⏹ Finalizar Repaso" cierra: borra chat y wordcloud,
  `phase:'waiting'`, muestra `ActivityEndBanner`.
- Timer de 30 min; tras agotarse muestra "TIEMPO EXTRA" pulsante.

### 5.4 N · Juegos
- Admin lanza `startJuegos()` → `phase:'juegos'`.
- Se abre `GameSelectMenu` con todos los juegos disponibles (ver §9).
- Timer de 15 min.

### 5.5 M · Misión
- Admin lanza `startMision(users, mode)` con `mode='commitments'` o
  `mode='projects'`.
- `sessions/<SID>/mision = {active, mode, startTime, commitments?, projects?}`.
- Componente `MisionActivity`:
  - **`commitments`**: cada usuario escribe su **compromiso misionero
    semanal** (máx 240 chars). Se guarda en
    `mision/commitments/<uid> = {name, color, avatar, photo, text, ts}`.
    Todos ven la lista en vivo (ordenada por timestamp). Cada uno puede
    editar/borrar el suyo.
  - **`projects`**: 4 proyectos hard-coded (`MISION_PROJECTS`):
    Voluntarios 🙌, ASA 🤲, OYIM 🌍, Amigo 🤝. Cada usuario puede
    "tocar la oración" en un proyecto (toggle). Se ve quién oró por
    cada uno (`mision/projects/<projectId>/prayers/<uid>`).
- Timer 20 min.
- Sólo admin puede cerrarla.

### 5.6 T · Templo
- Admin lanza `startTemplo()` → `phase:'templo'` + abre **encuesta
  semanal** automáticamente (`sessions/<SID>/survey = {active:true,
  responses}` — preserva respuestas previas del día).
- Encuesta (`SurveyModal`): un set de preguntas que cada usuario responde
  una sola vez. El admin la ve en `AdminSurveyModal` con conteo de
  respondidos vs pendientes, y botón de descargar resultados.
- Timer 20 min.
- Cerrarla apaga `templo/active` y `survey/active`.

---

## 6. Presentaciones (slides sincronizados)

- Catálogo en constante `PRESENTATIONS = [...]`. Cada presentación tiene
  `{id, week, weekLong, title, subtitle, icon, slides:[...]}`.
- Hoy hay 3 presentaciones cargadas:
  1. **`semana09`** — "El pecado, el evangelio y la ley" (Mateo 5–7).
  2. **`semana10`** — "Arrepentimiento y perdón" (Éxodo 34:1-9).
  3. **`algoritmos`** — "Fe más allá de los algoritmos" (tema especial,
     con imágenes desde `assets/algoritmos/slideNN.{png,jpg}`).
- **Cómo se elige**: durante la fase `repaso`, el admin pulsa "▶️ Iniciar
  presentación" → se abre `PresentationMenu` con la lista. Al elegir, se
  hace `startPresentation(presId)` → `sessions/<SID>/presentation =
  {active, id, slide:0, total, startedAt}`.
- **Cómo se ve en línea**: `PresentationOverlay` se abre para **todos los
  conectados** con la slide actual. El admin tiene barra inferior con:
  - ◀ / ▶ navegación (también flechas del teclado y PageUp/Down).
  - Indicador `N/Total · DÍA`.
  - Botón "⋯ Otras acciones" que despliega:
    - 🎬 Reproducir/Cambiar video → abre `VideoMenu`.
    - ☁ Abrir/Cerrar nube de palabras → pide pregunta opcional.
    - 📲 Mostrar/Ocultar QR.
  - "⏹ Finalizar".
- Los invitados ven la misma slide sin controles, con un chip
  "El anfitrión controla la presentación".
- **Tipos de slide** que renderiza `SlideView`:
  - **Plantilla verde** (`SLIDE_T`, bg verde oliva claro):
    - `cover` — portada con número de lección, título, ref bíblica.
    - `quote` — cita estilo red social.
    - `list` — lista numerada estilo "Investiga".
    - `egw` — cita de Elena de White, fondo crema con borde verde.
    - `dialoga` — 3 columnas de preguntas para conversar.
    - `preview` — vistazo de la próxima semana (BookLogo grande + texto).
    - `credits` — créditos del Ministerio Joven con redes sociales.
    - `content` — slide principal con título del día, kicker
      (INICIA/ESCRIBE/ASIMILA/INTERPRETA/INVESTIGA/ENFOCA/REFLEXIONA), un
      `VersesBox` con referencias bíblicas, y 4 cards (box/plain/note) con
      `{lead, rest}`.
  - **Plantilla FAA** (navy + dorado, para "Fe más allá de los
    algoritmos"):
    - `faa-image` — imagen a sangrado.
    - `faa-quote` — cita bíblica grande con comillas doradas.
    - `faa-image-quote` — imagen + cita lado a lado.
    - `faa-angels` — mensaje de los 3 ángeles (Apocalipsis 14).
    - `faa-title-image` — título + imagen.
    - `faa-testimony` — testimonio (Ben Carson, Samuel Melquiades…).
    - `faa-quote-bg` — cita con imagen de fondo.
- Cada slide se envuelve en `SlideShell` (verde) o `FAAShell` (navy) con
  indicador de progreso (dots), pie con `week · subtitle`, número de slide.

### Biblia y links
- En las slides `content`, las referencias bíblicas (verses) aparecen en
  `VersesBox` con icono 📖 y formato "Libro Capítulo:Versículo".
- **No hay link clickeable a una Biblia externa**: los versículos se
  imprimen como referencia textual para que cada persona los abra en su
  Biblia (papel/app). El diseño visual evoca tarjetas tipo Bible-app.
- En `credits` sí hay links a redes sociales del Ministerio Joven Chile
  (Facebook, Instagram, Telegram, YouTube, App Store, Google Play).
- Las presentaciones FAA usan referencias bíblicas (Mateo, Apocalipsis,
  Isaías, 1 Tesalonicenses) impresas en dorado/serif.

### Diseño MRNT en las presentaciones
- Las slides del repaso (semana09/10) **NO** llevan el diagrama MRNT;
  usan la plantilla verde oliva del Ministerio Joven.
- El **diagrama MRNT** sí aparece en el `RoomCanvas` (sala virtual)
  durante todas las fases, con el pétalo activo resaltado según la fase
  (M azul cuando misión, R amarillo cuando confra, etc.).
- Componente `MRNTDiagram({size, interactive, onSectorClick, activeSector,
  labels, hideCenterDot})` — SVG con 4 pétalos (`Petal`) en `MRNT_SECTORS`,
  rotación opcional, halo al activarse.

---

## 7. Videos sincronizados (YouTube)

- Catálogo en constante `VIDEOS = [{id, title, subtitle, icon, youtubeId}]`.
- Hoy: "A comenzar en mi" y "Ben Carson".
- Para agregar uno: pegar nuevo `{id, title, icon, youtubeId}` (el
  `youtubeId` son los 11 chars del link de YouTube).
- Admin abre `VideoMenu` (desde el panel de repaso o desde "Otras
  acciones" en una presentación) → elige → `startVideo(catalogId)`
  → `sessions/<SID>/video = {active, videoId, title, state:'paused',
  position:0, updatedAt}`.
- `VideoOverlay` renderiza el iframe de YouTube IFrame Player API.
  - **Admin**: ve controles nativos. Cada play/pausa/seek escribe en
    Firebase (`writeState(state, position)`).
  - **Invitados**: ven el video sin controles; un overlay transparente
    bloquea clicks. Empieza muteado (política de autoplay); botón
    "Activar audio" desmutea.
  - Sincroniza vía `syncFromFirebase` con tolerancia ~1s.
- El video puede coexistir con una presentación: se renderiza por encima;
  al cerrarlo la slide vuelve a verse (útil en las slides 13–14 de FAA
  donde hay testimonios).

---

## 8. Nube de palabras (WordCloud)

- Admin activa desde el panel del Repaso o desde "Otras acciones" en la
  presentación. Puede escribir una pregunta (opcional) que ven todos.
- `sessions/<SID>/wordcloud = {active, words:{<uid>:{...}}, question}`.
- `WordCloudActivity`:
  - Cada usuario escribe una palabra; se guarda con su uid en `words`.
  - Las palabras se agrupan por frecuencia, se renderizan con tamaño
    proporcional (`getFontSize(count)`), color del catálogo MRNT
    (`MRNT_COLORS`), rotación aleatoria por hash y jitter x/y.
  - El admin tiene botón "Limpiar" (borra `words`) y "Detener".
- Sirve para preguntas como "¿Qué aprendiste hoy?".

---

## 9. Juegos

`GameSelectMenu` muestra todos los juegos. Cada uno tiene su propio
controlador y estado en `sessions/<SID>/<juego>`.

### 9.1 Kahoot (preguntas con biblioteca + custom)
- Biblioteca hard-coded `GAME_LIBRARY`: Semana 2 (Juan 17), Semana 4
  (2 Timoteo 3), Semana 9 (Mateo 5–7). Cada una con 7–10 preguntas, 4
  opciones, índice correcto.
- Admin elige una de la biblioteca o "Crear nuevo juego" con `KahootSetup`
  (formulario para agregar preguntas/opciones).
- `KahootController` arma la partida:
  - `sessions/<SID>/kahoot = {questions, currentQ, gameState, answers, scores, qStart}`.
  - `gameState`: `waiting` → `playing` → `result` → `finished`.
  - Por pregunta: `GAME_TIMER = 30s`. Cada usuario tiene 4 botones con
    formas (`▲◆●■`) y colores (`ANS_COLORS`).
  - Al responder, se guarda `answers/<qIdx>/<uid> = ansIdx`. Se calculan
    puntos (más rápido → más puntos).
  - `KahootQuestion`, `KahootResultCard` y `KahootLeaderboard` muestran
    pregunta, resultado por pregunta y podio final con confetti.
  - El admin puede saltar pregunta, terminar tiempo, terminar partida.
  - El podio acepta reacciones (`podiumReactions`).
- Al cerrar se persiste `gameHistory/<uid>` con `{rank, ts, name}` para el
  ranking histórico.

### 9.2 Tabú Bíblico
- Cartas hard-coded `TABU_CARDS` (palabra a adivinar + 5 palabras
  prohibidas).
- `TabuController`:
  - Admin selecciona quién describe (random o manual).
  - El descriptor ve la carta; los demás intentan adivinar.
  - Admin marca quién adivinó (le suma punto), o "Skip", o "Otra carta".
  - Timer de 3 min por turno.
  - `sessions/<SID>/tabu = {gameState, currentCard, descriptor, scores, timeLeft}`.

### 9.3 ¿Quién soy?
- Personajes bíblicos hard-coded `QUIEN_SOY_PERSONAJES`.
- `QuienSoyController`:
  - Admin elige a quién le toca adivinar (random o manual).
  - Esa persona ve "¿Quién soy?" en su pantalla; todos los demás ven el
    personaje y le responden sí/no a sus preguntas.
  - Admin marca al adivinador correcto y le da el punto.
  - Sin timer; va por turnos.

### 9.4 Dibujando la Biblia
- Categorías `DIBUJA_CATS = {animal, verbo, personaje, lugar}` con
  palabras (`DIBUJA_WORDS`).
- Juego por **equipos**, con **tablero de 30 casillas** (`DIBUJA_BOARD`) y
  dado.
- `DibujaController` flujo:
  1. `setup`: admin define cantidad de equipos, asigna miembros (manual o
     "repartir al azar"), asigna avatares/colores a cada equipo.
  2. `captains`: los miembros votan a su capitán; el más votado dibuja /
     hace mímica.
  3. `play`: cada turno, equipo en juego tira el dado para decidir si
     dibuja, mima o habla (modo). El capitán saca carta (categoría +
     palabra). Tiene tiempo limitado para que su equipo adivine.
  4. Si adivinan → mueven N casillas en el tablero (según dado).
  5. El primer equipo en llegar a `DIBUJA_GOAL` gana.
- Canvas en vivo `DibujaCanvas` sincronizado por Firebase
  (`sessions/<SID>/dibuja/strokes` + `live` para el trazo en curso).
- Tablero `DibujaBoard` muestra avatares de cada equipo avanzando.
- `DibujaTeamsWatch` resume equipos y miembros para los que miran.

### Cómo agregar un juego nuevo
Para sumar un juego a `GameSelectMenu`:
1. Definir constantes con su contenido (cartas / preguntas / palabras).
2. Crear un `XxxController` que lea/escriba en
   `sessions/<SID>/<key>` con un `gameState` propio.
3. En `App`, agregar handler `startXxx`/`endXxx` que setee
   `phase:'juegos'` y persista el estado inicial.
4. Añadir un botón en `GameSelectMenu` con su icono/color/título.
5. Renderizar el controlador cuando `phase==='juegos'` y el subkey esté
   activo.

---

## 10. Asistencia y racha

- Al unirse a la sala un **sábado** (`getDay()===6`), se escribe
  `attendance/<uid>/<SID> = {name, avatar, color, timestamp}`.
- Las claves de `attendance` pueden ser `2026-06-13` (formato viejo) o
  `gteen-2026-06-13` / `ja-2026-06-13` (con prefijo de clase).
- **`AttendanceModal`**: cada usuario ve su histórico (lista por fecha) y
  su `gameHistory` con cuántas veces ganó (rank 1) o quedó top 3.
- **Racha histórica (`streak`)**: días consecutivos asistidos (tolerancia
  10 días entre uno y otro).
- **Mi Racha Espiritual del mes (`monthlyStreak`)**: sábados únicos
  asistidos en el mes en curso (0–4, tope 4).
- Niveles (`STREAK_LEVELS`): 0 😴 sin asistencia · 1 🤒 · 2 🙂 · 3 😊 ·
  4 🎉 ¡Mes completo!. Iconos, no palabras (para no juzgar).
- El admin tiene **`DataDownloadModal`** con descarga CSV por rango de
  fechas de:
  - Asistencia (uid, name, fechas, total).
  - Juegos ganados.
  - Respuestas de encuestas.
- **`AdminStatsModal`**: panel global con buscador, ranking de victorias
  en juegos, ranking de top 3, ranking por cantidad de juegos.

---

## 11. Confraternización (recordatorio) y acceso por sala

- La **Confraternización** es la fase `confra` descrita en §5.2.
- **Cola aleatoria**: al iniciarla el admin, se shufflean los UIDs en
  sala. Se va pasando uno por uno: el que tiene el turno aprieta
  "Terminé → Siguiente". Los demás reaccionan con emojis.
- Las reacciones se acumulan y se muestran en chips con conteo. Al pasar
  de speaker se limpian.
- El admin puede saltar a alguien o finalizar antes de tiempo.

### Acceso a cada "sala"
- **Una sola sala física** (`maranata-YYYY-MM-DD`). Gteen y JA comparten.
- Los usuarios **no eligen sala**, eligen **fase activa** moviéndose dentro
  del canvas (Entrada vs Sala) y participan en la actividad que el admin
  haya iniciado.
- El cambio entre Gteen y JA desde el topbar de la sala (`switchClass`)
  saca al usuario de la sala actual y lo mete en una sala separada
  (`gteen-YYYY-MM-DD` / `ja-YYYY-MM-DD`), recargando la app. Esto se usa
  cuando un usuario marcó la clase equivocada al registrarse.

---

## 12. Estructura de Firebase RTDB

```
profiles/<uid> = { name, avatar, color, photo, clase, isAdmin }
attendance/<uid>/<SID> = { name, avatar, color, timestamp }
gameHistory/<uid>/<gameId> = { rank, ts, name }
rooms/<SID>/users/<uid> = { name, avatar, color, photo, clase, zone, x, y, joinedAt }
rooms/<SID>/chat/<msgId> = { uid, name, avatar, color, text, ts, tag?, img? }
sessions/<SID> = {
  phase: 'waiting'|'confra'|'repaso'|'juegos'|'mision'|'templo',
  confra: { active, queue, currentIdx, startTime },
  repaso: { active, startTime },
  juegos: { active, startTime },
  mision: { active, mode, startTime, commitments, projects },
  templo: { active, startTime },
  presentation: { active, id, slide, total, startedAt },
  video: { active, videoId, title, state, position, updatedAt },
  wordcloud: { active, words, question },
  survey: { active, responses },
  reactions: { <speakerUid>/<uid>/<reactionId>:true },
  kahoot|tabu|quiensoy|dibuja: { ...estado del juego activo... },
  lastBanner: 'confra'|'repaso'|...  // para mostrar ActivityEndBanner
}
```

---

## 13. Resumen ejecutivo del flujo de un sábado

1. **Admin** abre la app en una pantalla grande → entra al modo
   `ScreenView` con QR visible.
2. **Invitados** escanean QR, hacen login (email o Google), eligen nombre/
   avatar/color/clase, entran a la sala. Su asistencia queda guardada.
3. **Entrada**: todos charlan, mueven sus avatares.
4. Admin pulsa **M · Misión** → todos escriben su compromiso semanal o
   eligen proyecto por el que orar.
5. Admin pulsa **R · Confra** → cola aleatoria, cada uno comparte algo,
   los demás reaccionan.
6. Admin pulsa **N · Repaso** → abre presentación de la semana, controla
   slides, intercala videos, abre nube de palabras para preguntas, el
   chat queda abierto.
7. Admin pulsa **N · Juegos** → elige Kahoot / Tabú / ¿Quién soy? /
   Dibujando.
8. Admin pulsa **T · Templo** → momento devocional, encuesta semanal se
   abre automáticamente. Todos responden.
9. Admin descarga CSV de asistencia/juegos/encuestas para reportes.
