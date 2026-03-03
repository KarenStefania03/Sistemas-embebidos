**Sistema de Detección de Postura con Visión Artificial y Control Embebido**

**Descripción General.**

Este proyecto pone en marcha un sistema de detección de la postura en tiempo real que permite saber si una persona está sentada o de pie, mediante visión por computadora y procesamiento geométrico de puntos del cuerpo.

El sistema incorpora procesamiento en Python y comunicación serial con un Arduino Uno, que opera como módulo de salida física a través de LEDs indicadores.

**Objetivo**

Demostrar la integración entre:

* Visión artificial
* Análisis geométrico
* Comunicación serial
* Sistemas embebidos

**Flujo de funcionamiento.**

Cámara Web
-->
Procesamiento en Python
-->
Detección de pose con MediaPipe
-->
Cálculo del ángulo de la rodilla
-->
Clasificación de postura
-->
Comunicación serial
-->
Control de LEDs con Arduino

**Fundamento Técnico.**

**1. Captura de Video.**

Se utiliza la librería OpenCV: 
```python
<img width="391" height="710" alt="image" src="https://github.com/user-attachments/assets/84981109-a9c1-4205-9c9b-7b83af53f5e7" />

```
Cada frame es procesado individualmente.

**2. Detección de Landmarks Corporales.**
Se utiliza: MediaPipe Pose

Este modelo tiene la capacidad de identificar 33 puntos anatómicos del cuerpo humano.

Para el proyecto, se emplean:
* Cadera
* Rodilla
* Tobillo
  
**3. Cálculo Geométrico del Ángulo**

Se calcula el ángulo en la rodilla mediante producto punto:

$$
\theta = \cos^{-1}\left(\frac{\vec{BA}\cdot\vec{BC}}{|\vec{BA}||\vec{BC}|}\right)
$$

Donde:

A = Cadera

B = Rodilla

C = Tobillo

Se implementa con NumPy.

**4. Criterio de Clasificación**
   
| Ángulo  | Postura |
|----------|----------|
| > 160°   | PARADA   |
| ≤ 160°   | SENTADA  |

Se definió el umbral de maemnra experimental.

**5. Comunicación Serial**

Se establece comunicación con:

Arduino IDE
-->
Arduino Uno

Configuración:
```python
arduino = serial.Serial('COM3', 9600)
```
| Carácter | Significado     |
| -------- | --------------- |
| 'R'      | Persona parada  |
| 'G'      | Persona sentada |
| 'X'      | Apagar LEDs     |

**6. Control Embebido**

Arduino controla:

* LED Rojo --> PARADO
* LED Verde--> SENTADO

**7. estión Segura de Recursos**
Se estructura:
```python
try:
    # código principal
finally:
    arduino.write(b'X')
    cap.release()
    arduino.close()
```

**Requisitos del Sistema**

**💻 Requisitos de Software**

* Python 3.11
* OpenCV
* MediaPipe
* NumPy
* PySerial
* Arduino IDE

Instalación:
```python
pip install opencv-python mediapipe numpy pyserial
```

**🔌Requisitos de Hardware**

* Cámara web
* Arduino Uno
* 2 LEDs
* 2 resistencias (220Ω – 330Ω)
* Protoboard
* Cables jumper

**Resultados Observados**

* Detección estable con buena iluminación
* Baja latencia serial
* Respuesta inmediata del hardware
* Apagado seguro automático

**Conclusión.**

Este proyecto demuestra una integración funcional entre visión por computadora y sistemas embebidos, utilizando un enfoque geométrico eficiente para clasificación de postura.

**Autores**

* Karen Stefania Rivera Carrero
* Lina Marcela Contreras Sanabria
* Carloa Alberto Castro Castillo

