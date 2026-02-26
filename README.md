# Proyecto-integrador-escenario-procedural
Explicación del proyecto integrador de la unidad 1. 


A partir del código proporcionado por el profesor, se hicieron unas mejoras al código para su resultado final.

A continuación se explicará el paso a paso de cómo funciona el código de Blender para el escenario procedural agregando la animación de la cámara a través del camino. 


## 1. Preparación del Escenario y Materiales
Antes de construir, el código "limpia la mesa" y define la paleta de colores. Se utiliza una función para crear materiales usando el motor de nodos de Blender (Principled BSDF).
```bash
def crear_material(nombre, color_rgb):
    mat = bpy.data.materials.new(name=nombre)
    mat.use_nodes = True
    bsdf = mat.node_tree.nodes.get("Principled BSDF")
    if bsdf:
        bsdf.inputs["Base Color"].default_value = (*color_rgb, 1.0)
    return mat

# Se limpian objetos previos y se definen materiales (Gris, Naranja, Oscuro)
bpy.ops.object.select_all(action='SELECT')
bpy.ops.object.delete()
mat_a = crear_material("ParedOscura", (0.1, 0.1, 0.1))
```
* Para qué sirve: El nodo Principled BSDF es un estándar de la industria (PBR - Physically Based Rendering). Al activarlo mediante código (mat.use_nodes = True), aseguramos que el material reaccione correctamente a la luz, sombras y reflejos, haciendo que el pasillo se vea realista y no como un dibujo plano.


## 2. La Lógica de la Curvatura (Matemáticas)
Aquí es donde entra la ingeniería. El pasillo no es recto; sigue una función matemática. Se define una función offset_x que usa el Coseno para crear transiciones suaves (curvas) en el eje X a medida que avanzamos en el eje Y.
```bash
def offset_x(i):
    x = 0.0
    if 15 <= i <= 30:
        t = (i - 15) / 15.0
        x += 6.0 * (0.5 - 0.5 * math.cos(t * math.pi)) # Interpolación cosenoidal
    # ... (lógica para las siguientes curvas)
    return x
```
* Tangente: También calculamos el ángulo de inclinación (math.atan2) para que las paredes roten según la curva y no queden "mirando" siempre al frente.
* Para qué sirve: La función coseno (cos) genera una curva suave de tipo sigmoide. Al usarla, creamos una transición orgánica donde el pasillo empieza a girar lentamente, llega a su punto máximo de curva y se endereza con suavidad. Sin esto, los giros serían bruscos y la cámara daría "saltones" poco profesionales.


## 3. Construcción de las Paredes (Instanciación)
Usamos un ciclo for para colocar cubos a lo largo del camino calculado. Se crean dos paredes (izquierda y derecha) desplazadas del centro por una variable ancho.
```bash
for i in range(total_bloques):
    cx = offset_x(i)
    cy = i * paso
    rot = angulo_tangente(i)

    # Pared izquierda
    bpy.ops.mesh.primitive_cube_add(location=(cx - ancho, cy, altura_pared / 2))
    p = bpy.context.active_object
    p.scale = (grosor_pared, fill_y / 2 + 0.1, esc_z)
    p.rotation_euler.z = rot # Alineación con la curva
```
* El cálculo de la Tangente (math.atan2)

¿Por qué? Porque si solo movemos los cubos a la posición (x, y), todos quedarían mirando hacia el mismo frente (como soldados en fila), chocando entre sí en las curvas.
  * Para qué sirve: La función atan2 calcula el ángulo exacto hacia donde apunta el "camino" en ese punto específico. Esto permite que cada bloque de pared rote para quedar perpendicular al suelo, manteniendo el ancho del pasillo constante durante todo el trayecto.


## 4. Generación Procedural del Suelo
A diferencia de las paredes, el suelo se crea como una malla única personalizada (mesh). Se calculan los vértices (puntos en el espacio) y las caras (cuadriláteros que unen esos puntos) para que el suelo sea una cinta continua.
```bash
# Se definen vértices a los lados del camino central
verts.append((cx + px * (-ancho), cy + py * (-ancho), 0))
verts.append((cx + px * ( ancho), cy + py * ( ancho), 0))

# Se crean las caras uniendo los vértices en grupos de 4
mesh.from_pydata(verts, [], faces)
```
* Generación de Malla desde Cero (from_pydata)

Podríamos haber usado muchos planos pequeños para el suelo, pero eso es ineficiente para el motor de render.

* Para qué sirve: Al definir verts (vértices) y faces (caras) manualmente, creamos un único objeto geométrico. Esto se conoce como modelado procedural. Es mucho más ligero para la memoria RAM y permite que las texturas se apliquen de forma continua sin costuras visibles entre bloques.


## 5. Animación de la Cámara (Cinemática)
Finalmente, el script crea una cámara y le asigna Keyframes. La cámara recorre el centro del pasillo, ajustando su posición y rotación en cada cuadro para simular un recorrido en primera persona.
```bash
for i in range(bloques_kf):
    # Cálculo del frame actual y posición
    cam_obj.location = (cx, cy, cam_z)
    cam_obj.rotation_euler = (math.radians(90), 0, rot)

    # Insertar "fotograma clave"
    cam_obj.keyframe_insert(data_path="location", frame=frame)
    cam_obj.keyframe_insert(data_path="rotation_euler", frame=frame)
```

Interpolación de Animación (BEZIER)

Por defecto, las animaciones pueden ser lineales (velocidad constante y robótica).
  * Para qué sirve: Al asignar kp.interpolation = 'BEZIER', le decimos a Blender que suavice la aceleración y desaceleración de la cámara. Esto imita el movimiento humano o de un dron, haciendo que el recorrido por el pasillo sea fluido visualmente.
```bash
# Suavizar interpolación de todos los keyframes
    action = cam_obj.animation_data.action
    for fcurve in action.fcurves:
      for kp in fcurve.keyframe_points:
            kp.interpolation = 'BEZIER'
```

Automatización del Render

Para asegurar que cualquier persona que ejecute el código vea exactamente lo mismo que nosotros.

  * Para qué sirve: Configurar los FPS (cuadros por segundo) y la resolución por código garantiza que la animación dure el tiempo correcto y tenga la calidad deseada sin que el usuario tenga que mover ajustes manuales en la interfaz de Blender.


### Resumen 
Este script demuestra el poder de la graficación procedural. En lugar de modelar a mano, usamos funciones trigonométricas para dictar la forma, bucles para la repetición de geometría y manipulación de datos de bajo nivel para generar mallas y curvas de animación suavizadas (Bézier).


# RESULTADO
<img width="757" height="478" alt="image" src="https://github.com/user-attachments/assets/daf4fb25-ff0f-4aac-bffe-448a45ced6b7" />
<img width="754" height="486" alt="image" src="https://github.com/user-attachments/assets/01fe2c79-3f18-4b2c-ac68-90ca9abb59d6" />

