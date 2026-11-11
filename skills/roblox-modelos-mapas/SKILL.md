---
name: roblox-modelos-mapas
description: Construir modelos en cubitos (voxeles) y mapas completos en Roblox Studio desde un agente de IA usando el MCP de Roblox Studio. Usala cuando pidan hacer modelos (trenes, arboles, cuevas, carteles, pantallas), mapas (bosques, zonas de tiro, lobbies), decorar con Plastic + Studs, o armar sistemas de scripts organizados (bases, economia, tiendas, teletransporte). Incluye la libreria VoxelLib, las reglas de calidad y la lista de comprobacion.
---
# Como hacer modelos y mapas en Roblox Studio con Claude (y el MCP de Studio)

Esta guia explica **la tecnica** que se uso para hacer trenes, arboles, cuevas, tablas de tops y un mapa
completo en Roblox Studio, todo desde Claude Code conectado al MCP de Studio. Sirve para pasarsela a otro
Claude (o a un amigo) y que haga el mismo tipo de trabajo con la misma calidad.

---

## 0. Prompt listo para pegar (copialo tal cual)

```
Vas a construir cosas en Roblox Studio usando el MCP de Roblox Studio (herramientas mcp__Roblox_Studio__*).
Lee la guia README.md de esta carpeta ENTERA antes de tocar nada y sigue sus reglas.

Reglas resumidas:
1. Primero list_roblox_studios (el studio_id cambia cuando se reabre Studio) y mira que hay con
   search_game_tree / execute_luau ANTES de construir.
2. Todo lo que construyas es Material Plastic + MaterialVariant "Studs", anclado, con colores variados
   (nada plano ni simetrico perfecto).
3. Los modelos organicos o complejos se hacen en "cubitos" (voxeles) con la libreria VoxelLib de la guia
   (seccion 3), no con 3 bloques grandes.
4. Despues de CADA cambio: screen_capture desde 2 o 3 angulos, mira que se vea bien, corrige, y recien
   entonces avisa. Nunca digas "listo" sin haberlo visto.
5. No borres nada del usuario: archivalo (muevelo a ServerStorage) y avisa. Deja un waypoint de deshacer
   (ChangeHistoryService:SetWaypoint) antes y despues de cada cambio grande.
6. Haz SOLO lo que se pide. Si hay dos lecturas posibles, elige la mas probable y dile en una frase
   cual elegiste y como cambiarla.
7. Responde en el idioma del usuario, corto y claro, diciendo QUE hiciste y DONDE esta en el Explorer.

Tarea: <aqui pones lo que quieres, por ejemplo "hazme un mapa de bosque con zona de tiros y dianas">
```

---

## 1. Antes de empezar: conectarse y mirar

| Paso | Herramienta | Para que |
|---|---|---|
| 1 | `list_roblox_studios` | Saca el `studio_id` y el nombre del lugar. Cambia cada vez que se reabre Studio. |
| 2 | `search_game_tree` (con `path`, `keywords`, `max_depth`) | Ver que hay. **No** pidas el arbol completo de StarterGui de un pack grande: se cuelga. |
| 3 | `execute_luau` con `datamodel_type: "Edit"` | Leer propiedades, medir y construir con codigo. |
| 4 | `screen_capture` con `camera_position` y `look_at_position` | Ver el resultado. Usa `capture_id` distinto cada vez. |
| 5 | `start_stop_play` + `execute_luau` en `Server`/`Client` + `get_console_output` | Probar de verdad (spawn, scripts, GUI). |

Cosas que muerden:

- `execute_luau` **solo devuelve lo que pongas en `return`**. `print` no se ve. Arma un string y devuelvelo.
- Si una llamada tarda mas de ~2 min se mueve a segundo plano y falla por timeout. Trocea el trabajo
  (`task.wait()` entre bloques) y sube el parametro `timeout` (hasta 600000).
- El primer `screen_capture` a veces no pinta el texto de un `SurfaceGui`: repite la captura.
- Si una captura da timeout, cambia de vista y reintenta.
- `require` de un ModuleScript desde `execute_luau` crea **otra instancia** distinta de la del juego. Para
  probar un sistema, lee atributos que el sistema pone, no hagas `require` para ver su estado.
- `workspace.StreamingEnabled` puede estar activo: el cliente no ve partes lejanas. Lo que dependa de
  "esta cerca" mejor medirlo en el servidor.
- Antes de editar algo que no hiciste tu, **lee** como esta (el usuario mueve y rehace cosas a mano).

---

## 2. Reglas de calidad (lo que hace que se vea "profesional" y no "de IA")

1. **Material**: `Material = Plastic`, `MaterialVariant = "Studs"` en todo. Las luces/ventanas/ojos brillantes
   en `Neon` (sin variante).
2. **Color con variacion**: cada pieza con +-3..6% de brillo (`vary`). Paletas de 3 a 5 tonos por material,
   no un solo color.
3. **Nada perfectamente simetrico**: carteles torcidos 1.5-4 grados, tablas de ancho distinto, huecos a proposito.
4. **Contraste**: oscuro + un color vivo (neon) para que "lea" bien. Cuidado: el neon tiene *bloom*, usa
   piezas chicas o se tapa la vista.
5. **Mira la iluminacion del lugar**: con mucha exposicion los grises se ven casi blancos. Si pasa, oscurece
   la paleta (multiplica el valor HSV por 0.45) en vez de tocar el Lighting global.
6. **Evita z-fighting**: dos caras de distinto color en el mismo plano parpadean. Separa 0.05-0.12 studs o
   no las pongas coplanares. Piezas encima del piso: que su base no coincida con la cara superior del piso.
7. **Texto de carteles**: `SurfaceGui` + `TextLabel`, fuente `FredokaOne`, sin `UIStroke` si el usuario no lo
   quiere, `LightInfluence = 0`. Un `Part` normal mira su cara *Front* hacia -Z. Si el que mira esta en +Z,
   la cara que ve es **Back**. Para que se lea desde los dos lados, pon `SurfaceGui` en Front y Back.
8. **Escala**: pregunta o mide el tamano del personaje del juego (en este caso eran ~11 studs de alto).
   Puertas, mostradores y pasillos tienen que ser proporcionales.
9. **Todo anclado**. Y organizado en carpetas/modelos con nombres claros (`Pantalla`, `Raiz`, `Cartel`...).

---

## 3. Modelos en cubitos (voxeles) con VoxelLib

Es la tecnica mas importante. En vez de poner bloques a mano, se "dibuja" en una cuadricula 3D de celdas
(`x, y, z` enteros) con formas (cajas, cilindros, elipsoides, tubos) y al final la libreria **fusiona las celdas
del mismo color en bloques de hasta 4x4x4** y solo deja la superficie (borra el interior). El resultado se ve
como Lego, con studs, y tiene pocas piezas.

### 3.1 Libreria (pegala en un ModuleScript temporal y haz `require`)

```lua
-- VoxelLib: celdas -> Parts fusionadas. Rango de coordenadas: x -256..255, y -32..223, z -128..127
local L = {}
local pal = {}
-- color: L.col(0xRRGGBB) o L.col(0xRRGGBB, "N") para neon
function L.col(h, mat)
  local key = h .. (mat or "")
  if not pal[key] then
    pal[key] = { c = Color3.fromRGB(bit32.rshift(h,16)%256, bit32.rshift(h,8)%256, h%256), neon = (mat == "N") }
  end
  return pal[key]
end
local G = {}
G.__index = G
function L.grid() return setmetatable({ c = {}, pos = {} }, G) end
local function K(x,y,z) return ((x+256)*256 + (y+32))*256 + (z+128) end

function G:set(x,y,z,e)           -- e = nil borra la celda
  local k = K(x,y,z)
  if e == nil then self.c[k] = nil; self.pos[k] = nil else self.c[k] = e; self.pos[k] = {x,y,z} end
end
function G:get(x,y,z) return self.c[K(x,y,z)] end
function G:box(x0,x1,y0,y1,z0,z1,e)
  for x=x0,x1 do for y=y0,y1 do for z=z0,z1 do self:set(x,y,z,e) end end end
end
function G:cyl(a,b,r,t0,t1,e,r0)  -- cilindro VERTICAL centro (a,b)=(x,z), radio r, alturas t0..t1, r0 = hueco
  local ra = math.ceil(r)+1
  for u = math.floor(a-ra), math.ceil(a+ra) do for v = math.floor(b-ra), math.ceil(b+ra) do
    local d = math.sqrt((u+.5-a)^2 + (v+.5-b)^2)
    if d <= r and (not r0 or d >= r0) then for t = t0, t1 do self:set(u,t,v,e) end end
  end end
end
function G:ell(cx,cy,cz,rx,ry,rz,e,clip) -- elipsoide; clip(x,y,z)->bool para recortar o "pintar solo lo que existe"
  for x = math.floor(cx-rx), math.ceil(cx+rx) do for y = math.floor(cy-ry), math.ceil(cy+ry) do for z = math.floor(cz-rz), math.ceil(cz+rz) do
    local d = ((x+.5-cx)/rx)^2 + ((y+.5-cy)/ry)^2 + ((z+.5-cz)/rz)^2
    if d <= 1 and (not clip or clip(x,y,z)) then self:set(x,y,z,e) end
  end end end
end
function G:line(ax,ay,az,bx,by,bz,r0,r1,e) -- tubo conico (cuernos, tentaculos, ramas, huesos)
  local dx,dy,dz = bx-ax, by-ay, bz-az
  local len = math.sqrt(dx*dx+dy*dy+dz*dz)
  local steps = math.max(1, math.ceil(len*2))
  for i = 0, steps do
    local t = i/steps
    local cx,cy,cz = ax+dx*t, ay+dy*t, az+dz*t
    local r = math.max(r0+(r1-r0)*t, 0.8)
    local ri = math.ceil(r); local r2 = r*r
    for x = math.floor(cx-ri), math.ceil(cx+ri) do for y = math.floor(cy-ri), math.ceil(cy+ri) do for z = math.floor(cz-ri), math.ceil(cz+ri) do
      if (x+.5-cx)^2 + (y+.5-cy)^2 + (z+.5-cz)^2 <= r2 then self:set(x,y,z,e) end
    end end end
  end
end
function G:recolor(from, fn)       -- cambia el color de las celdas de un color segun posicion (camuflaje, rayas, nieve)
  for k, p in pairs(self.pos) do
    if self.c[k] == from then local ne = fn(p[1],p[2],p[3]); if ne then self.c[k] = ne end end
  end
end

local MAXR = 4
-- build(parent, origin(CFrame), V(tamano de celda en studs), seed) -> numero de partes
function G:build(parent, origin, V, seed)
  local rnd = Random.new(seed or 1)
  local c, pos = self.c, self.pos
  local rm = {}
  for k, p in pairs(pos) do                      -- 1) quitar celdas totalmente rodeadas (interior)
    local x,y,z = p[1],p[2],p[3]
    if c[K(x+1,y,z)] and c[K(x-1,y,z)] and c[K(x,y+1,z)] and c[K(x,y-1,z)] and c[K(x,y,z+1)] and c[K(x,y,z-1)] then rm[#rm+1] = k end
  end
  for _, k in ipairs(rm) do c[k] = nil; pos[k] = nil end
  local keys = {}
  for k in pairs(c) do keys[#keys+1] = k end
  table.sort(keys)
  local vis, n = {}, 0
  for _, k in ipairs(keys) do                    -- 2) fusionar celdas del mismo color en cajas (greedy)
    if not vis[k] then
      local e, p = c[k], pos[k]
      local x,y,z = p[1],p[2],p[3]
      local function ok(xx,yy,zz) local kk = K(xx,yy,zz); return c[kk] == e and not vis[kk] end
      local x1 = x
      while x1-x+1 < MAXR and ok(x1+1,y,z) do x1 += 1 end
      local z1 = z
      while z1-z+1 < MAXR do
        local good = true
        for xx = x, x1 do if not ok(xx,y,z1+1) then good = false break end end
        if not good then break end
        z1 += 1
      end
      local y1 = y
      while y1-y+1 < MAXR do
        local good = true
        for xx = x, x1 do for zz = z, z1 do if not ok(xx,y1+1,zz) then good = false break end end if not good then break end end
        if not good then break end
        y1 += 1
      end
      for xx = x, x1 do for yy = y, y1 do for zz = z, z1 do vis[K(xx,yy,zz)] = true end end end
      local part = Instance.new("Part")
      part.Anchored = true
      part.Size = Vector3.new(x1-x+1, y1-y+1, z1-z+1) * V
      part.CFrame = origin * CFrame.new((x+x1+1)/2*V, (y+y1+1)/2*V, (z+z1+1)/2*V)
      local h, s, v = e.c:ToHSV()
      if e.neon then part.Material = Enum.Material.Neon; part.Color = e.c
      else
        part.Material = Enum.Material.Plastic; part.MaterialVariant = "Studs"
        part.Color = Color3.fromHSV(h, s, math.clamp(v * (1 + (rnd:NextNumber()-0.5)*0.10), 0, 1)) -- variacion por pieza
      end
      part.TopSurface = Enum.SurfaceType.Smooth; part.BottomSurface = Enum.SurfaceType.Smooth
      part.Name = "Cubo"; part.Parent = parent
      n += 1
    end
  end
  return n
end
return L
```

### 3.2 Como se usa

```lua
local L = require(game.ServerStorage.VoxelLib)    -- el ModuleScript temporal con el codigo de arriba
local c = L.col
local ROJO, NEGRO, ORO, NEON = c(0xB81A1F), c(0x14090E), c(0xF2B833), c(0xFF2A2A, "N")

local g = L.grid()
g:box(-20,19, 3,4, -5,4, NEGRO)                    -- chasis
g:cyl(10, 0, 4.9, 5, 20, ROJO)                     -- caldera/cilindro vertical
g:ell(0, 15, 0, 6, 5, 6, ORO)                      -- cupula
g:line(0,10,0, 8,20,3, 3,0.8, ROJO)                -- cuerno curvo (tubo conico)

local modelo = Instance.new("Model"); modelo.Name = "MiModelo"
local n = g:build(modelo, CFrame.new(0, 4, 0), 1, 7)   -- V=1 (celda de 1 stud); V=0.5 = mas detalle
modelo.Parent = workspace
-- al terminar: borra el ModuleScript temporal para no dejar basura
```

Tamano de celda `V`: `1` para cosas grandes (arboles, cuevas, tablas), `0.5` para trenes y personajes detallados
(unas 5 veces mas partes: un tren grande llega a 8.000-11.000 partes; vigila el rendimiento).

### 3.3 Receta para que un modelo salga bien

1. **Planifica por capas**: chasis -> cuerpo -> techo -> detalles -> luces -> decoracion.
2. **Silueta primero** con formas grandes (`box`, `cyl`, `ell`), **detalle despues** (ventanas, ruedas, dientes).
3. **Simetria barata**: dibuja un lado y espeja a mano (`-x` o `z -> -1-z`).
4. **Pinta despues de modelar**: `recolor` por posicion para rayas, camuflaje, escamas, nieve
   (por ejemplo: celdas del color verde que no tienen nada encima -> blancas = nieve).
5. **Decals sobre superficie**: recorre `g.pos`, y a las celdas del borde exterior que caen en un dibujo
   (numero, copo de nieve, calavera) cambiales el color. Sirve para ventanas, numeros y emblemas.
6. **Ojos/luces**: `ell` pequeno con `clip = function(x,y,z) return g:get(x,y,z) ~= nil end` para pintar al ras
   sin que sobresalga, y un neon mas chico adentro.
7. **Ruedas**: aro + placa + buje + radios (10 radios cada 36 grados) + tapa de neon; repetir en los dos lados.
8. Cada vehiculo/objeto: `Model` con una `Raiz` (Part invisible, `PrimaryPart`) para poder moverlo/soldarlo despues.
   Pon atributos utiles (`Largo`, `Desfase`...).
9. **Verifica con 3 capturas** (lado, frente, 3/4). Si algo flota, se ve plano o no se entiende, corrige antes de avisar.

---

## 4. Mapas (terreno, bosque, estructuras)

### 4.1 Plan

1. Decide el **eje largo** del mapa y la **zona del jugador** (donde aparece). Mide el suelo existente
   (`Part.Position`, `Size`) y calcula coordenadas locales con un origen (`X0, Z0, GY` = suelo).
2. Crea una carpeta raiz con **subcarpetas por tipo**: `Suelo`, `Arboles`, `ZonaTiros`, `Barrera`, `Dianas`,
   `Decoracion`. Asi el dueno lo encuentra y lo edita facil.
3. Helper unico para poner piezas (todo Plastic + Studs, con rotacion, forma y opciones):

```lua
local X0, Z0, GY = 1520, 172, 1.0         -- origen del mapa y altura del suelo
local function C(r,g,b) return Color3.fromRGB(r,g,b) end
local rnd = Random.new(2026)               -- semilla fija = resultado repetible
local function vary(c,a) local h,s,v=c:ToHSV(); return Color3.fromHSV(h,s,math.clamp(v*(1+rnd:NextNumber(-a,a)),0,1)) end
local function P(parent, name, size, lx, ly, lz, color, o)   -- coordenadas LOCALES; ly = altura sobre el suelo
  o = o or {}
  local p = Instance.new("Part"); p.Name = name; p.Anchored = true; p.Size = size
  p.CFrame = CFrame.new(X0+lx, GY+ly, Z0+lz) * CFrame.Angles(math.rad(o.rx or 0), math.rad(o.ry or 0), math.rad(o.rz or 0))
  p.Color = color; p.Material = Enum.Material.Plastic; p.MaterialVariant = "Studs"
  if o.neon then p.Material = Enum.Material.Neon; p.MaterialVariant = "" end
  if o.shape then p.Shape = o.shape end            -- Enum.PartType.Cylinder (eje en X)
  if o.collide == false then p.CanCollide = false end
  p.TopSurface = Enum.SurfaceType.Smooth; p.BottomSurface = Enum.SurfaceType.Smooth
  p.Parent = parent; return p
end
```

4. **Cilindros**: el eje es X. Para uno vertical rota `rz = 90` y pon la **altura en `Size.X`**.
   Para un cilindro que mira hacia Z (dianas, llantas de lado) rota `ry = 90`.
5. **Arboles**: un tronco + 5 pisos de hojas (bloques cada vez mas chicos, girados 45 grados alternando) +
   faldon mas ancho debajo de cada piso + punta. Varia `scale` (0.9-1.45), colores y tipo
   (pino 60%, roble 25%, muerto 15%). Las hojas `CanCollide = false`, el tronco si.
6. **Colocar muchos objetos**: lista `placed`, rechaza posiciones con `distancia < separacion` y las que caen
   en zonas reservadas (pista, plataforma, torre). Haz `task.wait()` cada 20 objetos.
7. **Parches de suelo**: 100-150 lajas planas (0.12 de grosor) de distintos verdes/tierra, cada una con una
   altura distinta (`0.07 + (i%9)*0.012`) para que no hagan z-fighting entre ellas.
8. **Muros invisibles** (Transparency 1, 90 de alto) en el borde si el suelo flota.
9. **Spawn**: `SpawnLocation` invisible, `Neutral = true`, `Duration = 0`, sin Decal.
10. **Luces**: `PointLight` en lamparas/torres (Range 36-70, Brightness 1.4-1.8). No toques el Lighting global.

### 4.3 Dianas, objetos "interactivos" y atributos

Para que otro desarrollador pueda programar encima, deja datos en **atributos**, no en codigo:

- Diana: `Model` con `Tablero` (PrimaryPart), anillos `Anillo` con atributo `Puntos` (1,2,3,5,10), `Centro`,
  atributos del modelo `Tipo` y `Distancia`.
- Tiendas/zonas: parte invisible `PuntoDeInteraccion` + atributo `Tipo`.
- Zonas seguras: parte invisible con atributo `ZonaSegura = true`.
- Pantallas (tabla de tops): `Part` con `SurfaceGui`; filas `Fila1..Fila10` con hijos `Rank`, `Nombre`, `Kills`.

### 4.4 Lo que hay que evitar

- No dejar un mapa a medias: si el script falla a la mitad, mira que se creo y completa o limpia.
- Un error comun: `Color3` no tiene `.Color`. Guarda colores como `Color3` y usalos directo.
- No agregues cosas que no pidieron (si el usuario dice "quitalo", se quita y no vuelve).
- No dejes ModuleScripts temporales (VoxelLib) en `ServerStorage` al terminar.

---

## 5. Estructura de scripts (sistemas del juego)

Regla: **un solo `Script` de servidor y un solo `LocalScript`**; todo lo demas son `ModuleScript` en carpetas.

```
ReplicatedStorage
  Shared
    Config        (numeros ajustables, rutas, nombres de tiendas)
    Remotes       (lista unica de RemoteEvents)
    Util/Formato  (2180 -> "2.18K$")
ServerScriptService
  Servidor        (Script: crea remotos, init de cada sistema, luego start)
    Sistemas/BaseService, CartelBase, Economia, ProximidadTiendas, TeleportRapido ...
StarterPlayer/StarterPlayerScripts
  Cliente         (LocalScript: igual, init y start)
    Controladores/AvisoBase, HudDinero, BotonesTP, TiendaUI ...
```

Reglas de oro:

- **Dos fases**: `init(sistemas)` (cablear dependencias, rapido) y luego `start()` (empezar a funcionar).
  Orden fijo en una lista `ORDEN`.
- **El servidor es la autoridad**. El cliente solo pide ("quiero ir a SHOP"); el servidor valida (lista blanca,
  cooldown, que exista) y ejecuta. Ningun remoto puede dar dinero.
- **El estado vive en atributos** (`Player:SetAttribute("Dinero", n)`, `Base:SetAttribute("OwnerUserId", id)`)
  para que otros sistemas y el HUD lo lean sin acoplarse, y para poder probar cambiando atributos.
- **Nada de `_G`**. Los modulos se piden con `require`. Eventos entre sistemas: `BindableEvent` expuesto como
  `Modulo.Changed`.
- **Probar con playtest**: `start_stop_play(true)`, luego `execute_luau` en `Server` y en `Client`
  (lee posiciones, atributos, textos de GUI), `get_console_output`, y `start_stop_play(false)`.
  Para el caso "bases llenas" pon `BasesHabilitadas = 0` un momento y restauralo.
- Spawn: si no hay `SpawnLocation`, el personaje cae en el origen. Pon uno por zona y usa `player.RespawnLocation`.
- Con `StreamingEnabled`, mide la cercania en el servidor y manda el resultado como atributo del jugador.

### GUI comprada (packs)

- **No borres**: mueve lo que no se usa a `ServerStorage.GuiPack_Respaldo`, de a pocos con `task.wait()`.
- No recorras `GetDescendants()` de un pack grande en una sola llamada (se cuelga). Entra por rutas conocidas.
- Los packs suelen tener cada texto **doble** (el label y un hijo `Label` de sombra): cambia los dos.
- Renombrar hijos de un `UIListLayout` puede reordenarlos si `SortOrder = Name`; revisa antes.
- Imagenes/mallas del pack que no te pertenecen dan "Requester does not have permission for Asset" y
  llenan la consola: archiva esas filas.

---

## 6. Lista de comprobacion antes de decir "listo"

- [ ] Esta en la carpeta correcta y con nombres claros.
- [ ] Todo Plastic + Studs, anclado, colores variados.
- [ ] Sin z-fighting ni piezas flotando (capturas desde 2-3 angulos).
- [ ] Texto legible (cara correcta del `SurfaceGui`).
- [ ] Probado en playtest si hay scripts (posiciones, atributos, textos, consola limpia).
- [ ] Nada borrado: lo viejo esta archivado; hay waypoints de deshacer.
- [ ] Temporales (VoxelLib, carpetas `_Tmp`) eliminados.
- [ ] Le dije al usuario: que hice, donde esta en el Explorer, que interprete yo y como cambiarlo,
      y que **los cambios estan en Studio pero no guardados**.

---

## 7. Ejemplos de lo que se logro con esta tecnica

- 5 trenes distintos en cubitos (azul bala, dorado, dragon rojo, barco secreto, hielo), de 6.000 a 11.000 partes.
- Pinos y palmeras navidenas, arbol de navidad gigante con regalos y estrella, tren de hielo gigante sobre un
  caballete de madera.
- Cueva de zona segura con globo y cartel "ZONE SAFE", tabla de tops con calaveras y sangre.
- Sistema de bases por jugador con cartel (nombre + foto del avatar), economia, teletransporte rapido y ventanas
  de tienda que se abren al acercarse.
- Mapa "bosque de entrenamiento": pista de tiro con 5 carriles, dianas, refugio, torre de vigilancia,
  vallas con alambre de puas y barrera de balas.
