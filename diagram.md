## Diagrama rápido de paquetes y nodos

```mermaid
flowchart LR
    subgraph Docker["Contenedor Docker: ros2_humble_unity_dev"]
        subgraph ROS2["ROS 2 workspace"]
            P1["paquete: panter_control"]
            P2["paquete: wheel_control_msgs"]
            P3["paquete: wheel_state"]
            P4["paquete: ros_tcp_endpoint"]

            N1["nodo: trajectory_test_node"]
            N2["nodo: spin_test_node"]
            N3["nodo: cmd_vel_to_wheel_cmd_node"]
            N4["nodo: default_server_endpoint"]
        end
    end

    subgraph Unity["Unity"]
        U1["ROS_wheelcontroller"]
        U2["WheelTopViewDebugUI"]
    end

    N1 -->|/PANTER/cmd_vel| N3
    N2 -->|/PANTER/cmd_vel| N3
    N3 -->|/PANTER/wheel_cmd| N4
    N4 -->|TCP ROS-Unity| U1

    U1 -->|/PANTER/joint_states| N4
    U1 -. opcional .->|/PANTER/wheel_state| N4

    P1 --- N1
    P1 --- N2
    P1 --- N3
    P2 --- N3
    P3 -. define msg .- U1
    P4 --- N4

    U2 -. solo visualización .- U1
```

## Flujo principal

```text
trajectory_test_node / spin_test_node
            ↓
       /PANTER/cmd_vel
            ↓
   cmd_vel_to_wheel_cmd_node
            ↓
      /PANTER/wheel_cmd
            ↓
    default_server_endpoint
            ↓
            Unity
            ↓
     /PANTER/joint_states
```

## Qué hace cada pieza

* `panter_control`

  * contiene los nodos de prueba y conversión
  * `trajectory_test_node`
  * `spin_test_node`
  * `cmd_vel_to_wheel_cmd_node`

* `wheel_control_msgs`

  * define `WheelCommand`

* `wheel_state`

  * define `WheelState`

* `ros_tcp_endpoint`

  * crea el puente ROS 2 ↔ Unity con `default_server_endpoint`

* `ROS_wheelcontroller` en Unity

  * recibe `/PANTER/wheel_cmd`
  * aplica el control a las ruedas
  * publica `/PANTER/joint_states`

* `WheelTopViewDebugUI` en Unity

  * solo visualiza datos
  * no es un nodo ROS

```
```
