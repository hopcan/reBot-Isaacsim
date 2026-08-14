# reBot-Isaacsim

[简体中文](./README.md) | [English](./README_EN.md) | Español

reBot-Isaacsim es un proyecto de NVIDIA Isaac Sim para el robot reBotDM. El código actual solo es compatible con reBotDM y no es adecuado para otros modelos.

> Notas importantes:
> - Compatibilidad actual: solo reBotDM
> - Versión recomendada de Isaac Sim: 6.0.0
> - Método recomendado de instalación: descargar el zip oficial y extraerlo localmente
> - Todos los scripts relacionados directamente con Isaac Sim deben lanzarse a través del `python.sh` oficial
> - El emisor del robot real sigue usando el entorno `uv` del repositorio

## Descripción general de los componentes

Este proyecto ofrece varios modos de envío para diferentes casos de uso:

| Componente | Descripción |
|------|------|
| `gravity_joint_sender` | **Modo de compensación de gravedad / empuñadura**: para robots modificados (pinza retirada y empuñadura instalada), el robot puede moverse manualmente bajo compensación de gravedad y sus ángulos articulares se sincronizan con Isaac Sim en tiempo real. |
| `isaacsim_ik_sender` | **Modo de cinemática inversa (IK)**: se indica la pose del efector final, se resuelve la IK para obtener los ángulos articulares y se envían a Isaac Sim. |
| `isaacsim_traj_sender` | **Modo de trayectoria (Traj)**: se añade planificación de trayectoria en el espacio articular (perfil temporal MIN_JERK) sobre la base de IK para un control suave del movimiento. |
| `isaacsim_joint_test_sender` | **Modo de prueba de articulaciones**: no necesita robot real; envía trayectorias predefinidas de ángulos articulares para verificar el receptor y la comunicación de Isaac Sim. |
| `joint_reader_sender` | **Modo de mapeo Real-to-Sim**: lee ángulos articulares en modo de solo lectura y los mapea a Isaac Sim, útil para combinarse con otros proyectos de control (por ejemplo, cuando el robot real realiza otras tareas, el mismo movimiento se refleja en Isaac Sim para visualización). |

## Arquitectura del sistema

```
┌──────────────────────────────────────────────────────────────────┐
│                         reBot-Isaacsim                           │
│                                                                  │
│   ┌──────────────────────┐        ┌─────────────────────────┐    │
│   │ Emisor (Terminal 2)  │  UDP   │   Receptor (Terminal 1) │    │
│   │                      │  JSON  │                         │    │
│   │ gravity_joint_sender │──────▶ │ isaacsim_joint_receiver │    │
│   │                      │ 5005   │                         │    │
│   │  • reBotArm_control  │        │  • Simulación Isaac Sim │    │
│   │    _py entorno uv    │        │  • Suelo + USD robot    │    │
│   │  • MIT + gravedad    │        │  • Sincronización       │    │
│   │  • Empuje manual OK  │        │  • Pinza de dos ejes    │    │
│   └──────────────────────┘        └─────────────────────────┘    │
└──────────────────────────────────────────────────────────────────┘
```

## Estructura del directorio

```
reBot-Isaacsim/
├── pyproject.toml                           # Configuración del workspace en uv
├── README.md
├── README_EN.md
├── README_ES.md
├── reBotArm_Isaacsim/                       # Directorio principal de ejemplos
│   ├── gravity_joint_sender.py              # Emisor del robot real (entorno uv)
│   ├── isaacsim_ik_sender.py                # Emisor IK (debe usar python.sh de Isaac)
│   ├── isaacsim_traj_sender.py              # Emisor de trayectoria (debe usar python.sh de Isaac)
│   ├── isaacsim_joint_test_sender.py        # Emisor de prueba (usar python.sh cuando haga falta)
│   ├── joint_reader_sender.py               # Script de mapeo de sólo lectura
│   ├── isaacsim_joint_receiver.py           # Receptor de Isaac Sim (debe usar python.sh de Isaac)
│   ├── live_sync.py                         # Script de guía de arranque
│   └── ...
├── third_party/
│   └── reBotArm_control_py/                 # Biblioteca de control (entorno uv independiente)
│       ├── pyproject.toml
│       └── ...
├── urdf/
│   └── ...                                  # URDF / configuración del robot
├── usd/
│   └── reBot_B601_DM/
│       └── reBot_B601_DM.usda               # Asset reBotDM
└── ...
```

> El repositorio ya no conserva scripts envoltorios como `run_sender.sh` / `run_isaacsim_receiver.sh`; ejecute directamente los scripts Python pertinentes y elija el método de arranque descrito a continuación.

## Dependencias y requisitos previos

| Componente | Requisito |
|------|------|
| Modelo del robot | El código actual solo es compatible con reBotDM |
| Isaac Sim | 6.0.0; se recomienda usar el paquete zip oficial y extraerlo localmente |
| Ruta de Isaac Sim | Se recomienda establecer `ISAACSIM_ROOT` al directorio de instalación oficial |
| Interfaz | Por defecto es `ttyACM0`, que se puede modificar en `third_party/reBotArm_control_py/config/rebotarm_dm.yaml` |
| Python | Para el emisor del robot, Python 3.10+ gestionado por `uv`; el runtime de Isaac Sim se proporciona mediante `python.sh` |
| uv | Se recomienda para gestionar las dependencias del proyecto actual y de `third_party/reBotArm_control_py` |
| reBotArm_control_py | `uv sync` ya se ha ejecutado dentro de `third_party/reBotArm_control_py` |

### Método recomendado de instalación de Isaac Sim

Este proyecto recomienda usar la release oficial de Isaac Sim en zip y extraerla en un directorio fijo, por ejemplo:

```bash
# Ejemplo: extraer en una carpeta común
mkdir -p /home/seeed/IsaacSim
# Extrae el zip oficial en este directorio
# Después de extraer, deberías obtener algo como:
# /home/seeed/IsaacSim/python.sh
```

### Comprobar el puerto USB2CAN

```bash
# Ver el puerto serie USB2CAN para asegurarse de que se detecta
ls ttyACM*

# Dar permisos al puerto
sudo chmod 666 /dev/ttyACM*
```

## Preparación del entorno

### 1. Instalar Isaac Sim 6.0.0

Utilice el paquete zip oficial y no dependa de un entorno Python incompleto ni mezcle otros runtimes de Isaac Sim.
Enlaces y recursos oficiales:
https://docs.isaacsim.omniverse.nvidia.com/6.0.0/installation/quick-install.html
https://docs.isaacsim.omniverse.nvidia.com/6.0.0/installation/download.html#isaac-sim-latest-release

Ejecute los siguientes comandos para instalar y definir variables de entorno:

```bash
mkdir ~/isaacsim
cd ~/Downloads
unzip "isaac-sim-standalone-6.0.0-linux-x86_64.zip" -d ~/isaacsim
cd ~/isaacsim
./post_install.sh
./isaac-sim.sh
```

### 2. Entorno `reBotArm_control_py`

```bash
cd third_party/reBotArm_control_py
uv sync
```

## Arranque (modo dual-terminal)

Se requieren dos terminales separados. **El terminal 1 es el receptor de Isaac Sim y el terminal 2 es el emisor del robot real**.
La ejecución local es posible configurando `DEFAULT_SIM_HOST` y `DEFAULT_REBOT_ARM_HOST` ambos a `127.0.0.1`.

### Terminal 1 — Iniciar el receptor de Isaac Sim

Todos los scripts relacionados con Isaac Sim deben lanzarse directamente con el `python.sh` oficial:

```bash
"ruta_a_tu_carpeta_isaacsim"/python.sh  "ruta_a_tu_workspace"/reBot-Isaacsim/reBotArm_Isaacsim/isaacsim_joint_receiver.py
```

**Salida esperada:**
- Arranca la interfaz gráfica de Isaac Sim
- Carga el suelo y los assets de reBotDM
- Escucha en UDP `DEFAULT_SIM_HOST:5005`
- Espera a que el emisor se conecte

### Terminal 2 — Emisor del robot real

**Orden de arranque: receptor primero, luego emisor.**

```bash
cd /ruta/a/reBot-Isaacsim
uv run python reBotArm_Isaacsim/gravity_joint_sender.py
```

**Comportamiento esperado:**
- Se conecta al robot real y activa la compensación MIT + feed-forward de gravedad
- El robot puede ser empujado y movido manualmente
- Los ángulos articulares se transmiten continuamente por UDP a 60 Hz

#### Otros scripts relacionados con Isaac Sim

Si está ejecutando scripts relacionados con `isaacsim`, siempre arránquelos con el `python.sh` oficial, por ejemplo:

```bash
export ISAACSIM_ROOT="ruta/a/tu/carpeta/isaacsim"   # ej.: /home/seeed/IsaacSim/
"$ISAACSIM_ROOT/python.sh" reBotArm_Isaacsim/isaacsim_ik_sender.py
"$ISAACSIM_ROOT/python.sh" reBotArm_Isaacsim/isaacsim_traj_sender.py
"$ISAACSIM_ROOT/python.sh" reBotArm_Isaacsim/isaacsim_joint_test_sender.py
```

Notas:
- `gravity_joint_sender.py` / `joint_reader_sender.py` y scripts similares que conectan con el robot real o paquetes UDP suelen ejecutarse en el entorno `uv` del proyecto actual.
- Los scripts `isaacsim_*` deben usar el `python.sh` oficial de Isaac; de lo contrario, `SimulationApp` / el runtime de Python de Isaac Sim puede no estar disponible.

#### Ejemplos de formato de entrada (emisor IK / Traj)

**Formato de entrada (una línea por comando):**
```
x y z                       # posición (m), mantener la orientación actual
x y z r p y                 # posición + orientación (m/deg)
q j1 j2 j3 j4 j5 j6         # enviar ángulos articulares directamente (deg)
gripper <0~1>                # actualizar solo la pinza
```

**Entrada en modo trayectoria:**
```
x y z                       # posición (m)
x y z r p y                 # posición + orientación (m/deg)
q j1 j2 j3 j4 j5 j6         # comando directo en espacio articular (deg)
gripper <0~1>                # actualizar solo la pinza
speed <scale>                # ajustar la relación de duración de la trayectoria
resync                       # volver a leer los ángulos articulares actuales desde el lado de la simulación
```

#### Modo de mapeo de sólo lectura (`joint_reader_sender`)

Lee los ángulos articulares y los mapea a Isaac Sim. Esto es útil cuando el robot real está ejecutando otras tareas y quieres visualizar el movimiento en Isaac Sim.

```bash
cd /ruta/a/reBot-Isaacsim
uv run python reBotArm_Isaacsim/joint_reader_sender.py
```

**Comportamiento esperado:**
- Lee los ángulos articulares solo en modo pasivo, sin enviar comandos de control
- Los ángulos articulares se transmiten continuamente por UDP a 60 Hz
- Cuando el robot real es controlado por otro proyecto, su movimiento también puede visualizarse en Isaac Sim

## Protocolo de comunicación

JSON por UDP en el puerto `127.0.0.1:5005`.

**Payload enviado por el emisor en cada frame:**

```json
{
  "sequence": 123,
  "timestamp": 1718000000.123,
  "joint_positions": [0.0, 0.1, 0.2, -0.1, 0.0, -0.02],
  "gripper_position": 0.05
}
```

| Campo | Tipo | Descripción |
|------|------|------|
| `sequence` | int | Número de secuencia creciente |
| `timestamp` | float | Marca temporal Unix (segundos) |
| `joint_positions` | float[6] | Primeros 6 ángulos articulares (rad) |
| `gripper_position` | float | Posición objetivo del dedo de la pinza (m); cada emisor usa un método de conversión distinto (ver tabla de abajo) |

**Cadena de control de la pinza:**
El receptor toma el `gripper_position` recibido directamente como posición objetivo de los ejes deslizantes izquierdo y derecho y lo recorta a `[0, límite]` para cada dedo (límite en USD: ambos dedos son 0.05 m; ambos se mueven mediante el mismo motor a través de un único engranaje, así que el recorrido es estrictamente 1:1). El receptor no aplica escalado adicional. La conversión de cada emisor a `gripper_position` es la siguiente:

| Emisor | Conversión a `gripper_position` (m) |
|------|------|
| `gravity_joint_sender` | `gripper_q × 0.03` (`GRIPPER_POSITION_SCALE = 0.03`) |
| `joint_reader_sender` | `gripper_q × 0.007` (`GRIPPER_POSITION_SCALE = 0.007`) |
| `isaacsim_traj_sender` | `ratio × 0.045` (entrada `gripper <0~1>`, recortada a 0.045 m) |
| `isaacsim_ik_sender` | `ratio ∈ [0, 1]` sin procesar, enviado directamente en metros; por eso cuando el ratio alcanza o supera el límite del dedo, ese dedo queda completamente abierto |

## Parámetros de configuración

### Emisor (`gravity_joint_sender.py`)

| Parámetro | Predeterminado | Descripción |
|------|--------|------|
| `ARM_JOINT_COUNT` | 6 | Número de articulaciones |
| `DEFAULT_PORT` | 5005 | Puerto UDP |
| `DEFAULT_SEND_HZ` | 60.0 | Frecuencia de envío (Hz) |
| `GRIPPER_POSITION_SCALE` | 0.03 | Factor de escala de ángulo a posición de la pinza |
| `position_alpha` | 0.2 | Coeficiente de filtro pasa-bajos |

### Receptor (`isaacsim_joint_receiver.py`)

| Parámetro | Predeterminado | Descripción |
|------|--------|------|
| `ARM_JOINT_COUNT` | 6 | Número de articulaciones |
| `DEFAULT_PORT` | 5005 | Puerto UDP |
| `DEFAULT_RENDER_HZ` | 120.0 | Frecuencia de renderización de la simulación (Hz) |
| `ROBOT_PRIM_PATH` | `/World/reBotArm` | Ruta del prim del robot en Isaac Sim |
| `ASSET_RELATIVE_PATH` | `usd/RS-rebot-dev-arm/RS-rebot-dev-arm.usda` | Ruta relativa del asset USD |

## Resolución de problemas

### `OSError: [Errno 98] Address already in use`

El puerto 5005 ya está en uso. Comprueba qué proceso lo está ocupando y deténlo:

```bash
# Ver qué proceso está usando el puerto
sudo lsof -i :5005

# Terminarlo (reemplaza PID por el valor real)
kill <PID>
```

### Asset de Isaac Sim no encontrado

Confirma que el asset USD existe o revisa si `REPO_ROOT` es correcto:

```bash
ls usd/reBot_B601_DM/reBot_B601_DM.usda
```

### El bus CAN no está listo

Asegúrate de que la interfaz CAN esté activa y que la velocidad de bits sea correcta:

```bash
can_restart can0
# comprobar:
ip -details link show can0 | grep bitrate
```

### Ángulos articulares desincronizados

- Verifica que el emisor y el receptor usen el mismo puerto (5005)
- Comprueba si el log del emisor sigue imprimiendo `[send]`
- Comprueba si el log del receptor sigue imprimiendo `[recv]`
- Prueba `isaacsim_joint_test_sender.py` para descartar problemas de hardware

## Componentes y entornos Python

| Componente | Entorno Python | Método de arranque |
|------|------------|---------|
| Emisor del robot real | Entorno actual `uv` del repositorio + `reBotArm_control_py` | `uv run python reBotArm_Isaacsim/gravity_joint_sender.py` |
| Emisor de mapeo sólo lectura | Entorno actual `uv` del repositorio | `uv run python reBotArm_Isaacsim/joint_reader_sender.py` |
| Receptor de Isaac Sim | Python oficial de Isaac Sim (`python.sh`) | `"$ISAACSIM_ROOT/python.sh" reBotArm_Isaacsim/isaacsim_joint_receiver.py` |
| Scripts relacionados con Isaac Sim | Python oficial de Isaac Sim (`python.sh`) | `"$ISAACSIM_ROOT/python.sh" reBotArm_Isaacsim/isaacsim_ik_sender.py` |
