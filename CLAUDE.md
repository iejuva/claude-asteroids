# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Cómo correr

Sin instalación ni build. Abre `index.html` directamente en el navegador, o sirve el directorio:

```bash
npx serve .
```

Visita `http://localhost:3000`.

## Controles

| Tecla | Acción |
|-------|--------|
| ← / → | Rotar nave |
| ↑ | Propulsar |
| Espacio | Disparar / reiniciar (game over) |
| B | Activar Bomba Nova (si hay stock) |
| — | Slow Motion (auto al recoger el rombo cian) |

## Arquitectura

Todo el juego vive en **dos archivos**: `index.html` (canvas 800×600 + estilos mínimos) y `game.js` (lógica completa). No hay bundler, frameworks ni dependencias.

### game.js — estructura interna

El archivo sigue un orden vertical claro:

1. **Input** — `keys` y `justPressed` capturan teclado; `pressed(code)` consume un evento one-shot.
2. **Utils** — `wrap`, `dist`, `rand`, `randInt`.
3. **Clases de entidades** — cada una tiene `update(dt)` y `draw()` y un flag `dead` para eliminación lazy:
   - `Bullet` — proyectil con TTL.
   - `Asteroid` — polígono irregular; `split()` devuelve dos fragmentos de `size - 1`.
   - `Ship` — física con arrastre (`DRAG = 0.987`), invencibilidad temporal al reaparecer.
   - `Particle` — chispa de explosión con alpha decreciente.
   - `PowerUp` — ítem recogible (rombo parpadeante); `ttl=30`, desaparece si no se recoge. Tipo `'nova'` (dorado) o `'slow'` (cian).
4. **Estado global** — `ship`, `bullets`, `asteroids`, `particles`, `powerups`, `score`, `lives`, `level`, `novaCount`, `slowTimer`, `state` (`'playing' | 'dead' | 'gameover'`), `deadTimer` (temporizador de reaparición).
5. **Funciones de juego** — `initGame`, `nextLevel`, `spawnAsteroids`, `explode`, `killShip`.
6. **`update(dt)`** — máquina de estados principal; aplica física, detecta colisiones (radio circular), gestiona transiciones.
7. **`draw()`** — limpia canvas, dibuja entidades, HUD y overlay.
8. **Game loop** — `requestAnimationFrame`; `dt` clampeado a 50 ms para evitar saltos grandes.

### Constantes clave

| Constante | Valor | Descripción |
|-----------|-------|-------------|
| `W / H` | 800 / 600 | Tamaño del canvas |
| `RADII` | [0, 16, 30, 50] | Radio por tamaño de asteroide (1–3) |
| `SPEEDS` | [0, 85, 55, 32] | Velocidad base por tamaño |
| `POINTS` | [0, 100, 50, 20] | Puntos por tamaño |

### Colisión y eliminación

Las entidades se marcan `dead = true` durante el update y se filtran al final del frame con `.filter(x => !x.dead)`. Los asteroides destruidos generan fragmentos vía `split()` que se concatenan al array en el mismo frame.

### Power-ups

Sistema extensible en `game.js`. Flujo: spawn al destruir asteroide → pickup por colisión nave-ítem → efecto inmediato o activación por tecla.

| Power-Up | Tecla | Spawn | Estado |
|----------|-------|-------|--------|
| Bomba Nova | `B` | 12% al destruir size=3 | Implementado |
| Slow Motion | auto | 12% size=3 (aleatorio con nova) · 2% size<3 (siempre slow) | Implementado |
| Disparo Triple | — | — | Pendiente |
| Escudo Temporal | — | — | Pendiente |
| Hiperpropulsión | — | — | Pendiente |

**Slow Motion — patrón de implementación:**
- Usa `effectiveDt` escalado para entidades afectadas; entidades exentas reciben `dt` real.
- `Ship.update(dt, rotDt)` acepta un segundo parámetro para desacoplar giro del resto de la física.
- Ver detalles de factores y spawn en `todo-funcionalidades/seguimiento-powerups.md`.

**Para añadir un power-up nuevo — checklist:**
1. Clase tras `Particle` en `game.js` (o reutilizar `PowerUp` con nuevo `type`).
2. Global de estado/timer junto a `novaCount` y `slowTimer`.
3. Reset del global en `initGame()`, `nextLevel()` y `state === 'dead'` si aplica.
4. Spawn en colisión bala-asteroide (buscar `powerups.push`).
5. Pickup en el loop nave-power-up (buscar `dist(ship, p)`); branch por `p.type`.
6. Efecto en `update()` y visual en `draw()` / `drawHUD()`.

### Gotchas

- **`keys` y `justPressed`**: deben declararse como `{}` antes de los event listeners. Si el juego no responde a teclado, verificar que estén presentes al inicio de la sección Input (se eliminaron accidentalmente en el commit `13e713f`).
- La colisión nave-asteroide usa `a.radius * 0.82` (no el radio completo) para hacer el hitbox más justo visualmente.
- Activar Bomba Nova destruye todos los asteroides y avanza el nivel (misma condición de victoria que disparo normal).
- `PowerUp` recibe `type` opcional; si es `null`, elige aleatoriamente entre los tipos implementados. Para forzar un tipo al spawn: `new PowerUp(x, y, 'tipo')`.
