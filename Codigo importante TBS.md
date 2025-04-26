# GameBoard

### Clase y propiedades principales

Define una clase llamada **GameBoard** que hereda de **Node2D**, lo que significa que tendrá una posición en 2D y puede contener otros nodos.

```GDScript
  class_name GameBoard
  extends Node2D
```

Define las **4 direcciones posibles de movimiento** (izquierda, derecha, arriba, abajo).

```GDScript
const DIRECTIONS = [Vector2.LEFT, Vector2.RIGHT, Vector2.UP, Vector2.DOWN]
```

**grid**: Recurso exportado que define la cuadrícula del juego
**\_units**: Diccionario que mapea posiciones (Vector2) a unidades
**\_active_unit**: Referencia a la unidad seleccionada actualmente
**\_walkable_cells**: Array de celdas a las que puede moverse la unidad activa

```GDScript
@export var grid: Resource
var _units := {}
var _active_unit: Unit
var _walkable_cells := []
```

### Nodos hijos

Referencias a nodos hijos que se inicializan cuando el nodo está listo:

- **\_unit_overlay**: Muestra las celdas donde se puede mover la unidad
- **\_unit_path**: Muestra el camino que tomará la unidad
- **Label1**: Una etiqueta en el nodo padre (probablemente para debug)

  ```GDScript
  @onready var _unit_overlay: UnitOverlay = $UnitOverlay
  @onready var _unit_path: UnitPath = $UnitPath
  @onready var Label1: Label = $"../Label1"
  ```

### Funciones principales

**\_ready() y \_reinitialize()**

Inicializan el tablero y llenan el diccionario **\_units** con las unidades encontradas como hijos.

```GDScript
func _ready() -> void:
	_reinitialize()
```

**is_occupied(cell)**

Verifica si una celda está ocupada por una unidad.

```GDScript
func is_occupied(cell: Vector2) -> bool:
	return _units.has(cell)
```

**get_walkable_cells(unit)**

Usa el algoritmo de "flood fill" para encontrar todas las celdas a las que puede moverse una unidad, considerando su rango de movimiento y celdas ocupadas.

```GDScript
func get_walkable_cells(unit: Unit) -> Array:
	return _flood_fill(unit.cell, unit.move_range)

```

**\_flood_fill(cell, max_distance)**

Implementa el algoritmo flood fill para encontrar celdas accesibles:

- Comienza desde la celda de la unidad
- Explora en las 4 direcciones
- Considera el rango máximo de movimiento
- Omite celdas fuera de los límites o ocupadas

```GDScript
func _flood_fill(cell: Vector2, max_distance: int) -> Array:
	var array := []
	var stack := [cell]
	while not stack.size() == 0:
		var current = stack.pop_back()
		if not grid.is_within_bounds(current):
			continue
		if current in array:
			continue

		var difference: Vector2 = (current - cell).abs()
		var distance := int(difference.x + difference.y)
		if distance > max_distance:
			continue

		array.append(current)
		for direction in DIRECTIONS:
			var coordinates: Vector2 = current + direction
			if is_occupied(coordinates):
				continue
			if coordinates in array:
				continue
			# Minor optimization: If this neighbor is already queued
			#	to be checked, we don't need to queue it again
			if coordinates in stack:
				continue

			stack.append(coordinates)
	return array
```

#### Manejo de unidades activas

**\_select_unit(cell)**: Selecciona una unidad en la celda especificada

```GDSCRIPT
func _select_unit(cell: Vector2) -> void:
	if not _units.has(cell):
		return

	_active_unit = _units[cell]
	_active_unit.is_selected = true
	_walkable_cells = get_walkable_cells(_active_unit)
	_unit_overlay.draw(_walkable_cells)
	_unit_path.initialize(_walkable_cells)
```

**\_deselect_active_unit()**: Deselecciona la unidad activa

```GDSCRIPT
func _deselect_active_unit() -> void:
	_active_unit.is_selected = false
	_unit_overlay.clear()
	_unit_path.stop()
```

**\_clear_active_unit()**: Limpia la referencia a la unidad activa

```GDSCRIPT
func _clear_active_unit() -> void:
	_active_unit = null
	_walkable_cells.clear()
```

**\_move_active_unit(new_cell)**: Mueve la unidad activa a una nueva celda.

```gdscript
func _move_active_unit(new_cell: Vector2) -> void:
	if is_occupied(new_cell) or not new_cell in _walkable_cells:
		return
	# warning-ignore:return_value_discarded
	_units.erase(_active_unit.cell)
	_units[new_cell] = _active_unit
	_deselect_active_unit()
	_active_unit.walk_along(_unit_path.current_path)
	await _active_unit.walk_finished
	_clear_active_unit()

```

#### Manejo de entrada

**\_unhandled_input()**: Cancela la selección al presionar "**ui_cancel**"

```gdscript
func _unhandled_input(event: InputEvent) -> void:
	if _active_unit and event.is_action_pressed("ui_cancel"):
		_deselect_active_unit()
		_clear_active_unit()
```

**\_on_Cursor_accept_pressed(cell)**: Maneja clics para seleccionar/mover unidades

```gdscript
func _on_Cursor_accept_pressed(cell: Vector2) -> void:
	if not _active_unit:
		_select_unit(cell)
	elif _active_unit.is_selected:
		_move_active_unit(cell)
```

**\_on_Cursor_moved(new_cell)**: Actualiza la visualización del camino cuando el cursor se mueve

```gdscript
func _on_Cursor_moved(new_cell: Vector2) -> void:
	if _active_unit and _active_unit.is_selected:
		_unit_path.draw(_active_unit.cell, new_cell)
```

---

# Grid

### Clase y propiedades principales

Define una clase llamada **Grid** que hereda de **Resource**, lo que significa que puede ser guardada como un recurso reusable en Godot.

```gdscript
class_name Grid
extends Resource
```

**size**: Define las dimensiones del grid (10 columnas x 7 filas por defecto)

**cell_size**: Define el tamaño en píxeles de cada celda (80x80 píxeles por defecto)

```gdscript
@export var size := Vector2(10, 7)
@export var cell_size := Vector2(80, 80)
```

Variable interna que almacena la mitad del tamaño de una celda (útil para cálculos de centrado)

```gdscript
 var _half_cell_size = cell_size / 2
```

### Funciones principales

**calculate_map_position(grid_position)**

**Propósito**: Convierte coordenadas de grid (ej: [2,3]) a posición en píxeles en el mapa.
**Cómo funciona**:

- Multiplica la posición en grid por el tamaño de celda
- Suma la mitad del tamaño de celda para centrar la posición
- Ejemplo: Para celda [1,2] con cell_size 80x80 → (1*80 + 40, 2*80 + 40) = (120, 200)

```gdscript
func calculate_map_position(grid_position: Vector2) -> Vector2:
    return grid_position * cell_size + _half_cell_size
```

**calculate_grid_coordinates(map_position)**

**Propósito**: Convierte posición en píxeles a coordenadas de grid.
**Cómo funciona**:

- Divide la posición en píxeles por el tamaño de celda
- Usa floor() para redondear hacia abajo y obtener índices enteros
- Ejemplo: Para posición (150, 220) con cell_size 80x80 → (150/80, 220/80).floor() = (1, 2)

```gdscript
func calculate_grid_coordinates(map_position: Vector2) -> Vector2:
    return (map_position / cell_size).floor()
```

**is_within_bounds(cell_coordinates)**

**Propósito**: Verifica si unas coordenadas de grid están dentro de los límites.
**Cómo funciona**:

- Comprueba que las coordenadas **x** e **y** sean ≥ 0 y menores que el tamaño del grid
- Retorno: **true** si está dentro, **false** si está fuera

```gdscript
func is_within_bounds(cell_coordinates: Vector2) -> bool:
    var out := cell_coordinates.x >= 0 and cell_coordinates.x < size.x
    return out and cell_coordinates.y >= 0 and cell_coordinates.y < size.y
```

**grid_clamp(grid_position)**

Propósito: Ajusta una posición de grid para que esté dentro de los límites.
Cómo funciona:

- Usa la función **clamp()** para asegurar que los valores estén entre 0 y size-1
- Útil para prevenir accesos fuera de rango

```gdscript
func grid_clamp(grid_position: Vector2) -> Vector2:
    var out := grid_position
    out.x = clamp(out.x, 0, size.x - 1.0)
    out.y = clamp(out.y, 0, size.y - 1.0)
    return out
```

---

# Unit

### Clase y propiedades principales

**@tool**: Permite que el script se ejecute en el editor de Godot

Hereda de **Path2D**, lo que le permite moverse a lo largo de un camino definido

```gdscript
@tool
class_name Unit
extends Path2D
```

### Señales

Se emite cuando la unidad termina de moverse por un camino

```gdscript
signal walk_finished
```

### Propiedades exportadas

```gdscript
@export var grid: Resource          # Recurso Grid para cálculos de posición
@export var move_range := 3         # Rango de movimiento en celdas
@export var move_speed := 600.0     # Velocidad de movimiento en píxeles/segundo
@export var skin: Texture           # Apariencia visual de la unidad
@export var skin_offset := Vector2.ZERO  # Ajuste de posición del sprite
```

### Nodos hijos importantes

**\_sprite**: Representación visual de la unidad
**\_anim_player**: Controla animaciones (seleccionado/inactivo)
**\_path_follow**: Permite seguir un camino suavemente

```gdscript
@onready var _sprite: Sprite2D = $PathFollow2D/Sprite
@onready var _anim_player: AnimationPlayer = $AnimationPlayer
@onready var _path_follow: PathFollow2D = $PathFollow2D
```

### Variables importantes

```gdscript
var cell := Vector2.ZERO            # Posición actual en celdas del grid
var is_selected := false            # Si la unidad está seleccionada
var _is_walking := false            # Si la unidad está en movimiento
```

### Funciones principales

**\_ready()**

- Inicializa la unidad:
- Desactiva el procesamiento inicial
- Configura el PathFollow2D para que no rote
- Establece posición inicial basada en el grid
- Crea una curva vacía para el movimiento (solo en tiempo de juego)

```gdscript
func _ready() -> void:
	set_process(false)
	_path_follow.rotates = false

	cell = grid.calculate_grid_coordinates(position)
	position = grid.calculate_map_position(cell)

	# We create the curve resource here because creating it in the editor prevents us from
	# moving the unit.
	if not Engine.is_editor_hint():
		curve = Curve2D.new()
```

**\_process(delta)**

- Maneja el movimiento cuando \_is_walking es true:
- Actualiza la posición a lo largo del camino
- Muestra "UNIT MOVE" en Label1 mientras se mueve
- Cuando termina el movimiento:
- Reinicia la posición
- Limpia la curva
- Emite señal walk_finished
- Muestra "UNIT STAY" en Label1

```gdscript
func _process(delta: float) -> void:
	_path_follow.progress += move_speed * delta
	Label1.text= " UNIT MOVE "


	if _path_follow.progress_ratio >= 1.0:
		_is_walking = false
		# Setting this value to 0.0 causes a Zero Length Interval error
		_path_follow.progress = 0.00001
		position = grid.calculate_map_position(cell)
		curve.clear_points()
		$"../../Player_2".play()
		emit_signal("walk_finished")
		Label1.text= " UNIT STAY "
```

**walk_along(path)**

- Inicia el movimiento a lo largo de un camino:
- Convierte coordenadas de grid a posiciones globales
- Actualiza la posición final (cell)
- Activa el movimiento (\_is_walking = true)

```gdscript
func walk_along(path: PackedVector2Array) -> void:
	if path.is_empty():
		return

	curve.add_point(Vector2.ZERO)
	for point in path:
		curve.add_point(grid.calculate_map_position(point) - position)
		cell = path[-1]
	_is_walking = true
```

# PathFinder

### Clase y propiedades principales

Define una clase llamada PathFinder que hereda de Resource, lo que permite guardarla como recurso reusable

```gdscript
class_name PathFinder
extends Resource
```

Define las 4 direcciones posibles de movimiento (sin diagonales)

```gdscript
const DIRECTIONS = [Vector2.LEFT, Vector2.RIGHT, Vector2.UP, Vector2.DOWN]
```

**\_grid**: Referencia al recurso Grid que define el tamaño y celdas

**\_astar**: Instancia del algoritmo **AStarGrid2D** integrado en Godot

```gdscript
var _grid: Resource
var _astar := AStarGrid2D.new()
```

### Inicialización (\_init)

Parámetros:

- **grid**: El recurso Grid con las dimensiones del tablero
- **walkable_cells**: Array de celdas por las que se puede caminar

```gdscript
func _init(grid: Grid, walkable_cells: Array) -> void:
```

Configura el tamaño del grid A\* para que coincida con el grid del juego

```gdscript
_astar.size = _grid.size
_astar.cell_size = _grid.cell_size
```

Desactiva el movimiento diagonal (**solo permite movimientos ortogonales**)

```gdscript
_astar.diagonal_mode = AStarGrid2D.DIAGONAL_MODE_NEVER
```

Usa la heurística Manhattan (**suma de diferencias en X e Y**) para calcular distancias

```gdscript
_astar.default_compute_heuristic = AStarGrid2D.HEURISTIC_MANHATTAN
_astar.default_estimate_heuristic = AStarGrid2D.HEURISTIC_MANHATTAN
```

Actualiza la estructura interna del grid A\*

```gdscript
_astar.update()
```

Marca como sólidos (no transitables) todas las celdas que no estén en walkable_cells

```gdscript
for y in _grid.size.y:
    for x in _grid.size.x:
        if not walkable_cells.has(Vector2(x,y)):
            _astar.set_point_solid(Vector2(x,y))
```

### Función principal: calculate_point_path

- **Propósito**: Calcula el camino más corto entre dos puntos
- **Parámetros**:
  - **start**: Punto de inicio (Vector2)
  - **end**: Punto destino (Vector2)
- **Retorno**: Array de Vector2 con las coordenadas del camino
- **Funcionamiento**:
  - Usa el algoritmo A\* integrado para encontrar el camino óptimo
  - Solo considera celdas marcadas como transitables en la inicialización
  - Devuelve un camino que evita obstáculos

```gdscript
func calculate_point_path(start: Vector2, end: Vector2) -> PackedVector2Array:
    return _astar.get_id_path(start, end)
```

### Características clave

- **Optimizado para grids 2D**: Usa AStarGrid2D que está optimizado para grids rectangulares
- **Movimiento ortogonal**: Solo permite movimientos en 4 direcciones (no diagonales)
- **Heurística Manhattan**: Adecuada para movimientos en grid sin diagonales
- **Eficiente**: Pre-calcula los obstáculos en la inicialización para búsquedas rápidas

---

# Cursor

### Clase y propiedades principales

**@tool**: Permite que el script se ejecute en el editor de Godot
Hereda de **Node2D** para tener posición en pantalla

```gdscript
@tool
class_name Cursor
extends Node2D
```

### Señales

```gdscript
signal accept_pressed(cell)  # Se emite al hacer clic o presionar "aceptar"
signal moved(new_cell)       # Se emite al mover el cursor a nueva celda
```

### Propiedades exportadas

```gdscript
@export var grid: Resource      # Recurso Grid para cálculos de posición
@export var ui_cooldown := 0.1  # Tiempo de espera entre movimientos (segundos)
```

### Variables importantes

Tiene un setter personalizado que:

- Limita el movimiento a los bordes del grid
- Actualiza la posición visual
- Emite la señal **moved**
- Inicia el temporizador de cooldown

```gdscript
var cell := Vector2.ZERO        # Celda actual donde está el cursor
set(value):
		var new_cell: Vector2 = grid.grid_clamp(value)
		if new_cell.is_equal_approx(cell):
			return

		cell = new_cell

		position = grid.calculate_map_position(cell)
		emit_signal("moved", cell)
		_timer.start()
```

### Nodos hijos

```gdscript
@onready var _timer: Timer = $Timer  # Temporizador para el cooldown de movimiento
@onready var Label1: Label = $"../../Label1"  # Etiqueta para mostrar información
```

### Funciones principales

**\_ready()**

- Inicializa el temporizador con el tiempo de cooldown
- Posiciona el cursor en su celda inicial

```gdscript
func _ready() -> void:
	_timer.wait_time = ui_cooldown
	position = grid.calculate_map_position(cell)
```

**\_unhandled_input(event)**

Maneja tres tipos de entrada:

- Movimiento del mouse:
  - Actualiza la celda según la posición del mouse

```gdscript
if event is InputEventMouseMotion:
    cell = grid.calculate_grid_coordinates(event.position) + Vector2(-1,-1)
```

- Clic/Aceptar:
  - Emite señal cuando el jugador hace clic o presiona Enter

```gdscript
elif event.is_action_pressed("click") or event.is_action_pressed("ui_accept"):
    emit_signal("accept_pressed", cell)
```

- Movimiento con teclado:
  - Maneja las 4 direcciones básicas
  - Respeta el cooldown para movimientos repetidos

```gdscript
# Moves the cursor by one grid cell.
	if event.is_action("ui_right"):
		cell += Vector2.RIGHT
	elif event.is_action("ui_up"):
		cell += Vector2.UP
	elif event.is_action("ui_left"):
		cell += Vector2.LEFT
	elif event.is_action("ui_down"):
		cell += Vector2.DOWN
```

**\_draw()**

- Actualiza la animación visual del cursor (comentado hay un ejemplo de dibujo alternativo)

```gdscript
func _draw() -> void:
	#draw_rect(Rect2(-grid.cell_size / 2, grid.cell_size), Color.ALICE_BLUE, false, 2.0)
	$AnimatedSprite2D.play()
```

### Características importantes

- **Cooldown inteligente**: Evita movimientos demasiado rápidos al mantener presionadas teclas
- **Doble control**: Soporta tanto mouse como teclado
- **Posición precisa**: Usa el grid para convertir entre coordenadas de pantalla y celdas
- **Feedback visual**: Tiene animación (AnimatedSprite2D) para mejor experiencia de juego

---

# UnitOverlay

### Clase y herencia

Hereda de **TileMap**, lo que le permite dibujar celdas de un grid fácilmente

Su propósito es mostrar visualmente las celdas accesibles para una unidad seleccionada

```gdscript
class_name UnitOverlay
extends TileMap
```

### Componentes clave

Referencia a un Label que muestra información de estado **("UNIT SELECTED")**

```gdscript
@onready var Label1: Label = $"../../Label1"
```

### Función principal

- **clear()**: Limpia cualquier celda previamente dibujada
- Itera sobre el array de celdas recibido (cells):

  - **set_cell(0, cell, 0, Vector2i(0,0))**: Dibuja cada celda accesible

    - **Parámetros**:

      - **0**: Capa del TileMap

      - **cell**: Coordenada de la celda a dibujar

      - **0**: ID del tileset (asumiendo que hay un tileset configurado)

      - **Vector2i(0,0)**: Coordenada del tile dentro del tileset

- Actualiza el Label para mostrar **"UNIT SELECTED"**

- Reproduce un efecto de sonido **(Player_1)**

```gdscript
func draw(cells: Array) -> void:
    clear()
    for cell in cells:
        set_cell(0, cell, 0, Vector2i(0,0))

    Label1.text = "UNIT SELECTED"
    $"../../Player_1".play()
```

### Características importantes

- **Eficiente**: Usa el sistema de TileMap integrado para renderizado óptimo
- **Feedback claro**: Muestra claramente dónde puede moverse la unidad
- **Retroalimentación multimodal**: Combina elementos visuales y auditivos
- **Simple pero efectivo**: Cumple su función con código mínimo

---

# UnitPath

### Clase y propiedades principales

Hereda de TileMap, permitiendo dibujar caminos como celdas conectadas

```gdscript
class_name UnitPath
extends TileMap
```

```gdscript
@export var grid: Resource  # Recurso Grid para cálculos de posición
@onready var Label1: Label = $"../../Label1"  # Referencia a un Label para mostrar estado
```

### Variables importantes

```gdscript
var _pathfinder: PathFinder  # Instancia del buscador de caminos
var current_path := PackedVector2Array()  # Almacena el camino actual calculado
```

### Funciones principales

**initialize(walkable_cells)**

- **Propósito**: Prepara el buscador de caminos para una unidad específica

- **Parámetro**: walkable_cells - array de celdas donde la unidad puede moverse

- **Acción**: Crea una nueva instancia de PathFinder con las celdas transitables

```gdscript
func initialize(walkable_cells: Array) -> void:
    _pathfinder = PathFinder.new(grid, walkable_cells)
```

**draw(cell_start, cell_end)**

- **Propósito**: Calcula y dibuja el camino entre dos puntos

- **Pasos**:
  - Limpia el dibujo anterior (clear())
  - Calcula el nuevo camino usando A\* (calculate_point_path)
  - Dibuja el camino con celdas conectadas visualmente (set_cells_terrain_connect)
  - Actualiza el Label para mostrar "UNIT STAY"

```gdscript
func draw(cell_start: Vector2, cell_end: Vector2) -> void:
    clear()
    current_path = _pathfinder.calculate_point_path(cell_start, cell_end)
    set_cells_terrain_connect(0, current_path, 0, 0)
    Label1.text = " UNIT STAY "
```

**stop()**

- **Propósito**: Limpia el camino visualizado y libera recursos
- **Acciones**:
  - Elimina la referencia al PathFinder
  - Limpia todos los tiles dibujados

```gdscript
func stop() -> void:
    _pathfinder = null
    clear()
```

### Características clave

- **Visualización conectada**: Usa set_cells_terrain_connect para mostrar caminos continuos
- **Eficiente**: Reutiliza la instancia de PathFinder mientras la unidad está seleccionada
- **Integración con Grid**: Trabaja con coordenadas del grid para posicionamiento preciso
- **Feedback visual claro**: Muestra exactamente por dónde se moverá la unidad
