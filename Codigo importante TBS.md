# GameBoard

### Clase y propiedades principales

- Define una clase llamada **GameBoard** que hereda de **Node2D**, lo que significa que tendrá una posición en 2D y puede contener otros nodos.

  ```GDScript
    class_name GameBoard
    extends Node2D
  ```

- Define las **4 direcciones posibles de movimiento** (izquierda, derecha, arriba, abajo).

  ```GDScript
  const DIRECTIONS = [Vector2.LEFT, Vector2.RIGHT, Vector2.UP, Vector2.DOWN]
  ```

- **grid**: Recurso exportado que define la cuadrícula del juego
- **\_units**: Diccionario que mapea posiciones (Vector2) a unidades
- **\_active_unit**: Referencia a la unidad seleccionada actualmente
- **\_walkable_cells**: Array de celdas a las que puede moverse la unidad activa

  ```GDScript
  @export var grid: Resource
  var _units := {}
  var _active_unit: Unit
  var _walkable_cells := []
  ```

### Nodos hijos

- Referencias a nodos hijos que se inicializan cuando el nodo está listo:

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

- Inicializan el tablero y llenan el diccionario **\_units** con las unidades encontradas como hijos.

```GDScript
func _ready() -> void:
	_reinitialize()
```

**is_occupied(cell)**

- Verifica si una celda está ocupada por una unidad.

```GDScript
func is_occupied(cell: Vector2) -> bool:
	return _units.has(cell)
```

**get_walkable_cells(unit)**

- Usa el algoritmo de "flood fill" para encontrar todas las celdas a las que puede moverse una unidad, considerando su rango de movimiento y celdas ocupadas.

```GDScript
func get_walkable_cells(unit: Unit) -> Array:
	return _flood_fill(unit.cell, unit.move_range)

```

**\_flood_fill(cell, max_distance)**

- Implementa el algoritmo flood fill para encontrar celdas accesibles:
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

- Define una clase llamada **Grid** que hereda de **Resource**, lo que significa que puede ser guardada como un recurso reusable en Godot.

  ```gdscript
  class_name Grid
  extends Resource
  ```

- **size**: Define las dimensiones del grid (10 columnas x 7 filas por defecto)

- **cell_size**: Define el tamaño en píxeles de cada celda (80x80 píxeles por defecto)

  ```gdscript
  @export var size := Vector2(10, 7)
  @export var cell_size := Vector2(80, 80)
  ```

- Variable interna que almacena la mitad del tamaño de una celda (útil para cálculos de centrado)

  ```gdscript
   var _half_cell_size = cell_size / 2
  ```

### Funciones principales

**calculate_map_position(grid_position)**

- **Propósito**: Convierte coordenadas de grid (ej: [2,3]) a posición en píxeles en el mapa.
- **Cómo funciona**:

  - Multiplica la posición en grid por el tamaño de celda
  - Suma la mitad del tamaño de celda para centrar la posición
  - Ejemplo: Para celda [1,2] con cell_size 80x80 → (1*80 + 40, 2*80 + 40) = (120, 200)

```gdscript
func calculate_map_position(grid_position: Vector2) -> Vector2:
    return grid_position * cell_size + _half_cell_size
```

**calculate_grid_coordinates(map_position)**

- **Propósito**: Convierte posición en píxeles a coordenadas de grid.
- **Cómo funciona**:

  - Divide la posición en píxeles por el tamaño de celda
  - Usa floor() para redondear hacia abajo y obtener índices enteros
  - Ejemplo: Para posición (150, 220) con cell_size 80x80 → (150/80, 220/80).floor() = (1, 2)

```gdscript
func calculate_grid_coordinates(map_position: Vector2) -> Vector2:
    return (map_position / cell_size).floor()
```

**is_within_bounds(cell_coordinates)**

- **Propósito**: Verifica si unas coordenadas de grid están dentro de los límites.
- **Cómo funciona**:
  - Comprueba que las coordenadas **x** e **y** sean ≥ 0 y menores que el tamaño del grid
  - Retorno: **true** si está dentro, **false** si está fuera

```gdscript
func is_within_bounds(cell_coordinates: Vector2) -> bool:
    var out := cell_coordinates.x >= 0 and cell_coordinates.x < size.x
    return out and cell_coordinates.y >= 0 and cell_coordinates.y < size.y
```

**grid_clamp(grid_position)**

- Propósito: Ajusta una posición de grid para que esté dentro de los límites.
- Cómo funciona:
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

- **@tool**: Permite que el script se ejecute en el editor de Godot
- Hereda de **Path2D**, lo que le permite moverse a lo largo de un camino definido

```gdscript
@tool
class_name Unit
extends Path2D
```

### Señales

- Se emite cuando la unidad termina de moverse por un camino

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

- **\_sprite**: Representación visual de la unidad
- **\_anim_player**: Controla animaciones (seleccionado/inactivo)
- **\_path_follow**: Permite seguir un camino suavemente

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

- Define una clase llamada PathFinder que hereda de Resource, lo que permite guardarla como recurso reusable

```gdscript
class_name PathFinder
extends Resource
```

- Define las 4 direcciones posibles de movimiento (sin diagonales)

```gdscript
const DIRECTIONS = [Vector2.LEFT, Vector2.RIGHT, Vector2.UP, Vector2.DOWN]
```

- **\_grid**: Referencia al recurso Grid que define el tamaño y celdas

- **\_astar**: Instancia del algoritmo **AStarGrid2D** integrado en Godot

```gdscript

```

```gdscript

```

```gdscript

```

```gdscript

```
