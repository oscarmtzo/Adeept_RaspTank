
# Sesión de Estudio — Adeept RaspTank
> Fecha: 26 de Febrero de 2026

---

## Índice

1. [Resumen de la Conversación](#1-resumen-de-la-conversación)
2. [Proyecto a Grandes Rasgos](#2-proyecto-a-grandes-rasgos)
3. [BufferPos y PWM de Servos](#3-bufferpos-y-pwm-de-servos)
4. [Cómo Imprimir el PWM de Cada Servo en Consola](#4-cómo-imprimir-el-pwm-de-cada-servo-en-consola)
5. [Arquitectura Completa con Diagramas](#5-arquitectura-completa-con-diagramas)

---

## 1. Resumen de la Conversación

Esta sesión cubrió tres temas principales sobre el proyecto **Adeept RaspTank**:

| Tema | Descripción |
|------|-------------|
| Arquitectura general | Se analizó la estructura cliente-servidor, módulos, inputs y outputs |
| `bufferPos` en RPIservo.py | Se explicó qué almacena, para qué sirve y cómo usarlo |
| Lectura de PWM en consola | Se mostró cómo imprimir los valores PWM de cada servo igual que los estados existentes |

---

## 2. Proyecto a Grandes Rasgos

El proyecto es un **robot controlado remotamente** basado en Raspberry Pi con arquitectura **cliente-servidor TCP**.

### Componentes principales

- **Servidor** (`server/`): Corre en la Raspberry Pi, controla hardware
- **Cliente** (`client/` y `GUI/`): Corre en PC, interfaz gráfica Tkinter
- **Comunicación**: TCP Sockets puertos 10123/10223 + WebSocket para video

### Inputs del usuario

```
forwardStart / backwardStart / leftStart / rightStart   → Movimiento
upStart / downStart / aStart / bStart / cStart / dStart → Servos y brazo
FindColor / FindLine / WatchDog / steady / funEnd        → Modos funcionales
L0-L15                                                   → Control de LEDs
ST1-ST14 / MIN / MAX                                     → Configuración PWM
```

### Outputs al usuario

```
Video Stream FPV         → Cámara en tiempo real
Telemetría               → CPU, GPU, RAM, temperatura, batería
Feedback LEDs WS2812     → Estados visuales (verde=OK, rojo=error)
Sensor ultrasónico       → Distancia en cm
Display OLED             → IP, estado, info del sistema
Confirmaciones TCP       → Estados de servos ST1-ST14
```

---

## 3. BufferPos y PWM de Servos

### ¿Qué es `bufferPos`?

Es un **array de 16 posiciones flotantes** en `RPIservo.py` que almacena posiciones intermedias para **interpolar movimientos suaves**. No contiene el PWM real enviado al servo.

### Variables del sistema de posición

```python
self.initPos    = [300]*16    # Posición de inicio configurada
self.lastPos    = [300]*16    # Última posición conocida (int)
self.nowPos     = [300]*16    # ✅ PWM ACTUAL enviado al servo (int)
self.bufferPos  = [300.0]*16  # Acumulador float para interpolación suave
self.goalPos    = [300]*16    # Posición objetivo del movimiento
```

### Rol del buffer en `moveCert()`

```python
def moveCert(self):
    while self.nowPos != self.goalPos:
        for i in range(0,16):
            if self.lastPos[i] < self.goalPos[i]:
                self.bufferPos[i] += self.pwmGenOut(self.scSpeed[i])/(1/self.scDelay)
                #                    ↑ acumula incrementos decimales
                newNow = int(round(self.bufferPos[i], 0))
                #         ↑ convierte a entero para enviar al hardware
                self.nowPos[i] = newNow
                pwm.set_pwm(i, 0, self.nowPos[i])
```

### ¿De dónde obtener el PWM real?

| Variable | Tipo | ¿Tiene el PWM real? |
|----------|------|---------------------|
| `nowPos[i]` | `int` | ✅ **SÍ — usar este** |
| `bufferPos[i]` | `float` | ❌ No (tiene decimales intermedios) |
| `lastPos[i]` | `int` | Sí (valor anterior) |
| `goalPos[i]` | `int` | Sí (valor futuro/objetivo) |

---

## 4. Cómo Imprimir el PWM de Cada Servo en Consola

Para imprimir los PWM de los servos igual que se imprimen otros estados en el proyecto, agregar dentro del método `run()` de `ServoCtrl` en `RPIservo.py`:

### Opción A — Imprimir en cada ciclo de movimiento (dentro de `scMove`)

```python
def scMove(self):
    if self.scMode == 'init':
        self.moveInit()
    elif self.scMode == 'auto':
        self.moveAuto()
    elif self.scMode == 'certain':
        self.moveCert()
    elif self.scMode == 'wiggle':
        self.moveWiggle()

    # Imprimir PWM actual de todos los servos
    print(f'[ServoCtrl] nowPos: {self.nowPos}')
```

### Opción B — Método dedicado para imprimir estado

```python
def printServoStatus(self):
    print('=' * 60)
    print(f'[ServoCtrl] Modo actual: {self.scMode}')
    for i in range(16):
        print(f'  Servo {i:02d} → PWM actual: {self.nowPos[i]:4d}  |  objetivo: {self.goalPos[i]:4d}  |  buffer: {self.bufferPos[i]:7.2f}')
    print('=' * 60)
```

Llamarlo desde `appserver.py` o `webServer.py`:

```python
# En appserver.py, donde ya se imprimen otros estados:
scGear.printServoStatus()
```

### Opción C — En `__main__` del propio RPIservo.py (para pruebas)

```python
if __name__ == '__main__':
    sc = ServoCtrl()
    sc.start()
    while 1:
        sc.moveAngle(0, 45)
        time.sleep(0.5)

        # Imprime todos los PWM actuales
        print(f'[PWM] Servo 0: {sc.nowPos[0]}')
        print(f'[PWM] Todos:   {sc.nowPos}')
        time.sleep(0.5)
```

### Ejemplo de salida esperada en consola

```
============================================================
[ServoCtrl] Modo actual: auto
  Servo 00 → PWM actual:  345  |  objetivo:  400  |  buffer:  345.00
  Servo 01 → PWM actual:  300  |  objetivo:  300  |  buffer:  300.00
  Servo 02 → PWM actual:  280  |  objetivo:  280  |  buffer:  280.00
  ...
  Servo 14 → PWM actual:  310  |  objetivo:  350  |  buffer:  310.00
  Servo 15 → PWM actual:  300  |  objetivo:  300  |  buffer:  300.00
============================================================
```

---

## 5. Arquitectura Completa con Diagramas

### Diagrama General de Arquitectura

```mermaid
graph TB
    subgraph "Cliente (PC de Usuario)"
        GUI[GUI.py<br/>Interfaz Tkinter]
        ClientConfig[config.py<br/>Configuración]
        GUI --> ClientConfig
    end

    subgraph "Red"
        TCP[TCP Sockets<br/>Puerto 10123/10223]
        WS[WebSockets<br/>Streaming Video]
    end

    subgraph "Servidor (Raspberry Pi)"
        AppServer[appserver.py<br/>Servidor Principal]
        WebServer[webServer.py<br/>Servidor Web]

        subgraph "Módulos de Control"
            Move[move.py<br/>Motores]
            Servo[servo.py<br/>Servomotores]
            LED[LED.py<br/>LEDs WS2812]
            RPIServo[RPIservo.py<br/>Control PWM]
        end

        subgraph "Sensores"
            Ultra[ultra.py<br/>Ultrasonido]
            Camera[camera_opencv.py<br/>Cámara OpenCV]
        end

        subgraph "Funciones Avanzadas"
            FindLine[findline.py<br/>Seguir Línea]
            Tracking[trackingMoudle.py<br/>Seguir Color/Objeto]
            FPV[FPV.py<br/>Control de Cámara]
        end

        subgraph "Display"
            OLED[OLED.py<br/>Pantalla OLED]
            RobotLight[robotLight.py<br/>Efectos LED]
        end
    end

    GUI -->|Comandos| TCP
    TCP -->|Conexión| AppServer
    TCP -->|Conexión| WebServer

    AppServer --> Move
    AppServer --> Servo
    AppServer --> LED
    AppServer --> Ultra
    AppServer --> FindLine
    AppServer --> Tracking

    WebServer --> Camera
    WebServer --> FPV

    Camera -->|Video Stream| WS
    WS -->|Video| GUI

    Move --> RPIServo
    Servo --> RPIServo

    AppServer --> OLED
    LED --> RobotLight

    Ultra -.->|Telemetría| AppServer
    AppServer -.->|Estado| TCP
    TCP -.->|Respuesta| GUI

    style GUI fill:#4CAF50
    style AppServer fill:#2196F3
    style Camera fill:#FF9800
    style Move fill:#9C27B0
```

### Flujo de Comunicación

```mermaid
sequenceDiagram
    participant U as Usuario
    participant G as GUI Cliente
    participant S as Servidor RaspTank
    participant M as Motores
    participant C as Cámara
    participant Sen as Sensores

    U->>G: Inicia aplicación
    G->>S: Conectar (TCP 10123)
    S-->>G: Confirmación de conexión

    par Streaming de Video
        C->>S: Captura frames
        S->>G: Stream video (WebSocket)
        G->>U: Muestra FPV
    and Telemetría
        Sen->>S: Lee sensores (CPU, RAM, Distancia)
        S->>G: Envía telemetría
        G->>U: Muestra estado
    end

    U->>G: Comando (ej: Forward)
    G->>S: 'forwardStart\n'
    S->>M: Activar motores
    M-->>S: Confirmación
    S-->>G: Estado actualizado
    G-->>U: Feedback visual

    U->>G: Comando (ej: Stop)
    G->>S: 'forwardStop\n'
    S->>M: Detener motores
    M-->>S: Confirmación
    S-->>G: Estado actualizado
```

### Inputs y Outputs del Sistema

```mermaid
graph LR
    subgraph "INPUTS DEL USUARIO"
        I1[Comandos de Movimiento<br/>forward, backward, left, right]
        I2[Control de Servos<br/>up, down, servo1-14]
        I3[Control de Brazo<br/>grip open/close, arm up/down]
        I4[Modos Funcionales<br/>FindColor, WatchDog, FindLine]
        I5[Control de LEDs<br/>Colores, Efectos L0-L15]
        I6[Configuración<br/>Presets ST1-ST14, MIN/MAX PWM]
    end

    subgraph "PROCESAMIENTO"
        P[Servidor RaspTank<br/>appserver.py]
    end

    subgraph "OUTPUTS AL USUARIO"
        O1[Video Stream FPV<br/>Cámara en tiempo real]
        O2[Telemetría<br/>CPU, GPU, RAM, Batería]
        O3[Feedback LEDs<br/>Estados visuales]
        O4[Datos Sensores<br/>Distancia ultrasónica]
        O5[Display OLED<br/>IP, Estado, Info]
        O6[Confirmaciones<br/>Estados de servos y comandos]
    end

    I1 --> P
    I2 --> P
    I3 --> P
    I4 --> P
    I5 --> P
    I6 --> P

    P --> O1
    P --> O2
    P --> O3
    P --> O4
    P --> O5
    P --> O6

    style I1 fill:#E3F2FD
    style I2 fill:#E3F2FD
    style I3 fill:#E3F2FD
    style I4 fill:#E3F2FD
    style I5 fill:#E3F2FD
    style I6 fill:#E3F2FD
    style P fill:#2196F3
    style O1 fill:#C8E6C9
    style O2 fill:#C8E6C9
    style O3 fill:#C8E6C9
    style O4 fill:#C8E6C9
    style O5 fill:#C8E6C9
    style O6 fill:#C8E6C9
```

### Módulos del Servidor y sus Responsabilidades

```mermaid
graph TD
    subgraph "Capa de Aplicación"
        AS[appserver.py<br/>Servidor Principal]
        WS[webServer.py<br/>Servidor Web]
        Auto[autorun.py<br/>Auto-inicio]
    end

    subgraph "Capa de Control"
        Move[move.py<br/>Control de Motores<br/>PWM para tracción]
        Servo[servo.py<br/>Control de Servos<br/>14 servomotores]
        LED[LED.py<br/>Control LEDs<br/>WS2812 RGB]
    end

    subgraph "Capa de Sensores"
        Ultra[ultra.py<br/>Ultrasonido<br/>Medición distancia]
        Cam[camera_opencv.py<br/>Cámara<br/>Procesamiento OpenCV]
    end

    subgraph "Capa de Funciones Inteligentes"
        FL[findline.py<br/>Seguir Línea<br/>PID + OpenCV]
        Track[trackingMoudle.py<br/>Seguimiento<br/>Color/Objeto]
        WD[Modo WatchDog<br/>Vigilancia<br/>Auto-navegación]
        Kalman[Kalman_filter.py<br/>Filtro Kalman<br/>Suavizado datos]
    end

    subgraph "Capa de Hardware"
        RPIServo[RPIservo.py<br/>Driver PWM<br/>PCA9685]
        GPIO[GPIO Raspberry Pi<br/>Pines I/O]
    end

    subgraph "Utilidades"
        Info[info.py<br/>Info Sistema<br/>CPU, RAM, Temp]
        OLED[OLED.py<br/>Display<br/>Información local]
        Switch[switch.py<br/>Switches<br/>Control digital]
    end

    Auto --> AS
    Auto --> WS

    AS --> Move
    AS --> Servo
    AS --> LED
    AS --> Ultra
    AS --> FL
    AS --> Track

    WS --> Cam

    FL --> Cam
    Track --> Cam
    WD --> Ultra
    WD --> Move

    Ultra --> Kalman

    Move --> RPIServo
    Servo --> RPIServo
    RPIServo --> GPIO

    AS --> Info
    AS --> OLED
    AS --> Switch

    style AS fill:#1976D2,color:#fff
    style WS fill:#1976D2,color:#fff
    style Move fill:#7B1FA2,color:#fff
    style Servo fill:#7B1FA2,color:#fff
    style Cam fill:#F57C00,color:#fff
    style FL fill:#388E3C,color:#fff
    style Track fill:#388E3C,color:#fff
```

### Modos de Operación

```mermaid
stateDiagram-v2
    [*] --> Desconectado
    Desconectado --> Esperando: autorun.py inicia servidor
    Esperando --> Conectado: Cliente se conecta

    Conectado --> Manual: Modo por defecto
    Conectado --> FindColor: Comando FindColor
    Conectado --> FindLine: Comando FindLine
    Conectado --> WatchDog: Comando WatchDog
    Conectado --> Steady: Comando steady

    Manual --> Manual: Comandos de movimiento/servo

    FindColor --> Manual: Comando funEnd
    FindLine --> Manual: Comando funEnd
    WatchDog --> Manual: Comando funEnd
    Steady --> Manual: Comando funEnd

    FindColor --> FindColor: Sigue objeto de color
    FindLine --> FindLine: Sigue línea negra
    WatchDog --> WatchDog: Navega autónomamente
    Steady --> Steady: Estabiliza orientación

    Conectado --> Desconectado: Pérdida de conexión
    Desconectado --> [*]
```

### Sistema de Posición de Servos (RPIservo.py)

```mermaid
graph LR
    subgraph "Entrada"
        Goal[goalPos int<br/>Posición objetivo]
        Last[lastPos int<br/>Posición anterior]
    end

    subgraph "Procesamiento bufferPos"
        Buffer[bufferPos float<br/>Acumula incrementos<br/>decimales por paso]
        Round[round y int<br/>Convierte a entero]
    end

    subgraph "Salida"
        Now[nowPos int<br/>PWM REAL actual]
        HW[PCA9685<br/>set_pwm]
    end

    Goal --> Buffer
    Last --> Buffer
    Buffer --> Round
    Round --> Now
    Now --> HW

    style Buffer fill:#FF9800,color:#fff
    style Now fill:#4CAF50,color:#fff
    style HW fill:#2196F3,color:#fff
```

### Estructura de Archivos Clave

```mermaid
graph TD
    Root[adeept_rasptank]

    Root --> Auto[autorun.py<br/>Script de auto-inicio]
    Root --> Setup[setup.py<br/>Instalación]
    Root --> Config[config.json<br/>Configuración global]
    Root --> Wifi[wifi_hotspot_manager.sh<br/>Gestión WiFi AP]

    Root --> ClientDir[client/]
    Root --> GUIDir[GUI/]
    Root --> ServerDir[server/]

    ClientDir --> CGUI[GUI.py<br/>Interfaz gráfica]
    ClientDir --> CConf[config.py<br/>Config cliente]
    ClientDir --> CReq[requirements.txt]

    ServerDir --> SApp[appserver.py<br/>Servidor principal]
    ServerDir --> SWeb[webServer.py<br/>Servidor web]
    ServerDir --> SMove[move.py<br/>Motores]
    ServerDir --> SServo[servo.py / RPIservo.py<br/>Servos PWM]
    ServerDir --> SCam[camera_opencv.py<br/>Cámara]
    ServerDir --> SFind[findline.py<br/>Seguir línea]
    ServerDir --> STrack[trackingMoudle.py<br/>Seguimiento]
    ServerDir --> SLED[LED.py<br/>LEDs]
    ServerDir --> SUltra[ultra.py<br/>Ultrasonido]
    ServerDir --> SInfo[info.py<br/>Telemetría]
    ServerDir --> SOLED[OLED.py<br/>Display]

    style Root fill:#FFF9C4
    style ClientDir fill:#BBDEFB
    style GUIDir fill:#BBDEFB
    style ServerDir fill:#C8E6C9
```

---

## Tecnologías Utilizadas

| Capa | Tecnología |
|------|------------|
| Hardware | Raspberry Pi, PCA9685 (PWM I2C), WS2812 LEDs, HC-SR04 (Ultrasonido), Cámara USB/CSI |
| Backend | Python 3.x, Socket TCP, OpenCV, RPi.GPIO, Adafruit_PCA9685 |
| Frontend | Tkinter (GUI Desktop) |
| Procesamiento | Control PID, Filtro de Kalman, Visión por computadora |
| Comunicación | TCP Sockets (10123/10223), HTTP, WebSocket |
| Auto-inicio | systemd / rc.local |
| Networking | WiFi cliente o modo Access Point |

---

*Documento generado el 26 de Febrero de 2026*
ENDOFFILE
