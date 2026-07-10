# Uso de `cmd_vel_to_wheel_cmd_node`

Este nodo recibe comandos estándar ROS 2 en `cmd_vel` y los convierte al mensaje `wheel_cmd` usado por el simulador en Unity.

## Flujo general

```text
/PANTER/cmd_vel --> cmd_vel_to_wheel_cmd_node --> /PANTER/wheel_cmd --> Unity
```

* **Entrada**: `geometry_msgs/msg/Twist`
* **Salida**: `wheel_control_msgs/msg/WheelCommand`

El nodo se ejecuta dentro del namespace `PANTER`, por lo que trabajará sobre:

* `/PANTER/cmd_vel`
* `/PANTER/wheel_cmd`

---

## Requisitos previos

En cada terminal ROS 2:

```bash
source /opt/ros/humble/setup.bash
source ~/ros2_ws/install/setup.bash
```

Además:

* Unity debe estar en **Play**
* El **TCP Connector** debe estar conectado
* El endpoint `ros_tcp_endpoint` debe estar levantado

---

## Comando base

```bash
ros2 run panter_control cmd_vel_to_wheel_cmd_node --ros-args -r __ns:=/PANTER
```

Esto lanza el nodo en el namespace `PANTER`.

---

# Modos de funcionamiento

## 1. Ackermann

### Descripción

Interpreta `cmd_vel` como un vehículo tipo Ackermann:

* `linear.x` controla la velocidad longitudinal
* `angular.z` se transforma en un ángulo de dirección equivalente

### Lanzamiento

```bash
ros2 run panter_control cmd_vel_to_wheel_cmd_node --ros-args -r __ns:=/PANTER -p mode:=ackermann
```

### Ejemplos de prueba

#### Recta

```bash
ros2 topic pub /PANTER/cmd_vel geometry_msgs/msg/Twist "{linear: {x: 1.0}, angular: {z: 0.0}}" -r 10
```

#### Giro suave

```bash
ros2 topic pub /PANTER/cmd_vel geometry_msgs/msg/Twist "{linear: {x: 1.0}, angular: {z: 0.4}}" -r 10
```

### Uso recomendado

* Seguimiento de trayectorias suaves
* Comparación de radios de giro
* Maniobras tipo automóvil

---

## 2. Skid Steering

### Descripción

No usa dirección Ackermann. El giro se produce por diferencia de velocidad entre el lado izquierdo y derecho del vehículo.

### Lanzamiento

```bash
ros2 run panter_control cmd_vel_to_wheel_cmd_node --ros-args -r __ns:=/PANTER -p mode:=skid -p skid_feedforward_torque_nm:=15000.0 -p skid_turn_gain:=3.0
```

### Parámetros importantes

#### `skid_feedforward_torque_nm`

Par base aplicado a las ruedas para ayudar al giro skid.

* Si es bajo, el vehículo puede no girar o girar muy poco.
* Si es alto, se facilita el giro sobre el sitio y maniobras agresivas.

#### `skid_turn_gain`

Ganancia que amplifica la diferencia de velocidad entre lado izquierdo y derecho.

* Si es bajo, el giro será débil.
* Si es alto, el giro será más agresivo.

### Ejemplos de prueba

#### Giro sobre el sitio

```bash
ros2 topic pub /PANTER/cmd_vel geometry_msgs/msg/Twist "{linear: {x: 0.0}, angular: {z: 2.0}}" -r 10
```

#### Avanzar girando

```bash
ros2 topic pub /PANTER/cmd_vel geometry_msgs/msg/Twist "{linear: {x: 0.5}, angular: {z: 0.8}}" -r 10
```

### Uso recomendado

* Giro sobre el sitio
* Maniobras cerradas
* Comparación de maniobrabilidad

### Nota importante

En Unity, `maxMotorTorque` debe ser suficientemente alto.
Si Unity limita el par, da igual el valor enviado desde ROS 2: el vehículo no responderá como se espera.

---

## 3. Hybrid

### Descripción

Combina comportamiento Ackermann y skid steering.

### Lanzamiento

```bash
ros2 run panter_control cmd_vel_to_wheel_cmd_node --ros-args -r __ns:=/PANTER -p mode:=hybrid -p hybrid_blend:=0.5
```

### Parámetro importante

#### `hybrid_blend`

Factor de mezcla entre ambos modelos:

* `0.0` → comportamiento cercano a Ackermann
* `1.0` → comportamiento cercano a skid steering
* `0.5` → mezcla intermedia

### Ejemplo de prueba

```bash
ros2 topic pub /PANTER/cmd_vel geometry_msgs/msg/Twist "{linear: {x: 0.8}, angular: {z: 0.6}}" -r 10
```

### Uso recomendado

* Comparación de comportamiento intermedio
* Estudio del compromiso entre maniobrabilidad y suavidad

---

# Comprobación de funcionamiento

## Ver `cmd_vel`

```bash
ros2 topic echo /PANTER/cmd_vel
```

## Ver la salida a Unity

```bash
ros2 topic echo /PANTER/wheel_cmd
```

Conviene revisar especialmente:

* `velocity_rad_s`
* `torque_nm`
* `mode`
* `steering_center_rad`

---

# Ejemplos rápidos

## Ackermann

```bash
ros2 run panter_control cmd_vel_to_wheel_cmd_node --ros-args -r __ns:=/PANTER -p mode:=ackermann
```

## Skid

```bash
ros2 run panter_control cmd_vel_to_wheel_cmd_node --ros-args -r __ns:=/PANTER -p mode:=skid -p skid_feedforward_torque_nm:=15000.0 -p skid_turn_gain:=3.0
```

## Hybrid

```bash
ros2 run panter_control cmd_vel_to_wheel_cmd_node --ros-args -r __ns:=/PANTER -p mode:=hybrid -p hybrid_blend:=0.5
```

---

# Ejemplo completo de uso

## Terminal 1: allocator en modo skid

```bash
source /opt/ros/humble/setup.bash
source ~/ros2_ws/install/setup.bash
ros2 run panter_control cmd_vel_to_wheel_cmd_node --ros-args -r __ns:=/PANTER -p mode:=skid -p skid_feedforward_torque_nm:=15000.0 -p skid_turn_gain:=3.0
```

## Terminal 2: giro sobre el sitio

```bash
source /opt/ros/humble/setup.bash
source ~/ros2_ws/install/setup.bash
ros2 topic pub /PANTER/cmd_vel geometry_msgs/msg/Twist "{linear: {x: 0.0}, angular: {z: 2.0}}" -r 10
```

---

# Problemas frecuentes

## El vehículo no gira

Posibles causas:

* no se está publicando `cmd_vel`
* el par en Unity está limitado
* `skid_feedforward_torque_nm` es demasiado bajo
* `skid_turn_gain` es demasiado bajo
* alguna rueda está invertida o mal configurada en Unity

## El vehículo sigue recto en modo skid

Comprobar:

```bash
ros2 topic echo /PANTER/wheel_cmd
```

Si las ruedas izquierda y derecha no tienen velocidades suficientemente distintas, el giro no será visible.

---

# Resumen

* **Ackermann**: giro mediante dirección
* **Skid**: giro mediante diferencia entre lados
* **Hybrid**: mezcla entre ambos

El comando base es el mismo y solo cambia el parámetro `mode`.
