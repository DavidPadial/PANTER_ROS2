## Diagrama rápido

- **Contenedor Docker**
  - `ros2_humble_unity_dev`
    - **ROS 2 workspace**
      - **paquete `panter_control`**
        - `trajectory_test_node`
          - publica `/PANTER/cmd_vel`
        - `spin_test_node`
          - publica `/PANTER/cmd_vel`
        - `cmd_vel_to_wheel_cmd_node`
          - suscribe `/PANTER/cmd_vel`
          - publica `/PANTER/wheel_cmd`
      - **paquete `wheel_control_msgs`**
        - define `WheelCommand`
      - **paquete `wheel_state`**
        - define `WheelState`
      - **paquete `ros_tcp_endpoint`**
        - `default_server_endpoint`
          - puente ROS 2 ↔ Unity
- **Unity**
  - `ROS_wheelcontroller`
    - recibe `/PANTER/wheel_cmd`
    - publica `/PANTER/joint_states`
    - publica opcionalmente `/PANTER/wheel_state`
  - `WheelTopViewDebugUI`
    - visualización y depuración

## Flujo principal

`trajectory_test_node` o `spin_test_node`  
→ `/PANTER/cmd_vel`  
→ `cmd_vel_to_wheel_cmd_node`  
→ `/PANTER/wheel_cmd`  
→ `default_server_endpoint`  
→ `ROS_wheelcontroller` en Unity  
→ `/PANTER/joint_states`
