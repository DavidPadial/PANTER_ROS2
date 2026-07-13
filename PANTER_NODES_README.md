# Arquitectura actual de PANTER en ROS 2 y Unity

## 1. Resumen rápido

Ahora mismo la arquitectura práctica del proyecto queda así:

- **Unity** simula el vehículo y recibe comandos por ROS-TCP.
- **ROS 2** genera referencias de movimiento y las convierte a comandos por rueda.
- El flujo principal es:

```text
/PANTER/cmd_vel -> cmd_vel_to_wheel_cmd_node -> /PANTER/wheel_cmd -> Unity
```

Y Unity publica de vuelta al menos:

```text
/PANTER/joint_states
```

Además, se ha preparado la base para `wheel_state`, aunque su uso en Unity depende de tener generado correctamente `WheelStateMsg`.

---

## 2. Paquetes que tienes ahora mismo

### 2.1 `panter_control`
Paquete principal de control en ROS 2.

**Tipo:** `ament_python`

**Contiene los nodos principales:**
- `cmd_vel_to_wheel_cmd_node`
- `trajectory_test_node`
- `spin_test_node`

**Responsabilidad:**
- generar comandos de prueba en `cmd_vel`
- convertir `cmd_vel` a `wheel_cmd`
- seleccionar el modo cinemático:
  - `ackermann`
  - `skid`
  - `hybrid`

---

### 2.2 `wheel_control_msgs`
Paquete de mensajes ROS 2 para el control de ruedas.

**Mensaje importante:**
- `WheelCommand`

**Responsabilidad:**
- definir el mensaje que Unity recibe como `/PANTER/wheel_cmd`

Campos usados de forma práctica:
- nombres de rueda
- velocidad angular objetivo por rueda
- par por rueda
- modo de control
- ángulo de dirección central

---

### 2.3 `wheel_state`
Paquete de mensajes ROS 2 para estado ampliado de ruedas.

**Mensaje importante:**
- `WheelState`

**Responsabilidad:**
- definir un mensaje más rico para publicar:
  - carga por rueda
  - velocidad angular
  - par motor
  - par de freno
  - slip longitudinal/lateral
  - grounded

**Estado actual:**
- la interfaz ROS 2 ya puede generarse correctamente
- para usarlo en Unity hace falta tener generado también `WheelStateMsg` en el ROS-TCP-Connector

---

### 2.4 `ros_tcp_endpoint`
Paquete externo usado como puente ROS 2 <-> Unity.

**Nodo usado:**
- `default_server_endpoint`

**Responsabilidad:**
- abrir el servidor TCP que usa Unity para conectarse a ROS 2

---

## 3. Nodos que tienes ahora mismo

### 3.1 `cmd_vel_to_wheel_cmd_node`
Nodo traductor principal.

**Entrada:**
- `cmd_vel` (`geometry_msgs/Twist`)

**Salida:**
- `wheel_cmd` (`wheel_control_msgs/WheelCommand`)

**Qué hace:**
- recibe una referencia estándar ROS 2 (`linear.x`, `angular.z`)
- la transforma en consignas por rueda
- soporta tres modos:
  - `ackermann`
  - `skid`
  - `hybrid`

**Topics normales bajo namespace `PANTER`:**
- suscribe: `/PANTER/cmd_vel`
- publica: `/PANTER/wheel_cmd`

---

### 3.2 `trajectory_test_node`
Nodo simple para pruebas temporizadas.

**Salida:**
- `cmd_vel`

**Qué hace:**
- publica una secuencia sencilla de maniobras
- se usa para probar:
  - recta
  - giro izquierda
  - giro derecha
  - cambios de fase simples

**Uso típico:**
- pruebas rápidas de funcionamiento general

---

### 3.3 `spin_test_node`
Nodo específico para ensayo de giro sobre el sitio.

**Salida:**
- `cmd_vel`

**Qué hace:**
- publica referencias tipo:
  - `linear.x = 0.0`
  - `angular.z = 2.0`
- sirve para estudiar cuánto par hace falta para que el vehículo gire sobre sí mismo, especialmente en `skid`

**Uso típico:**
- calibración de `skid_feedforward_torque_nm`
- calibración de `skid_turn_gain`

---

## 4. Scripts importantes en Unity

### 4.1 `ROS_wheelcontroller`
Script principal del vehículo en Unity.

**Responsabilidad:**
- subscribirse a `/PANTER/wheel_cmd`
- aplicar el comando a las ruedas NWH
- publicar `/PANTER/joint_states`
- opcionalmente, en el futuro, publicar `/PANTER/wheel_state`

**Topics usados normalmente:**
- entrada: `/PANTER/wheel_cmd`
- salida: `/PANTER/joint_states`
- salida futura/opcional: `/PANTER/wheel_state`

---

### 4.2 `WheelTopViewDebugUI`
Script de interfaz/debug en Unity.

**Responsabilidad:**
- mostrar una ventana con vista superior del vehículo
- visualizar:
  - ruedas
  - velocidad angular
  - peso/carga
  - par
  - ángulo
  - dirección del vehículo

**Importante:**
- no es un nodo ROS 2
- es solo una ayuda visual dentro de Unity

---

## 5. Topics principales del sistema

### Entrada de alto nivel
- `/PANTER/cmd_vel`

### Comando detallado al simulador
- `/PANTER/wheel_cmd`

### Estado desde Unity
- `/PANTER/joint_states`

### Estado ampliado de ruedas (cuando esté operativo en Unity)
- `/PANTER/wheel_state`

---

## 6. Contenedor Docker

Asumiendo que el contenedor de trabajo sigue llamándose:

```text
ros2_humble_unity_dev
```

si cambiaste el nombre, sustituye ese nombre por el tuyo real.

### 6.1 Arrancar el contenedor

```bash
docker start -ai ros2_humble_unity_dev
```

Si ya está arrancado y quieres entrar desde otra terminal:

```bash
docker exec -it ros2_humble_unity_dev bash
```

---

### 6.2 Preparar el entorno ROS 2 dentro del contenedor

Una vez dentro:

```bash
cd ~/ros2_ws
source /opt/ros/humble/setup.bash
source ~/ros2_ws/install/setup.bash
```

Si recompilas algún paquete:

```bash
cd ~/ros2_ws
source /opt/ros/humble/setup.bash
colcon build --packages-select panter_control --symlink-install
source ~/ros2_ws/install/setup.bash
```

---

## 7. Orden correcto de lanzamiento

## Paso 1. Arrancar el contenedor

```bash
docker start -ai ros2_humble_unity_dev
```

---

## Paso 2. Preparar ROS 2 en la terminal del contenedor

```bash
cd ~/ros2_ws
source /opt/ros/humble/setup.bash
source ~/ros2_ws/install/setup.bash
```

---

## Paso 3. Lanzar el TCP endpoint

```bash
ros2 run ros_tcp_endpoint default_server_endpoint --ros-args -p ROS_IP:=0.0.0.0
```

Este paso debe quedarse corriendo.

---

## Paso 4. Abrir otra terminal en el contenedor

```bash
docker exec -it ros2_humble_unity_dev bash
```

Y volver a preparar el entorno:

```bash
cd ~/ros2_ws
source /opt/ros/humble/setup.bash
source ~/ros2_ws/install/setup.bash
```

---

## Paso 5. Lanzar `cmd_vel_to_wheel_cmd_node`

### Ackermann

```bash
ros2 run panter_control cmd_vel_to_wheel_cmd_node --ros-args -r __ns:=/PANTER -p mode:=ackermann
```

### Skid

```bash
ros2 run panter_control cmd_vel_to_wheel_cmd_node --ros-args -r __ns:=/PANTER -p mode:=skid -p skid_feedforward_torque_nm:=15000.0 -p skid_turn_gain:=3.0
```

### Hybrid

```bash
ros2 run panter_control cmd_vel_to_wheel_cmd_node --ros-args -r __ns:=/PANTER -p mode:=hybrid -p hybrid_blend:=0.5
```

Este nodo también debe quedarse corriendo.

---

## Paso 6. Abrir otra terminal en el contenedor

```bash
docker exec -it ros2_humble_unity_dev bash
```

Preparar entorno otra vez:

```bash
cd ~/ros2_ws
source /opt/ros/humble/setup.bash
source ~/ros2_ws/install/setup.bash
```

---

## Paso 7. Lanzar un nodo de prueba de trayectoria

### Opción A. Prueba general temporizada

```bash
ros2 run panter_control trajectory_test_node --ros-args -r __ns:=/PANTER
```

### Opción B. Giro sobre el sitio

```bash
ros2 run panter_control spin_test_node --ros-args -r __ns:=/PANTER -p linear_x:=0.0 -p angular_z:=2.0
```

---

## Paso 8. Arrancar Unity

En Unity:
- abrir la escena
- comprobar que el vehículo tiene `ROS_wheelcontroller`
- comprobar que el `ROSConnection` apunta al endpoint correcto
- dar a **Play**

---

## 8. Resumen operativo mínimo

### Para una prueba normal en Ackermann

**Terminal 1**
```bash
cd ~/ros2_ws
source /opt/ros/humble/setup.bash
source ~/ros2_ws/install/setup.bash
ros2 run ros_tcp_endpoint default_server_endpoint --ros-args -p ROS_IP:=0.0.0.0
```

**Terminal 2**
```bash
cd ~/ros2_ws
source /opt/ros/humble/setup.bash
source ~/ros2_ws/install/setup.bash
ros2 run panter_control cmd_vel_to_wheel_cmd_node --ros-args -r __ns:=/PANTER -p mode:=ackermann
```

**Terminal 3**
```bash
cd ~/ros2_ws
source /opt/ros/humble/setup.bash
source ~/ros2_ws/install/setup.bash
ros2 run panter_control trajectory_test_node --ros-args -r __ns:=/PANTER
```

**Unity**
- abrir escena
- Play

---

### Para una prueba de giro sobre el sitio en Skid

**Terminal 1**
```bash
cd ~/ros2_ws
source /opt/ros/humble/setup.bash
source ~/ros2_ws/install/setup.bash
ros2 run ros_tcp_endpoint default_server_endpoint --ros-args -p ROS_IP:=0.0.0.0
```

**Terminal 2**
```bash
cd ~/ros2_ws
source /opt/ros/humble/setup.bash
source ~/ros2_ws/install/setup.bash
ros2 run panter_control cmd_vel_to_wheel_cmd_node --ros-args -r __ns:=/PANTER -p mode:=skid -p skid_feedforward_torque_nm:=15000.0 -p skid_turn_gain:=3.0
```

**Terminal 3**
```bash
cd ~/ros2_ws
source /opt/ros/humble/setup.bash
source ~/ros2_ws/install/setup.bash
ros2 run panter_control spin_test_node --ros-args -r __ns:=/PANTER -p linear_x:=0.0 -p angular_z:=2.0
```

**Unity**
- abrir escena
- Play

---

## 9. Estado actual del sistema

### Operativo ahora mismo
- `cmd_vel_to_wheel_cmd_node`
- `trajectory_test_node`
- `spin_test_node`
- `ROS_wheelcontroller` en Unity
- `WheelTopViewDebugUI` en Unity
- `ros_tcp_endpoint`

### Preparado pero no completamente integrado en Unity
- `wheel_state`

---

## 10. Siguiente paso recomendado

El siguiente paso lógico es crear un nodo tipo `experiment_runner_node` que lea experimentos desde un fichero YAML y publique `cmd_vel` para los tres ensayos del artículo:

1. recta sobre terreno irregular
2. comparación de radios de giro
3. subida de pendiente

Así se podrán ejecutar los mismos experimentos en:
- `ackermann`
- `skid`
- `hybrid`

sin cambiar el código de cada prueba.
