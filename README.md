# Voltaje Kart

Juego 3D de control de territorio con karts robóticos en estilo neón. Todo el juego vive en un solo archivo `index.html` (~1090 líneas), sin assets externos ni servidor.

## Cómo jugar

Abrir `index.html` en un navegador moderno (Chrome, Safari, Firefox). Funciona en escritorio y móvil.

- **Objetivo:** Pintar la mayor cantidad de casillas de tu color y mantenerlas conectadas a tu generador (torre de luz en la esquina) durante 90 segundos.
- Si un rival corta tu línea, las casillas aisladas se apagan y dejan de contar hasta que reconectes.

### Controles

| Acción       | Teclado              | Móvil (horizontal)           |
|--------------|----------------------|------------------------------|
| Manejar      | Flechas / WASD       | Joystick (pulgar izquierdo)  |
| Reversa      | Flecha abajo / S     | Joystick hacia abajo         |
| Turbo        | Espacio              | Botón TURBO (pulgar derecho) |
| Usar objeto  | F                    | Botón de objeto              |

En móvil vertical el juego rota 90 grados automáticamente para jugarse en horizontal.

---

## Robots

| Robot    | Color              | Velocidad | Giro | Pintura | Característica                |
|----------|--------------------|-----------|------|---------|-------------------------------|
| **VOLT** | Cyan `#00e5ff`     | 26        | 2.6  | Normal  | Equilibrado en todo           |
| **BRASA**| Magenta `#ff2bd6`  | 31        | 2.0  | Normal  | El más rápido, giro abierto   |
| **NODO** | Lime `#7dff3c`     | 23        | 3.4  | Normal  | Gira casi en su lugar         |
| **TANKA**| Ámbar `#ffb300`    | 20        | 2.4  | Ancho (3.2) | Lento pero pinta más ancho |

Stats visibles en menú: Velocidad (`s[0]`), Giro (`s[1]`), Pintura (`s[2]`), en escala 0-100.

## Pistas (Arenas)

Grilla de **24×24 casillas**, cada casilla = 4 unidades 3D. Generadores en las 4 esquinas `(3,3), (20,3), (20,20), (3,20)`.

| Pista        | Descripción                                          |
|--------------|------------------------------------------------------|
| **Plaza**    | Bloques dispersos en esquinas y centro               |
| **Cruz**     | Muros en forma de cruz central                       |
| **Columnas** | 4 grupos de columnas en cuadrantes + centro          |
| **Anillo**   | Muro rectangular con centro abierto                  |

## Ítems y mecánicas

### Turbo
- Se carga pintando casillas (+0.035 por casilla, lleno = 1.0)
- Al activar: velocidad ×1.65 durante 1.8 segundos
- Chocar con boost a un rival lo aturde 1.3 segundos

### Pulsos (octaedros blancos)
- Aparecen cada 4.5 seg (máx 3 activos)
- Al recoger: pintan un área circular de radio 2.6 bloques del color del jugador
- Generan efecto de anillo visual + sonido

### Objetos recogibles (aparecen cada 5 seg, máx 3 activos)
| Objeto    | Probabilidad | Efecto                                        |
|-----------|--------------|-----------------------------------------------|
| **Balas** | 55%          | 3 disparos, aturden 1.2 seg, cooldown 0.28s   |
| **Bomba** | 45%          | Mina de área, radio detonación ~3.5, daño ~8, aturde 2 seg |

## IA (3 niveles de dificultad)

| Nivel       | Velocidad | Ruido | Corte | Embestida | Puntería |
|-------------|-----------|-------|-------|-----------|----------|
| **Normal**  | ×0.95     | 6     | 0     | No        | 0.04     |
| **Difícil** | ×1.03     | 3     | 6     | Sí        | 0.20     |
| **Experto** | ×1.10     | 1     | 14    | Sí        | 0.50     |

La IA usa **BFS** para rutas alrededor de bloques y evalúa targets con una función de costo que considera: distancia, ángulo, prioridad del líder, cercanía al generador rival, y ruido aleatorio.

Comportamientos: buscar casillas frontera propias para expandir, cortar líneas del líder (más agresivo en niveles altos), recoger pulsos e ítems, embestir con turbo, y recuperarse si queda atascada.

---

## Arquitectura del código

Todo está en `index.html`: CSS inline + HTML + un solo `<script>` con Three.js (CDN r128).

### Estructura del script (líneas aproximadas)

```
229-265   Datos: ROBOTS, DIFF, ARENAS, SPAWN, arrays de estado
267-343   Escena Three.js: renderer, cámara, piso con CanvasTexture, muros, bloques
345-412   Modelos 3D: buildKart(), generadores con partículas aditivas
415-503   Audio: Web Audio API, síntesis de SFX y música (4 compases Am-F-C-G)
505-566   Lógica de pintura y energía: paint(), power() (BFS flood-fill)
575-602   startGame(): inicialización de partida
604-698   Ítems: spawnPick(), spawnItem(), stepItems(), useItem()
700-782   IA: route() (BFS), think() (decisiones), clearTo() (visibilidad)
785-855   Simulación: moveKart() (física + colisiones), step() (game loop)
857-917   Renderizado: sync() (actualiza meshes), hud() (actualiza UI)
920-938   finish(): pantalla de resultados
940-1088  Menú, controles, joystick, teclado, boot(), loop principal
```

### Estado global principal

- **`S`** — Estado del juego: `mode`, `t` (timer), `karts[]`, `picks[]`, `items[]`, `shots[]`, `mines[]`, `fx[]`, `beams[]`, dificultad, arena, etc.
- **`input`** — Estado de entrada: `l/r/u/d`, `turbo`, `fire`, `sx/sy` (joystick), `touch`
- **`owner[]`** — `Int8Array(N×N)`: dueño de cada casilla (-1 = nadie, 0-3 = robot)
- **`powered[]`** — `Uint8Array(N×N)`: 1 si la casilla está conectada al generador de su dueño
- **`blocked[]`** — `Uint8Array(N×N)`: 1 si es un bloque/obstáculo
- **`baseOf[]`** — `Int8Array(N×N)`: indica si una casilla es generador de algún robot

### Cada kart (objeto en `S.karts[]`)

```
id, ch (robot data), ci (robot index), x, z, ang, v (velocidad),
turbo (carga 0-1), boostT (tiempo restante boost), stunT (aturdimiento),
spin, score (casillas energizadas), total (casillas pintadas),
item ("bala"/"bomba"/null), ammo, fireCd,
me (true si es el jugador), mesh (Three.js Group),
// Solo IA:
li, lj (última casilla), tx, tz, tk (target), reT (re-evaluate timer),
stuckT, revT, revS, path, pt
```

### Constantes clave

| Constante | Valor | Descripción                        |
|-----------|-------|------------------------------------|
| `N`       | 24    | Casillas por lado                  |
| `T`       | 4     | Unidades 3D por casilla            |
| `HALF`    | 48    | Mitad del tablero (N×T/2)          |
| `DUR`     | 90    | Duración de partida en segundos    |
| `TEX`     | 1024  | Tamaño de la textura del piso      |
| `CELL`    | ~42.7 | Píxeles por casilla en la textura  |
| `CR`      | 1.4   | Radio de colisión del kart         |

### Sistemas principales

1. **Pintura y energía:** Cada casilla tiene un dueño. `power()` hace BFS desde cada generador para marcar qué casillas están conectadas. Solo las conectadas suman puntos.

2. **Física:** Cada kart tiene 2 círculos de colisión (frente y cola). `pushOut()` resuelve contra muros y bloques con corrección de penetración. Los karts se empujan entre sí.

3. **Piso dinámico:** Canvas de 1024×1024 como textura Three.js. Cada casilla se redibuja solo cuando cambia de estado (owner × powered). Estilo visual: energizada = borde sólido, desconectada = borde punteado.

4. **Audio sintetizado:** Sin archivos de audio. Todo con osciladores Web Audio API: motor del kart (sawtooth filtrado), SFX (combinaciones de sine/square/triangle/noise), música procedural (4 acordes Am-F-C-G en loop de 64 pasos).

5. **Cámara:** Sigue al jugador con interpolación suave. En menú orbita el tablero. En boost se aleja. En resultado se eleva.

### Debug

El juego expone `window.__vk` con: `setArena`, `S` (estado), `step`, `startGame`, `owner`, `powered`, `FREE`.

---

## Dependencias

- [Three.js r128](https://cdnjs.cloudflare.com/ajax/libs/three.js/r128/three.min.js) (CDN)
- [Google Fonts: Chakra Petch](https://fonts.googleapis.com/css2?family=Chakra+Petch:wght@400;600;700) (CDN)

No requiere npm, bundler, ni servidor. Solo un navegador con WebGL.
