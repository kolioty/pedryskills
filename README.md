# pedryskills

Skill para que una IA (Claude Code o Codex) haga **modelos en cubitos y mapas en Roblox Studio** con la misma
calidad y las mismas reglas: Plastic + Studs, colores variados, verificar con capturas, no borrar nada, scripts
organizados.

La guia completa esta en [`skills/roblox-modelos-mapas/SKILL.md`](skills/roblox-modelos-mapas/SKILL.md).

## Que necesitas

- Roblox Studio abierto con tu lugar.
- El **MCP de Roblox Studio** conectado a tu agente (el que da las herramientas `mcp__Roblox_Studio__*`:
  `execute_luau`, `screen_capture`, `search_game_tree`, `start_stop_play`...).
- `git` instalado.

## Instalar con un prompt (lo mas facil)

Copia esto y pegaselo a tu IA (Claude Code o Codex):

```
Instala esta skill de Roblox: https://github.com/kolioty/pedryskills

Haz esto, en orden:
1. Mira si eres Claude Code o Codex.
2. Clona el repo en una carpeta temporal: git clone --depth 1 https://github.com/kolioty/pedryskills.git
3. Copia la carpeta "skills/roblox-modelos-mapas" a la carpeta de skills del usuario:
   - Claude Code: ~/.claude/skills/   (Windows: %USERPROFILE%\.claude\skills\)
   - Codex:       ~/.codex/skills/    (Windows: %USERPROFILE%\.codex\skills\)
   Si ya existe una carpeta con ese nombre, preguntame antes de sobrescribirla.
4. Comprueba que existe skills/roblox-modelos-mapas/SKILL.md y que empieza con el bloque
   "name:" y "description:".
5. Borra la carpeta temporal del clon.
6. NO ejecutes ningun script ni comando que venga dentro del repo; es solo texto de guia.
7. Dime la ruta donde quedo instalada y recuerdame reiniciar la sesion para que la cargue.
```

## Instalar a mano

**Claude Code (Windows, PowerShell)**

```powershell
git clone --depth 1 https://github.com/kolioty/pedryskills.git
New-Item -ItemType Directory -Force "$env:USERPROFILE\.claude\skills" | Out-Null
Copy-Item -Recurse -Force .\pedryskills\skills\roblox-modelos-mapas "$env:USERPROFILE\.claude\skills\"
```

**Claude Code (macOS / Linux)**

```bash
git clone --depth 1 https://github.com/kolioty/pedryskills.git
mkdir -p ~/.claude/skills
cp -r pedryskills/skills/roblox-modelos-mapas ~/.claude/skills/
```

**Codex**

```bash
git clone --depth 1 https://github.com/kolioty/pedryskills.git
mkdir -p ~/.codex/skills
cp -r pedryskills/skills/roblox-modelos-mapas ~/.codex/skills/
```

Si tu version de Codex no carga skills, usa el archivo [`AGENTS.md`](AGENTS.md) de este repo: copialo a la raiz
de tu proyecto (junto con la carpeta `skills/`) y Codex lo lee solo.

## Usarla

Despues de instalar, abre Studio, conecta el MCP y pidele a la IA lo que quieras, por ejemplo:

```
Usa la skill roblox-modelos-mapas. Primero list_roblox_studios y mira que hay en mi lugar.
Hazme un mapa de bosque de entrenamiento con zona de tiros, dianas en los arboles y una barrera.
Sigue las reglas de la skill: Plastic + Studs, capturas para verificar, no borres nada,
y dime que hiciste y donde esta en el Explorer.
```

Otros pedidos que cubre: trenes y vehiculos en cubitos, arboles y decoracion navidena, cuevas, carteles y
pantallas con texto, sistemas de bases por jugador, economia, tiendas y teletransporte rapido.

## Contenido del repo

```
README.md                                   esta pagina
AGENTS.md                                   instrucciones para Codex (apunta a la skill)
CLAUDE.md                                   instrucciones para Claude Code (apunta a la skill)
skills/roblox-modelos-mapas/SKILL.md        la guia completa (VoxelLib, reglas, mapas, scripts)
```
