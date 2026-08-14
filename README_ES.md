# reBot-Isaacsim

[简体中文](./README.md) | [English](./README_EN.md) | Español

reBot-Isaacsim es un proyecto de NVIDIA Isaac Sim orientado al robot reBotDM. El código en este repositorio es compatible actualmente solo con reBotDM y puede no funcionar con otras variantes reBotArm o RS.

Notas importantes:

- Compatibilidad: solo reBotDM.
- Versión recomendada de Isaac Sim: 6.0.0.
- Método de instalación recomendado: descargar el paquete zip oficial de Isaac Sim y extraerlo localmente.
- Todos los scripts que interactúan directamente con Isaac Sim deben iniciarse con el `python.sh` oficial incluido en la distribución de Isaac Sim.
- Los scripts emisores que usan hardware deben ejecutarse en el entorno `uv` del repositorio (ver `third_party/reBotArm_control_py`).

## Descripción general de los componentes

Este proyecto ofrece varios emisores para distintos casos de uso:

| Componente | Descripción |
|------|------|
| `gravity_joint_sender` | Modo compensación de gravedad / asa: para robots modificados (pinza retirada, asa acoplada). Permite guiado manual y envía ángulos articulares a Isaac Sim. |
| `isaacsim_ik_sender` | Modo de cinemática inversa (IK): introducir la pose del efector final, resolver IK y enviar ángulos articulares a Isaac Sim. |
| `isaacsim_traj_sender` | Modo trayectoria: IK + generación de trayectoria en espacio articular (MIN_JERK) para movimientos suaves. |
| `isaacsim_joint_test_sender` | Modo de prueba de articulaciones: sin hardware; envía trayectorias predefinidas para validar el receptor y la comunicación. |
| `joint_reader_sender` | Mapeo Real-to-Sim: espejo de ángulos articulares en modo solo-lectura para visualización junto con otros proyectos de control. |

## Arquitectura del sistema

```
┌──────────────────────────────────────────────────────────────────┐
│                         reBot-Isaacsim                           │
│                                                                  │
│   ┌──────────────────────┐        ┌──────────────────────────┐   │
│   │ Emisor (Terminal 2)  │  UDP   │   Receptor (Terminal 1)  │   │
│   │                      │  JSON  │                          │   │
│   │ gravity_joint_sender │──────▶ │ isaacsim_joint_receiver  │   │
│   │                      │ 5005   │                          │   │
│   │  • reBotArm_control  │        │  • Isaac Sim             │   │
│   │    _py entorno uv    │        │  • USD suelo + robot     │   │
│   │  • MIT + FF gravedad │        │  • Sincronización        │   │
│   │  • Guiado a mano OK  │        │  • Pinza biarticulada    │   │
│   └──────────────────────┘        └──────────────────────────┘   │
└──────────────────────────────────────────────────────────────────┘
```

## Estructura de directorios

```
reBot-Isaacsim/
├── pyproject.toml                           # Configuración del workspace de uv
├── README.md
├── README_EN.md                             # Versión en inglés de este README
├── README_ES.md                             # Versión en español de este README
├── reBotArm_Isaacsim/                       # Directorio principal de ejemplos
│   ├── gravity_joint_sender.py              # Emisor hardware (ejecutar dentro de uv)
│   ├── isaacsim_ik_sender.py                # Emisor IK (iniciar con python.sh de Isaac)
│   ├── isaacsim_traj_sender.py              # Emisor trayectoria (iniciar con python.sh de Isaac)
│   ├── isaacsim_joint_test_sender.py        # Emisor de prueba (usar python.sh si importa Isaac Sim)
│   ├── joint_reader_sender.py               # Emisor solo-lectura
│   ├── isaacsim_joint_receiver.py           # Receptor Isaac Sim (iniciar con python.sh)
│   ├── live_sync.py                         # Script de guía de arranque
│   └── ...
├── third_party/
│   └── reBotArm_control_py/                 # Biblioteca de control (entorno uv independiente)
│       ├── pyproject.toml
│       └── ...
├── urdf/
│   └── ...                                  # URDF / archivos de configuración
├── usd/
│   └── reBot_B601_DM/
│       └── reBot_B601_DM.usda               # Asset reBotDM
└── ...
```

Nota: ejecute los módulos Python directamente y seleccione el lanzador correcto:

- Use `uv run python ...` para scripts que se ejecutan en el entorno `uv` del proyecto (emisores hardware).
- Use `"$ISAACSIM_ROOT/python.sh"` para scripts que requieren el runtime nativo de Isaac Sim.

## Dependencias y requisitos previos

| Componente | Requisito |
|------|------|
| Modelo | Código compatible con reBotDM únicamente |
| Isaac Sim | 6.0.0; se recomienda la versión zip oficial |
| Ruta de Isaac Sim | Se recomienda establecer `ISAACSIM_ROOT` al directorio de la release |
| Interfaz serial/CAN | Por defecto `ttyACM0`, configurable en `third_party/reBotArm_control_py/config/rebotarm_dm.yaml` |
| Python | Scripts emisores bajo Python 3.10+ administrado por `uv`; runtime de Isaac Sim provisto por `python.sh` |
| uv | Recomendado para gestionar dependencias |
| reBotArm_control_py | Ejecutar `uv sync` dentro de `third_party/reBotArm_control_py` |

### Instalación recomendada de Isaac Sim

Descargue la release zip oficial y extraiga el contenido en una ruta estable. Después, establezca `ISAACSIM_ROOT` apuntando a la carpeta de la release. Ejemplo:

```bash
mkdir -p /home/seeed/IsaacSim
# descomprimir el zip oficial en ese directorio
# tras extraer debería aparecer algo como:
# /home/seeed/IsaacSim/_build/linux-x86_64/release/python.sh

export ISAACSIM_ROOT=/home/seeed/IsaacSim/_build/linux-x86_64/release
```

### Comprobar interfaz serial/CAN

```bash
ls /dev/ttyACM*
sudo chmod 666 /dev/ttyACM*
```

## Preparación del entorno

### 1. Instalar Isaac Sim 6.0.0

Use la release zip oficial y verifique que `python.sh` exista en la carpeta de la release.

### 2. Entorno `reBotArm_control_py`

```bash
cd third_party/reBotArm_control_py
uv sync
```

## Arranque (modo dos terminales)

Se requieren dos terminales independientes. El Terminal 1 es el receptor Isaac Sim; el Terminal 2 es el emisor hardware. Puede ejecutar ambos en la misma máquina ajustando `DEFAULT_SIM_HOST` y `DEFAULT_REBOT_ARM_HOST` a `127.0.0.1`.

### Terminal 1 — Iniciar el receptor Isaac Sim

Inicie el receptor con `python.sh` de Isaac Sim:

```bash
"/ruta/a/tu/isaacsim/release/python.sh" \
    /ruta/a/reBot-Isaacsim/reBotArm_Isaacsim/isaacsim_joint_receiver.py
```

Salida esperada:

- Se inicia la GUI de Isaac Sim.
- Se carga el suelo y el asset reBotDM.
- El receptor escucha en UDP `DEFAULT_SIM_HOST:5005`.
- El receptor espera la conexión del emisor.

### Terminal 2 — Iniciar el emisor hardware

Inicie el emisor después de haber arrancado el receptor:

```bash
cd /ruta/a/reBot-Isaacsim
uv run python reBotArm_Isaacsim/gravity_joint_sender.py
```

Salida esperada:

- El robot físico se conecta y se activa MIT con compensación de gravedad.
- El robot puede ser guiado a mano.
- Los ángulos articulares se transmiten por UDP a ~60 Hz.

### Otros scripts relacionados con Isaac Sim

Inicie otros scripts de Isaac Sim con `python.sh`:

```bash
export ISAACSIM_ROOT=/home/seeed/IsaacSim/_build/linux-x86_64/release
"$ISAACSIM_ROOT/python.sh" reBotArm_Isaacsim/isaacsim_ik_sender.py
"$ISAACSIM_ROOT/python.sh" reBotArm_Isaacsim/isaacsim_traj_sender.py
"$ISAACSIM_ROOT/python.sh" reBotArm_Isaacsim/isaacsim_joint_test_sender.py
```

## Formatos de entrada (IK / Traj)

```
x y z                       # posición (m), mantener la orientación
x y z r p y                 # posición + orientación (m/grados)
q j1 j2 j3 j4 j5 j6         # ángulos articulares directos (grados)
gripper <0~1>                # actualizar solo la pinza
```

Modo trayectoria:

```
x y z
x y z r p y
q j1 j2 j3 j4 j5 j6
gripper <0~1>
speed <scale>
resync
```

### Modo solo-lectura (`joint_reader_sender`)

```bash
cd /ruta/a/reBot-Isaacsim
uv run python reBotArm_Isaacsim/joint_reader_sender.py
```

Salida esperada:

- Lee ángulos articulares en modo pasivo (no envía comandos de control).
- Envía los ángulos por UDP a ~60 Hz.
- Refleja el movimiento del robot físico en Isaac Sim para visualización.

## Protocolo de comunicación

JSON sobre UDP en `127.0.0.1:5005`.

Ejemplo de payload por datagrama:

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
| `sequence` | int | Número de secuencia monótonamente creciente |
| `timestamp` | float | Marca de tiempo Unix (segundos) |
| `joint_positions` | float[6] | Ángulos de las primeras 6 articulaciones (rad) |
| `gripper_position` | float | Posición objetivo de la pinza (m) |

Mapeos de pinza (ejemplos):

| Emisor | Mapeo a `gripper_position` (m) |
|------|------|
| `gravity_joint_sender` | `gripper_q × 0.03` (`GRIPPER_POSITION_SCALE = 0.03`) |
| `joint_reader_sender` | `gripper_q × 0.007` (`GRIPPER_POSITION_SCALE = 0.007`) |
| `isaacsim_traj_sender` | `ratio × 0.045` (entrada `gripper <0~1>`, recortada a 0.045 m) |
| `isaacsim_ik_sender` | `ratio ∈ [0,1]` enviado como metros |

## Parámetros de configuración

### Emisor (`gravity_joint_sender.py`)

| Parámetro | Por defecto | Descripción |
|------|--------|------|
| `ARM_JOINT_COUNT` | 6 | Número de articulaciones |
| `DEFAULT_PORT` | 5005 | Puerto UDP |
| `DEFAULT_SEND_HZ` | 60.0 | Frecuencia de envío (Hz) |
| `GRIPPER_POSITION_SCALE` | 0.03 | Factor de escala de la pinza |
| `position_alpha` | 0.2 | Coeficiente de filtro pasa-bajo |

### Receptor (`isaacsim_joint_receiver.py`)

| Parámetro | Por defecto | Descripción |
|------|--------|------|
| `ARM_JOINT_COUNT` | 6 | Número de articulaciones |
| `DEFAULT_PORT` | 5005 | Puerto UDP |
| `DEFAULT_RENDER_HZ` | 120.0 | Frecuencia de render de la simulación (Hz) |
| `ROBOT_PRIM_PATH` | `/World/reBotArm` | Ruta del prim del robot en Isaac Sim |
| `ASSET_RELATIVE_PATH` | `usd/reBot_B601_DM/reBot_B601_DM.usda` | Ruta relativa del asset USD |

## Resolución de problemas

### `OSError: [Errno 98] Address already in use`

Puerto 5005 ocupado. Identifique y detenga el proceso que lo usa:

```bash
sudo lsof -i :5005
kill <PID>
```

### Asset de Isaac Sim no encontrado

```bash
ls usd/reBot_B601_DM/reBot_B601_DM.usda
```

### Problemas serial / CAN

```bash
can_restart can0
ip -details link show can0 | grep bitrate
```

### Ángulos articulares fuera de sincronía

- Confirme que los puertos sender/receiver coinciden (5005).
- Verifique que el emisor muestre repetidamente `[send]`.
- Verifique que el receptor muestre repetidamente `[recv]`.
- Use `isaacsim_joint_test_sender.py` para descartar problemas de hardware.

## Componentes y entornos Python

| Componente | Entorno Python | Método de lanzamiento |
|------|------------|---------|
| Emisor hardware | Entorno `uv` del proyecto + `reBotArm_control_py` | `uv run python reBotArm_Isaacsim/gravity_joint_sender.py` |
| Emisor solo-lectura | Entorno `uv` del proyecto | `uv run python reBotArm_Isaacsim/joint_reader_sender.py` |
| Receptor Isaac Sim | Python oficial de Isaac Sim (`python.sh`) | `"$ISAACSIM_ROOT/python.sh" reBotArm_Isaacsim/isaacsim_joint_receiver.py` |
| Scripts Isaac Sim | Python oficial de Isaac Sim (`python.sh`) | `"$ISAACSIM_ROOT/python.sh" reBotArm_Isaacsim/isaacsim_ik_sender.py` |
