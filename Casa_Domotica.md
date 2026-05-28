# Casa Domótica Inteligente — Proyecto Final
### Aplicaciones en Sistemas Embebidos | Fundación Universitaria Compensar

> Sistema domótico completo con control por voz, visión artificial (YOLO) y tablero de visualización en la nube. Integra Arduino, Python y múltiples módulos inteligentes para automatizar una casa completa.

---

## 📋 Tabla de Contenidos

1. [Descripción General](#descripción-general)
2. [Arquitectura del Sistema](#arquitectura-del-sistema)
3. [Módulos de la Casa](#módulos-de-la-casa)
   - [Garaje Inteligente (Parqueadero)](#-garaje-inteligente--parqueadero)
   - [Cocina Inteligente](#-cocina-inteligente)
   - [Habitación Inteligente](#-habitación-inteligente)
   - [Baño Inteligente](#-baño-inteligente)
   - [Sala Inteligente](#-sala-inteligente)
   - [Nube y Dashboard](#-nube-y-dashboard)
4. [Control por Voz](#control-por-voz)
5. [Protocolo de Comunicación Serial](#protocolo-de-comunicación-serial)
6. [Hardware y Componentes](#hardware-y-componentes)
7. [Instalación y Dependencias](#instalación-y-dependencias)
8. [Ejecución del Sistema](#ejecución-del-sistema)
9. [Plan de Pruebas](#plan-de-pruebas)
10. [Estructura del Repositorio](#estructura-del-repositorio)

---

## Descripción General

Este proyecto implementa una **casa domótica completa** con 5 módulos inteligentes interconectados. Cada módulo tiene sensores, actuadores y un sistema de control que responde a comandos de voz. Los datos de todos los módulos se envían a la nube para visualización en tiempo real mediante un dashboard.

**Tecnologías clave:**
- `Arduino UNO` — microcontrolador de control
- `Python 3` — lógica principal, reconocimiento de voz, visión artificial
- `YOLO (Ultralytics)` — detección de vehículos en tiempo real
- `OpenCV` — procesamiento de video
- `SpeechRecognition` — reconocimiento de voz en español
- `PySerial` — comunicación USB entre Python y Arduino

---

## Arquitectura del Sistema

```
┌─────────────────────────────────────────────────────────────┐
│                    CAPA DE PRESENTACIÓN                      │
│   Dashboard Nube  │  Monitor Serial  │  LEDs Indicadores    │
└──────────────────────────┬──────────────────────────────────┘
                           │
┌──────────────────────────▼──────────────────────────────────┐
│                    CAPA DE APLICACIÓN                        │
│     Python (YOLO + Voz + OpenCV + PySerial)                 │
└──────────────────────────┬──────────────────────────────────┘
                           │
┌──────────────────────────▼──────────────────────────────────┐
│               CAPA DE COMUNICACIÓN                           │
│              USB Serial (9600 baud)                          │
└──────────────────────────┬──────────────────────────────────┘
                           │
┌──────────────────────────▼──────────────────────────────────┐
│              CAPA DE CONTROL (Arduino UNO)                   │
│   Lógica de control │ Gestión de pines │ Temporizadores      │
│                                                              │
│   Pin 7  → Relé (Motor Parqueadero)                         │
│   Pin 11 → LED Verde (Puerta Abierta)                       │
│   Pin 12 → LED Rojo (Puerta Cerrada)                        │
│   Pins varios → Motores, LEDs RGB, Buzzer de cada módulo    │
└─────────────────────────────────────────────────────────────┘
```

---

## Módulos de la Casa
---------------------------------

### Garaje Inteligente / Parqueadero

**Función:** Detecta vehículos con cámara y YOLO. Abre/cierra la puerta automáticamente según la ocupación del parqueadero.

#### ¿Qué es YOLO?
YOLO (*You Only Look Once*) es un algoritmo de visión artificial basado en redes neuronales convolucionales (CNN) que detecta objetos en tiempo real procesando la imagen completa en una sola pasada. En este proyecto identifica autos, motos, camiones y buses.

#### Componentes del Garaje

| Componente | Especificación | Función |
|---|---|---|
| Microcontrolador | Arduino UNO (ATmega328P) | Control del motor y LEDs |
| Relé | SRD-05VDC 5V (Pin 7) | Activa/desactiva el motor |
| Motor DC | 9V, 100 RPM | Mueve la puerta (sube/baja) |
| Cámara | Celular (IP Webcam) | Captura video para YOLO |
| LED Verde | Pin 11 | Indica puerta abierta |
| LED Rojo | Pin 12 | Indica puerta cerrada |

#### Protocolo de Comandos Serial

```
COMANDO PYTHON → ARDUINO     ACCIÓN
─────────────────────────────────────────────────
"G_OPEN\n"    →  Activa relé (Pin 7), gira motor 5 seg hacia adelante
                 LED Verde ON, LED Rojo OFF
"G_CLOSE\n"   →  Activa relé (Pin 7), gira motor 5 seg en reversa
                 LED Rojo ON, LED Verde OFF
"ESTADO\n"    →  Arduino responde: "ABIERTO" o "CERRADO"
```

#### Diagrama de Flujo — Garaje

```
        ┌─────────────┐
        │    INICIO   │
        └──────┬──────┘
               │
        ┌──────▼──────────────┐
        │ Inicializar Cámara  │
        │ y Conexión Serial   │
        └──────┬──────────────┘
               │
        ┌──────▼──────────────┐
        │  YOLO detecta frame │◄─────────────┐
        └──────┬──────────────┘              │
               │                             │
       ┌───────┼───────┐                     │
       ▼       ▼       ▼                     │
  Vehículo  Sin     Comando                  │
  detectado vehículo  voz                    │
       │       │       │                     │
  ┌────▼──┐ ┌──▼────┐ ┌▼──────┐             │
  │G_OPEN │ │G_CLOSE│ │Según  │             │
  │Serial │ │Serial │ │frase  │             │
  └────┬──┘ └──┬────┘ └┬──────┘             │
       └───────┴────────┘                    │
               │                             │
        ┌──────▼──────────────┐              │
        │  Esperar 5 seg      │──────────────┘
        │  Repetir ciclo      │
        └─────────────────────┘
```

#### Código Python — Detección con YOLO

```python
import cv2
import serial
import time
from ultralytics import YOLO

# Configuración
PUERTO_SERIAL = 'COM4'
BAUD_RATE = 9600
CLASES_VEHICULOS = [2, 3, 5, 7]  # car, motorcycle, bus, truck en COCO

modelo = YOLO('yolov8n.pt')
arduino = serial.Serial(PUERTO_SERIAL, BAUD_RATE, timeout=1)
time.sleep(2)

cap = cv2.VideoCapture(0)

while True:
    ret, frame = cap.read()
    if not ret:
        break

    resultados = modelo(frame)
    vehiculos_detectados = 0

    for r in resultados:
        for box in r.boxes:
            if int(box.cls) in CLASES_VEHICULOS:
                vehiculos_detectados += 1
                x1, y1, x2, y2 = map(int, box.xyxy[0])
                cv2.rectangle(frame, (x1, y1), (x2, y2), (0, 255, 0), 2)

    if vehiculos_detectados > 0:
        arduino.write(b"G_OPEN\n")
    else:
        arduino.write(b"G_CLOSE\n")

    cv2.putText(frame, f"Vehiculos: {vehiculos_detectados}", (10, 30),
                cv2.FONT_HERSHEY_SIMPLEX, 1, (0, 255, 0), 2)
    cv2.imshow("Garaje Inteligente", frame)

    if cv2.waitKey(1) & 0xFF == ord('q'):
        break

cap.release()
cv2.destroyAllWindows()
arduino.close()
```

#### Código Arduino — Control del Garaje

```cpp
// Pines
const int PIN_RELE = 7;
const int PIN_LED_VERDE = 11;
const int PIN_LED_ROJO = 12;

String comando = "";

void setup() {
  Serial.begin(9600);
  pinMode(PIN_RELE, OUTPUT);
  pinMode(PIN_LED_VERDE, OUTPUT);
  pinMode(PIN_LED_ROJO, OUTPUT);
  digitalWrite(PIN_LED_ROJO, HIGH); // Inicio: cerrado
}

void loop() {
  if (Serial.available() > 0) {
    comando = Serial.readStringUntil('\n');
    comando.trim();

    if (comando == "G_OPEN") {
      digitalWrite(PIN_RELE, HIGH);
      digitalWrite(PIN_LED_VERDE, HIGH);
      digitalWrite(PIN_LED_ROJO, LOW);
      delay(5000); // Motor gira 5 segundos
      digitalWrite(PIN_RELE, LOW);
      Serial.println("ABIERTO");
    }

    else if (comando == "G_CLOSE") {
      digitalWrite(PIN_RELE, HIGH);
      digitalWrite(PIN_LED_ROJO, HIGH);
      digitalWrite(PIN_LED_VERDE, LOW);
      delay(5000);
      digitalWrite(PIN_RELE, LOW);
      Serial.println("CERRADO");
    }

    else if (comando == "ESTADO") {
      // Responde estado actual según último LEDs
      if (digitalRead(PIN_LED_VERDE)) Serial.println("ABIERTO");
      else Serial.println("CERRADO");
    }
  }
}
```
<img width="1600" height="900" alt="image" src="https://github.com/user-attachments/assets/5d76d023-4311-450b-af01-a8b889dcd5c5" />

---

### 🍳 Cocina Inteligente

**Función:** Controla por voz la iluminación, el extractor de aire, la estufa y la nevera de la cocina.

#### Componentes

| Elemento | Actuador | Comando de Voz |
|---|---|---|
| Estufa | LED (Pin) | "encender cocina" / "apagar cocina" |
| Nevera | LED (Pin) | "encender nevera" / "apagar nevera" |
| Extractor | Motor DC pequeño | "encender extractor" / "apagar extractor" |
| Iluminación | LED blanco | "encender luces cocina" / "apagar luces cocina" |

#### Tramas Seriales — Cocina

```
COMANDO VOZ             →  TRAMA SERIAL
─────────────────────────────────────────
"encender nevera"       →  N_ON\n
"apagar nevera"         →  N_OFF\n
"encender cocina"       →  E_ON\n
"apagar cocina"         →  E_OFF\n
"encender extractor"    →  EXT_ON\n
"apagar extractor"      →  EXT_OFF\n
"encender luces cocina" →  L_ON\n
"apagar luces cocina"   →  L_OFF\n
```
<img width="900" height="1600" alt="image" src="https://github.com/user-attachments/assets/193de556-836d-4ce0-b85d-a74726dde3f8" />

---

### Habitación Inteligente

**Función:** Gestiona la iluminación, la persiana motorizada y muestra un reloj en pantalla OLED 128x128. Control mediante asistente de voz.

#### Componentes

| Elemento | Hardware | Descripción |
|---|---|---|
| Iluminación | LED blanco | Luz principal de la habitación |
| Persiana | Servo motor | Sube/baja automáticamente |
| Reloj | OLED 1.5" 128x128 | Muestra hora en tiempo real |
| Asistente de Voz | Micrófono + Python | Control hands-free |

<img width="900" height="1600" alt="image" src="https://github.com/user-attachments/assets/cd4aac41-b6ac-4de4-b253-ced1914b491c" />

---

### Baño Inteligente

**Función:** Cambia la iluminación RGB del baño según el modo (Mañana / Spa / Noche) mediante comandos de voz. Muestra el modo activo en una pantalla OLED pequeña.

#### Modos de Iluminación

| Modo | Color LED RGB | Comando de Voz |
|---|---|---|
| Modo Mañana | Azul | "modo mañana" |
| Modo Spa | Rojo | "modo spa" |
| Modo Noche | Verde | "modo noche" |

#### Conexión del LED RGB

```
Arduino Pin R  →  LED RGB (Pin R)
Arduino Pin G  →  LED RGB (Pin G)
Arduino Pin B  →  LED RGB (Pin B)
GND            →  Cátodo común (-)
```
<img width="900" height="1600" alt="image" src="https://github.com/user-attachments/assets/981b8969-59dd-4ddb-b4e7-640d04bb26e5" />

---

### Sala Inteligente

**Función:** Módulo de entretenimiento con piano digital, televisor de juegos (pantalla LCD 16x2), cuadro de arte dinámico (OLED) y comedor. Control por voz.

#### Elementos

| Elemento | Hardware | Función |
|---|---|---|
| Piano | Piezobuzzer + botones | Toca notas musicales |
| TV de Juegos | LCD 16x2 | Ejecuta 2 minijuegos |
| Cuadro Arte | OLED | Muestra arte generativo dinámico |
| Comedor | LEDs | Iluminación ambiente |
| Asistente Voz | Micrófono + Python | Control de todos los elementos |

<img width="1600" height="900" alt="image" src="https://github.com/user-attachments/assets/dcd7c52a-e0d9-4941-93b2-d531b5dda1f7" />

---

### Nube y Dashboard

**Función:** Centraliza todos los datos de los módulos, los procesa con un flujo ETL y los visualiza en un dashboard en tiempo real.

#### Flujo ETL

```
Base de Datos
    │
    ├── E (Extracción)   → Lee estados de cada módulo vía Serial/MQTT
    ├── T (Transformación) → Normaliza y codifica los datos
    └── L (Carga)         → Publica al dashboard en la nube
```

#### Dashboard
- Muestra el estado en tiempo real de cada módulo (Garaje, Cocina, Habitación, Baño, Sala)
- Gráficas históricas de actividad
- Repositorio público en GitHub para acceso a todo el código

---

## Control por Voz

El módulo de voz es el componente central que conecta todos los módulos. Escucha comandos en **español** mediante el micrófono y los transmite al Arduino vía USB Serial.

### Módulo `voz_control.py` — Código Completo

```python
"""
Módulo de Control por Voz - Sistema Domótico
Escucha comandos de voz a través del micrófono y los transmite
vía puerto Serial a un microcontrolador.
"""
import sys
import time
import serial
import speech_recognition as sr

# --- Configuración del Sistema ---
PUERTO_SERIAL = 'COM4'
BAUD_RATE = 9600
TIEMPO_ESPERA = 1

def inicializar_conexion(puerto: str, baud_rate: int) -> serial.Serial:
    """Inicializa y retorna la conexión serial de forma segura."""
    try:
        conexion = serial.Serial(puerto, baud_rate, timeout=TIEMPO_ESPERA)
        time.sleep(2)  # Tiempo de estabilización del bootloader del Arduino
        print(f"Conexión establecida en {puerto}.")
        return conexion
    except serial.SerialException:
        print(f"Error: No se pudo abrir el puerto {puerto}. Verifique la conexión física.")
        sys.exit(1)

def procesar_comando(texto: str, conexion: serial.Serial) -> None:
    """Mapea el texto reconocido a tramas seriales y las transmite."""
    comandos = {
        "abrir parqueadero":    b"G_OPEN\n",
        "cerrar parqueadero":   b"G_CLOSE\n",
        "encender nevera":      b"N_ON\n",
        "apagar nevera":        b"N_OFF\n",
        "encender cocina":      b"E_ON\n",
        "apagar cocina":        b"E_OFF\n",
        "encender extractor":   b"EXT_ON\n",
        "apagar extractor":     b"EXT_OFF\n",
        "encender luces cocina": b"L_ON\n",
        "apagar luces cocina":  b"L_OFF\n",
    }

    comando_ejecutado = False
    for frase, trama in comandos.items():
        if frase in texto:
            conexion.write(trama)
            print(f"Enviando orden asociada a: '{frase.upper()}'")
            comando_ejecutado = True
            break

    if not comando_ejecutado:
        print("Comando no registrado en la base de datos operativa.")

def escuchar_comandos(reconocedor: sr.Recognizer, conexion: serial.Serial) -> None:
    """Captura audio del micrófono y lo procesa mediante la API de Google."""
    with sr.Microphone() as source:
        print("\n--- ESPERANDO ORDEN ---")
        reconocedor.adjust_for_ambient_noise(source, duration=0.5)

        try:
            audio = reconocedor.listen(source)
            texto = reconocedor.recognize_google(audio, language="es-ES").lower()
            print(f"Comandante dice: '{texto}'")
            procesar_comando(texto, conexion)

        except sr.UnknownValueError:
            print("El sistema no pudo entender el audio.")
        except sr.RequestError:
            print("Error de conexión: Fallo en la API de reconocimiento de voz.")

def main():
    arduino = inicializar_conexion(PUERTO_SERIAL, BAUD_RATE)
    reconocedor = sr.Recognizer()

    try:
        while True:
            escuchar_comandos(reconocedor, arduino)
    except KeyboardInterrupt:
        print("\nSistema detenido por el usuario.")
    finally:
        if 'arduino' in locals() and arduino.is_open:
            arduino.close()
            print("Puerto serial cerrado de forma segura.")

if __name__ == "__main__":
    main()
```

### ¿Cómo funciona el reconocimiento de voz?

```
Micrófono (audio)
       │
       ▼
sr.Recognizer.listen()        ← captura el audio
       │
       ▼
recognize_google(audio, language="es-ES")  ← API Google convierte a texto
       │
       ▼
procesar_comando(texto)       ← busca la frase en el diccionario
       │
       ▼
conexion.write(trama)         ← envía bytes al Arduino por USB
       │
       ▼
Arduino ejecuta acción        ← mueve motor, enciende LED, etc.
```

---

## Protocolo de Comunicación Serial

Toda la comunicación entre Python y Arduino usa **USB Serial a 9600 baud**. Los comandos son cadenas de texto terminadas en `\n`.

| Módulo | Trama | Acción Arduino |
|---|---|---|
| Garaje | `G_OPEN\n` | Motor avanza 5 seg, LED Verde ON |
| Garaje | `G_CLOSE\n` | Motor retrocede 5 seg, LED Rojo ON |
| Garaje | `ESTADO\n` | Responde ABIERTO/CERRADO |
| Cocina | `N_ON\n` / `N_OFF\n` | Enciende/apaga LED de Nevera |
| Cocina | `E_ON\n` / `E_OFF\n` | Enciende/apaga LED de Estufa |
| Cocina | `EXT_ON\n` / `EXT_OFF\n` | Enciende/apaga motor extractor |
| Cocina | `L_ON\n` / `L_OFF\n` | Enciende/apaga luz de cocina |

---

## Hardware y Componentes

### Lista de Materiales (BOM)

| Componente | Cantidad | Especificación | Módulo |
|---|---|---|---|
| Arduino UNO | 1+ | ATmega328P | Todos |
| Relé 5V | 1 | SRD-05VDC | Garaje |
| Motor DC | 1 | 9V, 100 RPM | Garaje |
| Fuente de poder | 1 | 9V, 5A | Garaje |
| Cámara | 1 | Celular (IP Webcam) | Garaje |
| LED Verde | 1 | Pin 11 | Garaje |
| LED Rojo | 1 | Pin 12 | Garaje |
| LED RGB | 1 | Ánodo/Cátodo común | Baño |
| Pantalla OLED pequeña | 1 | I2C | Baño |
| Pantalla OLED 1.5" | 1 | 128x128 px | Habitación |
| Servo motor | 1 | SG90 | Habitación |
| LCD 16x2 | 1 | Con módulo I2C | Sala |
| Piezobuzzer | 1 | Activo/Pasivo | Sala |
| Micrófono USB | 1 | Compatible con Python | Control Voz |

### Diagrama de Conexiones — Garaje

```
GND COMÚN:
Arduino GND ══ Fuente 9V(-) ══ Motor GND

CONTROL:
Arduino Pin 7  → Relé Coil+  → Motor ON/OFF
Fuente 9V(+)   → Relé NO     → Motor(+)

INDICADORES:
Arduino Pin 11 → Resistencia 220Ω → LED Verde(+) → GND
Arduino Pin 12 → Resistencia 220Ω → LED Rojo(+)  → GND
```

---

## Instalación y Dependencias

### Requisitos del sistema
- Python 3.8+
- Arduino IDE
- Cable USB tipo A-B

### Instalar librerías Python

```bash
pip install ultralytics opencv-python pyserial SpeechRecognition
```

> **Nota:** Para `SpeechRecognition` en Windows puede ser necesario instalar también `PyAudio`:
> ```bash
> pip install pyaudio
> ```

### Librerías utilizadas

| Librería | Función |
|---|---|
| `ultralytics` (YOLO) | Detección de vehículos en tiempo real |
| `opencv-python` | Captura y procesamiento de video |
| `pyserial` | Comunicación USB Serial con Arduino |
| `SpeechRecognition` | Reconocimiento de voz en español |
| `time`, `sys` | Utilidades del sistema |

### Instalar librerías Arduino

Desde el **Gestor de Librerías** del Arduino IDE:
- `Wire.h` (incluida por defecto) — para I2C (OLED, LCD)
- `Adafruit SSD1306` — pantallas OLED
- `LiquidCrystal I2C` — LCD 16x2
- `Servo` (incluida por defecto) — servo motor persiana

---

## Ejecución del Sistema

### 1. Cargar firmware al Arduino

```bash
# Abrir Arduino IDE
# Seleccionar: Herramientas → Puerto → COMx (el puerto de tu Arduino)
# Seleccionar: Herramientas → Placa → Arduino UNO
# Cargar el sketch correspondiente a cada módulo
```

### 2. Verificar el puerto serial

En Windows: Administrador de dispositivos → Puertos COM y LPT → Arduino UNO (COMx)

Editar en `voz_control.py`:
```python
PUERTO_SERIAL = 'COM4'  # Cambiar por tu puerto real
```

### 3. Ejecutar el módulo de voz

```bash
python voz_control.py
```

### 4. Ejecutar el módulo del garaje (YOLO)

```bash
python garaje_yolo.py
```

### 5. Comandos de voz disponibles

Habla claramente en español. El sistema reconoce:

```
"abrir parqueadero"       → Abre la puerta del garaje
"cerrar parqueadero"      → Cierra la puerta del garaje
"encender nevera"         → Enciende la nevera (LED)
"apagar nevera"           → Apaga la nevera
"encender cocina"         → Enciende la estufa (LED)
"apagar cocina"           → Apaga la estufa
"encender extractor"      → Prende el ventilador extractor
"apagar extractor"        → Apaga el extractor
"encender luces cocina"   → Enciende iluminación de cocina
"apagar luces cocina"     → Apaga iluminación de cocina
```

---

## Plan de Pruebas

| Prueba | Método | Criterio de Aceptación |
|---|---|---|
| Hardware | Multímetro (continuidad) | GND común verificado, voltajes correctos |
| Motor | Activación manual desde Serial | Gira en ambas direcciones sin ruidos |
| Comunicación Serial | Monitor Serial Arduino IDE | Arduino responde a G_OPEN y G_CLOSE |
| YOLO | Video en tiempo real | Detección de vehículos confiable (>80% conf.) |
| Reconocimiento de voz | Prueba en entorno silencioso | Reconoce ≥8/10 comandos correctamente |
| Integración | Ciclo completo | Apertura/cierre automático sin errores consecutivos |
| Dashboard | Revisión de datos en nube | Todos los módulos reportan estado correctamente |

---

## Estructura del Repositorio

```
casa-domotica/
│
├── README.md                    ← Este archivo
│
├── garaje/
│   ├── garaje_yolo.py           ← Detección YOLO + control serial
│   └── arduino_garaje.ino       ← Firmware Arduino (motor + LEDs)
│
├── cocina/
│   └── arduino_cocina.ino       ← Firmware Arduino (estufa, nevera, extractor)
│
├── habitacion/
│   └── arduino_habitacion.ino   ← Firmware Arduino (persiana, OLED, luz)
│
├── bano/
│   └── arduino_bano.ino         ← Firmware Arduino (RGB, OLED, modos)
│
├── sala/
│   └── arduino_sala.ino         ← Firmware Arduino (piano, LCD, arte)
│
├── voz/
│   └── voz_control.py           ← Módulo central de control por voz
│
└── dashboard/
    └── etl_dashboard.py         ← Extracción, transformación y carga a nube
```

---

## Video Demostrativo



---

## Créditos

**Curso:** Aplicaciones en Sistemas Embebidos  
**Institución:** Fundación Universitaria Compensar  
**Docente:** Diego Alejandro Barragán Vargas — Ingeniero Electrónico, Magíster en Ingeniería (UDFJC)
**Alumnos:** Lina Contreras, Brandon Andres Leon, Samuel Rojas, Luis Naranjo, Carlos Castro y Karen Rivera


