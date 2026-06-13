# Handoff — Rediseño de la sala virtual MRNT Class

> Para: **Claude Code**
> Desde: Diseño aprobado en mockup (`Maranata Class.html`)
> Objetivo: Portar el nuevo diseño de la **sala virtual** a la app real (Firebase + React) sin romper la lógica existente.

---

## 🎯 Qué cambia

El **canvas de la sala** (la vista donde los avatares se mueven entre zonas) cambia completamente:

- **Antes**: fondo navy oscuro, 5 zonas en columnas, sin estructura visual del plan MRNT.
- **Ahora**: fondo crema (igual al landing), un **diagrama MRNT central** con los 4 pétalos del Plan Maranata, y **6 zone cards** posicionadas radialmente alrededor del diagrama, cada una a la altura de su letra correspondiente.

El resto de la app (auth, confraternización, repaso, juegos, kahoot, wordcloud, chat, admin panel) **se mantiene idéntica**.

---

## 📋 Checklist de cambios

### ✅ Portar desde el mockup
- [ ] Componente `Petal` (helper SVG)
- [ ] Componente `MRNTDiagram` (con prop nuevo `hideCenterDot`)
- [ ] Componente `RoomCanvas` completo (rediseñado)
- [ ] Constante `ZONES` (con nuevas coordenadas y la nueva zona `templo`)
- [ ] Función `detectZone(x, y)` (sin cambios pero confirmar que sigue funcionando con las nuevas coords)
- [ ] Fondo crema `#f8f5ef` para el contenedor de la sala

### ✅ Añadir nueva zona "Templo"
- [ ] Nueva clave `templo` en `ZONES`
- [ ] Añadir `templo: <duración>` a `PHASE_DURATIONS` (sugerencia: `templo: 1200` segundos)
- [ ] Decidir si la zona Templo tendrá su propia actividad (oración, lectura bíblica, etc.) o será solo decorativa por ahora
- [ ] Si tiene actividad: añadir handlers `onStartTemplo`, `onEndTemplo` en el admin panel
- [ ] Validar que Firebase RTDB acepta `phase: "templo"` (no debería haber problema, pero confirma reglas)

### ✅ NO TOCAR
- [ ] Firebase config y reglas RTDB
- [ ] Lógica de auth (login con nombre, avatar, color)
- [ ] Componentes: `ConfraSpeakerPanel`, `RepasoChat`, `KahootController`, `WordCloudActivity`, `RepasoChatScreen`
- [ ] Estructura de datos en RTDB (`/users`, `/sessions`, `/messages`, etc.)
- [ ] El header / topbar de la app
- [ ] El admin panel (solo añadir el botón de "Iniciar Templo" si decides que la zona es interactiva)

---

## 🎨 Sistema visual

### Colores MRNT (oficiales del plan)

| Letra | Significado | Zona(s)            | Color principal | Color oscuro |
|-------|-------------|--------------------|-----------------|--------------|
| **M** | Misión      | `mision`           | `#3b82f6`       | `#1d4ed8`    |
| **R** | Relación    | `confra`           | `#eab308`       | `#a16207`    |
| **N** | Nutrición   | `repaso`           | `#fb923c`       | `#c2410c`    |
| **N** | Nutrición   | `juegos`           | `#f97316`       | `#c2410c`    |
| **T** | Templo      | `templo`           | `#7c3aed`       | `#5b21b6`    |
| —     | Llegada     | `entrada`          | `#0a0e2e` (navy)| `#0a0e2e`    |

### Tipografía
- **Display / títulos**: `Barlow Condensed`, weight 900, tracking `.04–.18em`, uppercase
- **Subtítulos**: `Barlow Condensed`, weight 700, tracking `.18–.22em`, uppercase
- **Body / nombres**: `Barlow Condensed` o `Barlow`, weight 600–700

### Fondos
- **Sala (canvas)**: `#f8f5ef` (cream) con gradientes radiales sutiles
- **Cards de zona**: tinte transparente del color de la zona (`rgba(color, 0.13)` reposo, `0.22` activa)
- **Card "Entrada"**: fondo blanco con borde navy

### Bordes
- Reposo: 3px sólido, color saturado del MRNT
- Activa: 4px + sombra de color + halo `0 0 0 5px {color}22`

---

## 📦 Snippets a portar

### 1. Constantes (línea ~119–172 del mockup)

```js
const REACTIONS = [
  {id:'heart',e:'❤️'},{id:'pray',e:'🙏'},{id:'thumbs',e:'👍'},
  {id:'smile',e:'😊'},{id:'cry',e:'😢'},{id:'party',e:'🎉'}
];

// Zonas de la sala — mapeadas al Plan MRNT
// Entrada arriba; M-izq, R-der, N-der-abajo, T-izq-abajo, juegos extra
const ZONES = {
  entrada: {
    label:'Entrada', letter:'', mrnt:'Llegada',
    color:'#0a0e2e', colorDark:'#0a0e2e',
    bg:'rgba(10,14,46,.06)', icon:'🚪',
    cardX:38, cardY:3, cardW:24, cardH:9,
    x1:36, x2:64, y1:1, y2:14
  },
  mision: {
    label:'Misión', letter:'M', mrnt:'Misión',
    color:'#3b82f6', colorDark:'#1d4ed8',
    bg:'rgba(59,130,246,.10)', icon:'✈️',
    cardX:2, cardY:24, cardW:22, cardH:22,
    x1:0, x2:26, y1:18, y2:50
  },
  confra: {
    label:'Confraternización', letter:'R', mrnt:'Relación',
    color:'#eab308', colorDark:'#a16207',
    bg:'rgba(234,179,8,.10)', icon:'🤝',
    cardX:76, cardY:24, cardW:22, cardH:22,
    x1:74, x2:100, y1:18, y2:50
  },
  repaso: {
    label:'Repaso Bíblico', letter:'N', mrnt:'Nutrición',
    color:'#fb923c', colorDark:'#c2410c',
    bg:'rgba(251,146,60,.10)', icon:'📖',
    cardX:76, cardY:54, cardW:22, cardH:22,
    x1:74, x2:100, y1:50, y2:80
  },
  juegos: {
    label:'Juegos', letter:'N', mrnt:'Nutrición',
    color:'#f97316', colorDark:'#c2410c',
    bg:'rgba(249,115,22,.10)', icon:'🎮',
    cardX:62, cardY:84, cardW:22, cardH:13,
    x1:60, x2:84, y1:80, y2:98
  },
  templo: {
    label:'Templo', letter:'T', mrnt:'Templo',
    color:'#7c3aed', colorDark:'#5b21b6',
    bg:'rgba(124,58,237,.10)', icon:'⛪',
    cardX:2, cardY:54, cardW:22, cardH:22,
    x1:0, x2:26, y1:50, y2:80
  },
};

// Duraciones de fase (segundos)
const PHASE_DURATIONS = {
  confra: 600,    // 10 min
  repaso: 1800,   // 30 min
  juegos: 900,    // 15 min
  mision: 1200,   // 20 min
  templo: 1200,   // 20 min — NUEVO
};

const detectZone = (x, y) => {
  for (const [key, z] of Object.entries(ZONES)) {
    if (x >= z.x1 && x <= z.x2 && y >= z.y1 && y <= z.y2) return key;
  }
  return 'entrada';
};
```

### 2. Componentes `Petal` y `MRNTDiagram`

Cópialos **textuales** desde el mockup `Maranata Class.html`:
- `Petal`: línea ~450
- `MRNTDiagram`: línea ~462–550 (asegúrate de incluir la prop `hideCenterDot`)

> **Nota crítica**: el SVG dentro de `MRNTDiagram` debe usar `width="100%" height="100%"` (NO `width={size}`) para que el diagrama se centre correctamente en el contenedor del canvas.

### 3. Componente `RoomCanvas`

Cópialo **textual** desde el mockup, líneas ~626–844 aprox. Es el componente principal a reemplazar.

Este componente recibe:
```jsx
<RoomCanvas
  users={users}            // dict {uid: {name, x, y, zone, color, ...}}
  myId={currentUserId}
  session={session}        // {phase, ...}
  onDragMove={(x, y) => updateMyPosition(x, y)}
  roomRef={ref}
  full={isProjectionMode}  // true para vista pantalla completa
  reactions={reactions}
/>
```

---

## 🤔 Decisiones que necesito de ti

Antes de que Claude Code empiece, decide:

1. **¿La zona Templo es interactiva o decorativa?**
   - Si es interactiva: ¿qué actividad ocurre? (oración guiada, lectura, momento de silencio…)
   - Si es decorativa: solo con que los avatares puedan ir ahí basta — no cambies el admin panel.

2. **¿Tu RTDB ya tiene datos con `phase: 'entrada'` u otros valores históricos?**
   - Si sí: ojo, los usuarios viejos pueden tener `zone: 'entrada'` que ahora está en otra posición. Esto es OK porque solo afecta la posición x/y inicial.

3. **¿La proporción del canvas en tu app real es 16:9 (como el mockup) o distinta?**
   - Si es distinta, las coordenadas `cardX/Y/W/H` y `x1/x2/y1/y2` necesitan ajuste proporcional.

---

## 🚀 Pasos sugeridos para Claude Code

1. **Leer** `Maranata Class.html` adjunto.
2. **Identificar** dónde está `RoomCanvas` actual en el repo de la app real.
3. **Crear branch** `feat/sala-rediseño-mrnt`.
4. **Reemplazar** los componentes y constantes listados arriba.
5. **Probar local** con varios usuarios simultáneos:
   - Login OK
   - Drag de avatar entre zonas funciona
   - Cambios de fase (admin → confra → repaso → etc.) resaltan la zona correcta y el cuadrante MRNT correspondiente
   - Vista de proyección (`full={true}`) se ve bien
6. **Confirmar** que todas las actividades (confra, repaso, juegos, mision) siguen abriendo sus modales correctamente.
7. **Deploy** a un preview URL antes de mergear.

---

## 📎 Archivos adjuntos

- `Maranata Class.html` — el mockup completo (source of truth visual)
- `HANDOFF.md` — este documento

---

¿Dudas? Pregúntame antes de modificar lógica de Firebase o componentes que no estén en la lista de "Portar".
