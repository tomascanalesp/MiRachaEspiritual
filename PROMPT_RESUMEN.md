# Prompt resumen — Funcionalidades de Mi Racha Espiritual / Maranata Class

> Pega este prompt en Claude Design (o el asistente que estés usando) para
> recuperar las funciones de la sala virtual. Está escrito de forma
> resumida pero detallada, organizado por sala / actividad. Incluye
> también cómo funcionan el admin, los usuarios, las presentaciones, la
> Biblia, el diseño MRNT, la asistencia, los juegos, el QR y la
> confraternización.

---

## 1. CONTEXTO GENERAL

App web (React + Firebase RTDB + Firebase Storage + EmailJS) llamada
**Mi Racha Espiritual / Maranata Class**, usada por los jóvenes
adventistas (Gteen y JA) de la iglesia de Las Condes los sábados 9:00.
Todos entran a UNA sola sala virtual (`maranata-YYYY-MM-DD`), sin
importar si son Gteen o JA — la clase solo cambia el **badge/logo** del
avatar.

- Auth: email + contraseña (Firebase Auth, persistencia LOCAL).
- Perfiles: nombre, avatar (emoji o foto comprimida a 200px base64),
  color, clase (`gteen` | `ja`).
- Realtime: posiciones, fase de la sala, chat, encuestas, juegos, nube
  de palabras, presentación y video se sincronizan por
  `sessions/<SID>/...`.
- Versión auto-actualizable: si cambia `APP_VERSION`, los clientes
  abiertos recargan solos.
- Correo de bienvenida HTML vía EmailJS al registrarse (template en
  `buildWelcomeHTML`).

---

## 2. PLAN MRNT (diseño visual de la sala)

La sala se organiza visualmente alrededor de un **diagrama MRNT** de 4
pétalos (Misión · Relación · Nutrición · Templo). Cada pétalo tiene su
color oficial y mapea a una zona/actividad.

| Letra | Significado | Zona       | Color principal | Color oscuro | Icono |
|-------|-------------|------------|-----------------|--------------|-------|
| M     | Misión      | `mision`   | `#3b82f6`       | `#1d4ed8`    | ✈️    |
| R     | Relación    | `confra`   | `#eab308`       | `#a16207`    | 🤝    |
| N     | Nutrición   | `repaso`   | `#fb923c`       | `#c2410c`    | 📖    |
| N     | Nutrición   | `juegos`   | `#f97316`       | `#c2410c`    | 🎮    |
| T     | Templo      | `templo`   | `#7c3aed`       | `#5b21b6`    | ⛪    |
| —     | Llegada     | `entrada`  | `#0a0e2e`       | `#0a0e2e`    | 🚪    |

- Fondo de sala: crema `#f8f5ef` con gradientes radiales sutiles.
- Tipografía: `Barlow Condensed` 900 para títulos uppercase, `Barlow`
  600 para body, `Fraunces` serif para landing.
- El **diagrama MRNT central** se compone de 4 `<Petal>` SVG (helper
  geométrico). El componente `MRNTDiagram` acepta props `size`,
  `interactive`, `onSectorClick`, `activeSector`, `labels`,
  `hideCenterDot`.
- La fase activa de la sala se highlightea por color/letra MRNT en el
  header.
- Layout actual (mobile-first): banner ENTRADA arriba (full-width,
  `y:0–14%`) + 4 columnas verticales debajo (M · R · N · T). La columna
  N se parte en mitad superior (Repaso) / mitad inferior (Juegos).
- `detectZone(x,y)` simplificado: `y<=14` → entrada, resto → `sala`.
  Las claves MRNT se mantienen solo para el highlight del header.
  `effectiveZone(u)` recalcula desde (x,y) si el zone es legacy.

---

## 3. ROLES — ADMIN vs USUARIOS

### Admin
- Lista hardcodeada en `ADMIN_EMAILS` (ej.
  `sromeromolina10@gmail.com`); flag persistido en
  `profiles/<uid>/isAdmin`.
- Tiene un **botón flotante ⚙️ amarillo abajo a la derecha** que abre
  el `AdminPanel`. El panel muestra:
  - Conteos por zona MRNT (M, R, N·Repaso, N·Juegos, T).
  - Chip con la fase actual (`waiting | confra | repaso | juegos |
    mision | templo`).
  - Acceso a Encuesta (ver respuestas con badge X/Total), Descargar
    datos (CSV/Excel), Estadísticas globales, Reiniciar sesión.
  - Botones para arrancar cada actividad según la fase:
    - `waiting`: M·Compromiso, M·Proyectos, Confraternización, Repaso,
      Juegos, Templo.
    - `confra`: → Siguiente persona / ✓ Finalizar confra.
    - `repaso`: ▶️ Iniciar presentación / ⏹ Cerrar presentación, 🎬
      Videos, ☁️ Iniciar/Cerrar Nube de palabras (con input opcional
      de pregunta), ⏹ Finalizar Repaso.
    - `juegos`: ⏹ Finalizar Juegos.
    - `mision` / `templo`: ⏹ Finalizar.
- Tiene una **vista de proyección** (`ScreenView`) con el canvas a
  pantalla completa + QR de acceso arriba a la izquierda + panel de
  fase activa con timer arriba a la derecha. Pensada para proyectar en
  la pared de la sala física.
- En cada actividad ve controles extra: saltar speaker, marcar
  adivinador, revelar carta, otra carta/personaje, finalizar, cerrar.

### Usuarios (no admin)
- Login → eligen nombre, avatar emoji o foto, color, clase (Gteen o JA).
- Entran a la sala y aparecen como **avatar arrastrable** en el canvas.
  Pueden moverse libremente entre zonas (drag con mouse o touch).
- Ven la fase activa y participan automáticamente cuando el admin
  inicia una actividad (si están en zona `sala`).
- Pueden lanzar **reacciones flotantes** (❤️ 🙏 👍 😊 😢 🎉) sobre el
  canvas; aparecen como burbujas que suben y se desvanecen.
- Pueden editar perfil (foto, nombre, color, clase) desde
  `EditProfileModal`.
- Pueden abrir `AttendanceModal` para ver su propia asistencia y
  juegos.

---

## 4. SALAS / FASES Y QUÉ APARECE EN CADA UNA

### 4.1 ENTRADA / WAITING (fase por defecto)
- Banner navy en la parte superior; los avatares aparecen ahí al
  llegar.
- El admin ve sus contadores y los botones para iniciar cada
  actividad.
- Aparece el QR de acceso en la vista de proyección (`ScreenView`).
- Música/landing claro; los usuarios pueden moverse a `sala`
  (cualquier zona MRNT) para "elegir" en qué quieren participar
  cuando arranque.

### 4.2 M · MISIÓN  (color azul `#3b82f6`, icono ✈️)
Componente: `MisionActivity`. Timer 20 min (`PHASE_DURATIONS.mision =
1200`). Dos modos elegidos por el admin:

- **Compromisos** (`mode:'commitments'`)
  - Cada persona escribe su compromiso de la semana (máx 240
    caracteres). Ej.: "invitar a Marco al culto", "llevar mercadería
    a la familia X".
  - Aparece lista con todos los compromisos de la sala (avatar +
    nombre + texto).
  - Puede editar/borrar el propio.
- **Proyectos** (`mode:'projects'`)
  - 4 proyectos misioneros oficiales:
    - 🙌 Misión de Voluntarios — Pioneros y misioneros (azul)
    - 🤲 ASA — Acción Social Adventista (verde)
    - 🌍 OYIM — Misión global juvenil (amarillo)
    - 🤝 Amigo — Evangelismo de amistad (rosa)
  - Cada usuario toca un proyecto para "orar por él"; queda marcado
    su avatar en ese proyecto. Puede orar por varios.
  - Vista grid 2×2 con animación pop al seleccionar.

Admin ve: timer, conteos, botón ⏹ Finalizar Misión.

### 4.3 R · CONFRATERNIZACIÓN (color amarillo `#eab308`, icono 🤝)
Componente: `ConfraSpeakerPanel`. Timer 10 min
(`PHASE_DURATIONS.confra = 600`).

- Funcionamiento: cuando el admin inicia, se arma una **cola
  aleatoria** con todos los usuarios presentes (`confra.queue`,
  `currentIdx`). Va de uno en uno.
- Pantalla centrada con avatar grande (90px, con glow del color del
  speaker), nombre en mayúsculas y pregunta: "🙏 ¿Quieres contarnos,
  pedir o agradecer algo, {nombre}?".
- Si **es tu turno**: ves "¡Es tu turno!" y un botón grande
  "Terminé → Siguiente" (o "Listo, todos compartieron ✨" si es el
  último).
- Si **no eres el speaker**: ves 6 botones de reacciones (❤️ 🙏 👍 😊
  😢 🎉) que puedes seleccionar (varias). Cada reacción tuya se guarda
  en `sessions/<SID>/reactions/<speakerUid>/<myUid>/<rid>`.
- Bajo el avatar aparece la **suma acumulada de reacciones** que
  recibió el speaker (chips con emoji + contador).
- Progreso con puntitos (uno por cada persona en la cola: verdes los
  ya pasados, color del speaker actual el activo, gris los
  pendientes).
- Admin ve botones: ⏭ Saltar, ✓ Finalizar confra.

### 4.4 N · REPASO BÍBLICO (color naranja `#fb923c`, icono 📖)
Timer 30 min (`PHASE_DURATIONS.repaso = 1800`). Es el **corazón** del
sábado: chat + presentación + videos + nube de palabras.

#### 4.4.1 Chat (`RepasoChat`)
- Mensajes en tiempo real (`sessions/<SID>/messages`).
- Cada mensaje muestra avatar/color del autor; mensajes propios
  alineados a la derecha.
- Soporta:
  - Texto.
  - Imágenes (comprimidas a 480px base64).
  - **Mensajes con tag** (cards especiales): `TAG_TYPES`
    (ej. "pregunta", "testimonio", "petición", "agradecimiento" —
    abren `TagMessageModal`).
  - Emojis picker.
  - Reacciones por mensaje (REACTIONS × usuario;
    `chatReactions/<msgId>/<uid>/<rid>`).
- Header: indicador con animación pulse, conteo de personas en sala,
  timer, avatares apilados de los últimos 5 presentes.
- Vista compacta para proyección: `RepasoChatScreen` muestra los
  últimos 6 mensajes.

#### 4.4.2 Presentaciones (`PresentationOverlay`, `SlideView`,
`PresentationMenu`)
- El admin abre un menú con todas las **presentaciones** disponibles
  (`PRESENTATIONS`). Cada presentación tiene `id`, `week`, `title`,
  `subtitle`, `icon` y un array `slides`.
- Tipos de slide soportados:
  - `cover` — portada con bigTitle, num, lessonTitle, ref.
  - `content` — día + kicker (INICIA/ASIMILA/INTERPRETA/...) + título +
    `verses` (referencias bíblicas) + `cards` con variantes
    (`box`/`plain`/`note`).
  - `quote` — cita tipo tweet con avatar/handle.
  - `list` — secciones numeradas (01/02/03) con título y refs.
  - `egw` — cita de Elena G. de White con texto largo + autor.
  - `dialoga` — 3 columnas con preguntas para discutir.
  - `preview` — vista previa de la siguiente semana.
  - `credits` — créditos finales con plataformas (Facebook, IG,
    Telegram, YouTube, App Store, Google Play).
  - `verse`, `faa-image`, `faa-quote`, `faa-image-quote`, `faa-angels`,
    `faa-title-image`, `faa-testimony`, `faa-quote-bg` (para la
    presentación especial "Fe más allá de los algoritmos").
- Cada slide se sincroniza por
  `sessions/<SID>/presentation.{active,id,slide}`; todos los usuarios
  ven la misma slide al mismo tiempo.
- **Las referencias bíblicas** (`verses`) aparecen en un componente
  `VersesBox` (caja blanca con 📖 + lista bullet). Idea: cada
  referencia se renderiza con un link a un visor bíblico
  (Reina-Valera). NOTA: actualmente son texto plano; ver § 5.
- Admin tiene una barra inferior con: ◀ ▶ navegación, contador
  `idx/total`, botón "Otras acciones" (☁️ Nube, 🎬 Video, 📲 QR), y
  ⏹ Finalizar. Teclas ← / → en desktop.
- Para usuarios no admin: "El anfitrión controla la presentación".

#### 4.4.3 Videos (`VideoOverlay`)
- Catálogo `VIDEOS` con id, title, subtitle, icon, youtubeId.
- Admin abre menú; al elegir uno, se inserta YouTube IFrame
  sincronizado por `sessions/<SID>/video` (play/pause/seek vía RTDB).
- Botón ⏹ Cerrar video en admin panel.
- Se puede lanzar **encima** de una slide (slide queda detrás, video
  cubre la pantalla).

#### 4.4.4 Nube de palabras (`WordCloudActivity`)
- Admin la activa (opcional: con pregunta). Cada usuario manda hasta
  5 palabras (máx 3 palabras por entrada).
- Tabs "Escribir" / "Ver Nube".
- La nube renderiza palabras con tamaño proporcional a la frecuencia,
  rotación aleatoria (±25°), jitter X/Y para romper la grilla. Colores
  MRNT al azar por palabra.
- Admin puede 🗑 limpiar y ⏹ terminar.

### 4.5 N · JUEGOS (color naranja claro `#f97316`, icono 🎮)
Timer 15 min (`PHASE_DURATIONS.juegos = 900`). Menú `GameSelectMenu`
con 4 juegos + 1 opción de crear Kahoot nuevo.

#### 4.5.1 Kahoot bíblico (`KahootController` + `KahootSetup` +
`KahootQuestion` + `KahootResultCard` + `KahootLeaderboard`)
- Biblioteca `GAME_LIBRARY` con kahoots predefinidos (Semana 2, 4, 9
  — cada uno tiene 7–10 preguntas con 4 opciones y `correct`).
- Admin puede elegir uno de la biblioteca o **crear uno nuevo** desde
  cero (preguntas + opciones + respuesta correcta).
- Timer 30s por pregunta (`GAME_TIMER`).
- Puntos: tiempo restante × multiplicador (más rápido = más puntos).
- Colores Kahoot por opción (rojo / azul / amarillo / verde) +
  formas (▲ ◆ ● ■).
- Cada usuario ve la pregunta y elige una opción. Al expirar el
  timer (o cuando todos contestan), pasa a resultados.
- Resultados por pregunta: animación correcto/incorrecto, puntos
  ganados, puntaje total.
- Al final: `KahootLeaderboard` con podio (🥇🥈🥉), confetti, y guarda
  `gameHistory/<uid>/<gameId>` con rank, score, totalPlayers, date
  (para el historial personal).
- Admin: → Siguiente / Finalizar antes / ⏹ Cerrar.

#### 4.5.2 Tabú Bíblico (`TabuController`)
- Cartas `TABU_CARDS` con `tipo`, `nombre` y array `prohibidas` (5
  palabras que NO se pueden decir).
- Admin elige un describidor (al azar o manual) y se le revela una
  carta. Tiene 3 min para describir el personaje/lugar/historia.
- El resto adivina en voz alta; admin marca quién adivinó (+10 pts;
  describidor +5 pts).
- Botones admin: 🔄 Otra carta, ⏭ Saltar turno, 🏆 Finalizar.
- Ranking final con confetti.

#### 4.5.3 ¿Quién Soy? (`QuienSoyController`)
- Personajes `QUIEN_SOY_PERSONAJES` con `nombre`, `testamento`
  (AT/NT), `genero` (F/M), array `pistas`.
- A un jugador se le revela el personaje (los demás NO lo ven).
- El resto hace preguntas de SÍ/NO hasta adivinar. Admin marca al
  ganador (+10 pts; jugador con personaje +3 pts).
- Admin tiene botón "👁 Mostrar respuesta" para revelarse el
  personaje sin estropearlo.

#### 4.5.4 Dibujando la Biblia (`DibujaController`,
`DibujaSetup`, `DibujaCanvas`, `DibujaDie`, `DibujaBoard`)
- Juego de tablero serpenteante con equipos.
- Admin arma equipos en `DibujaSetup` (asignación determinista,
  visible en vivo).
- Tablero `DIBUJA_BOARD` con casillas categorizadas (`DIBUJA_CATS`):
  dibujar, mímica, oral, todos participan, etc.
- Dado: todos ven el mismo resultado por timestamp.
- Lienzo en vivo (`DibujaCanvas`): el dibujante traza con mouse/touch,
  los puntos se normalizan 0..1 y se escriben en
  `sessions/<SID>/dibuja/strokes`; todos ven el dibujo en vivo.
- Categorías con colores y leyenda en el tablero.

### 4.6 T · TEMPLO (color morado `#7c3aed`, icono ⛪)
Timer 20 min (`PHASE_DURATIONS.templo = 1200`).

- Al iniciar Templo, se abre la **Encuesta semanal** (`SurveyModal`)
  para los usuarios. 3 preguntas:
  1. ¿Cuántos días estudiaste la lección esta semana? (0–7).
  2. ¿Tienes estudiantes de la Biblia? (sí/no).
  3. ¿Participaste de una acción misionera esta semana? (sí/no).
- Respuestas guardadas en `sessions/<SID>/survey/responses/<uid>` +
  copia persistente en `surveyHistory/<SID>/<uid>`.
- Admin abre `AdminSurveyModal` para ver respuestas en vivo, con
  badge "done/total" o "✓ Todos".
- Si el usuario ya respondió hoy, ve mensaje "Ya respondiste hoy".

---

## 5. BIBLIA Y LINKS EN PRESENTACIONES

- Cada slide tipo `content` tiene un array `verses` con referencias
  (ej. `"Mateo 5:22"`, `"Romanos 3:19-23"`).
- Se renderizan en `VersesBox`: caja blanca con 📖 + label "REF.
  BÍBLICA" + lista bullet de referencias.
- IDEA pendiente: convertir cada referencia en un **link clicable**
  que abra el pasaje en un visor (ej. biblegateway.com o
  bibleserver.com con la traducción Reina-Valera 1960).
  - Formato sugerido:
    `https://www.biblegateway.com/passage/?search=<encoded ref>&version=RVR1960`
  - O un modal in-app con el texto del versículo.
- Hoy son texto estático. Es una buena mejora para la próxima versión.

---

## 6. DISEÑO MRNT EN PRESENTACIONES

- Tema `SLIDE_T` (verde oliva claro) para las presentaciones del
  Repaso semanal:
  - bg: `linear-gradient(135deg,#b4c587,#c7d6a0,#e0e9c8)`
  - ink (títulos): `#3c511d`
  - accent: `#6f8f3c` (verde oliva oscuro)
- Tema `FAA` (navy + dorado) para la presentación especial "Fe más
  allá de los algoritmos".
- Marco `SlideShell` común: card grande con logo arriba, contenido
  central, footer con week/subtitle + progreso de puntitos
  (igual que la confra).
- Las cards de cada slide tienen 3 variantes: `box` (con fondo
  blanco), `plain` (sin fondo), `note` (con borde acentuado).
- Las slides también incluyen el `MRNTDiagram` cuando aplica para
  mostrar el contexto del Plan Maranata.

---

## 7. ASISTENCIA (`AttendanceModal`)

- Cada vez que un usuario entra a la sala del día, se marca en
  `attendance/<uid>/<SID>` con timestamp.
- El modal tiene 2 tabs:
  - **📅 Asistencia**: agrupa por mes, muestra check ✅ por día
    asistido + resumen (Total clases / Este año / Último mes).
  - **🏆 Juegos**: historial de juegos jugados con
    posición (🥇🥈🥉/#n), puntos, total de jugadores, fecha. Stats:
    Partidas / 🥇 Ganadas / 🏅 Top 3.
- **Racha mensual** (`STREAK_LEVELS`): iconos según sábados únicos
  del mes en curso:
  - 0 sábados → 😴
  - 1 → 🤒
  - 2 → 🙂
  - 3 → 😊
  - 4 → 🎉 ¡Mes completo!
- Admin tiene `AdminStatsModal` con estadísticas globales (todos los
  usuarios, asistencias acumuladas).
- Admin puede `DataDownloadModal` para exportar datos (CSV/Excel).

---

## 8. ACCESO A LA SALA / CONFRATERNIZACIÓN INICIAL

- Al entrar a la app: landing claro (crema) con logo de Maranata
  (triángulo amarillo + libro abierto) y botón "Entrar".
- Login/registro (`AuthModal`): email + password. Si es nuevo,
  pregunta nombre / avatar (emoji o foto) / color / clase
  (Gteen/JA). Envía correo de bienvenida HTML por EmailJS.
- Modal `GuideModal` opcional al entrar por primera vez explicando
  cómo moverse.
- Llegan a la **sala virtual del día** (`maranata-YYYY-MM-DD`):
  - QR de acceso visible en la vista de proyección para que más gente
    se sume escaneando.
  - Banner ENTRADA arriba; al moverse hacia abajo (cualquier zona
    MRNT) entran a `sala` y quedan listos para participar.
  - Cuando el admin inicia la **Confraternización**, se arma una cola
    aleatoria con TODOS los presentes y van compartiendo uno a uno
    (5–10 min por sesión completa, según cantidad de personas).
  - Las reacciones (corazones, oración, aplauso) son visibles en
    tiempo real para crear sensación de cercanía.
- Después de la confra, el admin va guiando por las otras zonas según
  el plan del día: Misión → Repaso (con presentación + videos + nube) →
  Juegos → Templo (encuesta + cierre).
- Cualquier usuario puede moverse libremente entre zonas durante el
  rato libre, lanzar reacciones, abrir su perfil o ver su asistencia.

---

## 9. RESUMEN DE LO QUE VE CADA ROL EN CADA SALA

| Sala       | Usuario común                                                                 | Admin                                                                          |
|------------|-------------------------------------------------------------------------------|---------------------------------------------------------------------------------|
| Entrada    | Avatar arrastrable, otros avatares, QR proyectado                             | Conteos por zona, botones para iniciar M/R/N/T, encuesta, stats                |
| Misión     | Form de compromiso o grid de proyectos para orar                              | Timer + conteo + ⏹ Finalizar; ve compromisos / oraciones en vivo                |
| Confra     | Avatar del speaker actual, reacciones, su turno con botón "Terminé"           | + ⏭ Saltar, ✓ Finalizar, ve la cola completa                                    |
| Repaso     | Chat, slide actual sincronizada, tags, imágenes, reacciones, nube/video      | + ◀ ▶ slides, menú Otras acciones (☁️/🎬/📲), ⏹ Finalizar                       |
| Juegos     | UI del juego activo (Kahoot/Tabú/Quien Soy/Dibuja)                            | + control de turnos, marcar adivinador, otra carta/personaje, finalizar         |
| Templo     | Encuesta semanal (3 preguntas)                                                | + ver respuestas en vivo (X/Total), abrir/cerrar encuesta, ⏹ Finalizar         |

---

## 10. CÓMO AGREGAR UN JUEGO NUEVO

1. Agregar entrada a `GAME_LIBRARY` (para Kahoots) con:
   ```js
   {id, title, subtitle, icon,
    questions:[{text, options:[a,b,c,d], correct:idx},...]}
   ```
2. Para Tabú: agregar carta a `TABU_CARDS` con
   `{tipo, nombre, prohibidas:[...5]}`.
3. Para ¿Quién Soy?: agregar a `QUIEN_SOY_PERSONAJES` con
   `{nombre, testamento, genero, pistas:[...]}`.
4. Para Dibuja: agregar casillas a `DIBUJA_BOARD` o categorías a
   `DIBUJA_CATS`.
5. Para un juego nuevo entero: crear `<Nombre>Controller`,
   añadirlo al menú `GameSelectMenu`, escribir su estado en
   `sessions/<SID>/<nombre>` y limpiar al cerrar.

---

## 11. STACK Y CONVENCIONES

- React 18 + Babel standalone (todo en un solo `index.html` ~10k
  líneas).
- Firebase: Auth (LOCAL), Realtime Database, Storage.
- EmailJS para correo de bienvenida.
- Sin build step — `index.html` se sirve estático (Netlify).
- Estilo inline con vars CSS (`--navy`, `--yellow`, `--cream`,
  `--m1/m2`, `--r1/r2`, etc.).
- Tipografías cargadas desde Google Fonts (Barlow Condensed, Barlow,
  Fraunces, Inter).
- Avatares: emoji o foto comprimida a 200px JPEG base64.
- IDs de sesión: `maranata-YYYY-MM-DD` (uno por día).
- Mobile-first; hook `useIsMobile` con breakpoint 640px.
- App version `APP_VERSION` se compara en cliente para auto-reload.
