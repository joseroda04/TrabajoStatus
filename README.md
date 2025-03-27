# TrabajoStatus 
Patrón de diseño de desarrollo: Estado aplicado a un proyecto python.

Grupo: 
Jose Antonio Roda Donoso
David Alberto Cruz Barranco
Manuel Blanes Toril

Asignatura: Entornos de Desarrollo

Curso: !º DAM

Año: 2024-2025

Introducción

El Patrón de Diseño Estado es un patrón que permite que un objeto cambie su comportamiento según su estado interno. Es muy útil cuando un objeto tiene varios estados y su comportamiento depende de cuál esté activo.

¿Por qué elegir este patrón y no otro?

Este patrón es ideal cuando tienes un objeto con muchos estados y quieres evitar usar condicionales (como if o switch). Si no usas este patrón, el código puede volverse desordenado y difícil de mantener

Ventajas 

Mejora la legibilidad
El patrón Estado hace que el código sea más limpio al separar el comportamiento de cada estado en clases diferentes, eliminando condicionales complicadas.

Facilita el mantenimiento
Cambiar o agregar nuevos estados es más fácil porque cada estado está en su propia clase. No necesitas tocar el resto del código.

Permite extender el sistema
Es fácil agregar nuevos estados sin escribir mucho código, sólo creando nuevas clases para cada uno.

Elimina condicionales
En lugar de usar if o switch grandes, el comportamiento se maneja por clases separadas, lo que hace el código más simple.

Responsabilidad única
Cada clase de estado tiene una tarea específica, lo que hace el código más claro y organizado.

Perfecto para sistemas con muchos estados
Si el sistema tiene muchos estados, como un reproductor de música con "Pausado", "Reproduciendo", "Detenido", el patrón ayuda a gestionar todo de forma más ordenada.

Mayor flexibilidad
El objeto puede cambiar de comportamiento fácilmente durante la ejecución sin complicaciones, lo que lo hace más adaptable.

Desventajas 
Complejidad adicional 
Se deben crear varias clases para representar cada estado, lo que aumenta la cantidad de código y puede hacer que la estructura sea más difícil de mantener.
Mayor consumo de memoria 
Cada estado es una instancia de clase diferente, lo que puede aumentar el uso de memoria si hay demasiados estados.
Puede ser excesivo para problemas simples
Si el número de estados es bajo o los cambios de estado son mínimos, usar este patrón puede ser innecesario y hacer que el código sea más difícil de leer.
Dificultad en la depuración
Seguir el flujo del programa puede ser más complicado porque los cambios de estado pueden ocurrir en diferentes partes del código y no siempre es obvio qué estado está activo en un momento dado.
Acoplamiento con la clase principal 
La clase que gestiona los estados puede volverse dependiente de todas las implementaciones de estado, lo que reduce la flexibilidad si se quieren hacer cambios o agregar nuevos estados.

Solución
Aplicación del Patrón State en el Proyecto
Problema Identificado
El sistema debe gestionar el estado de un LED y sincronizarlo con una API. Sin embargo, el código original maneja los estados del LED y de la API de manera procedural, sin una estructura clara para gestionar los cambios de estado. Esto causa problemas como:

Código difícil de mantener.

Dependencias directas entre la Raspberry Pi y la API.

Problemas de sincronización con la interfaz web.

Para solucionar esto, aplicamos el Patrón State.

Patrón de Diseño Aplicado: State
El Patrón State permite encapsular los diferentes estados del LED y su comportamiento en clases separadas. Esto ayuda a hacer el código más modular y fácil de escalar.

Implementación del Patrón State
En nuestro caso, los estados posibles del LED son:
Estado Inactivo (LED apagado, API reporta "inactivo").
Estado Activo (LED encendido, API reporta "activo").

En lugar de manejar estos estados con simples variables booleanas, creamos una clase de contexto (LEDController) que delega la lógica a clases de estado específicas (EstadoActivo y EstadoInactivo).

Solución Implementada
En la Raspberry Pi (Código en Python)
Implementamos el patrón State en la gestión del LED:

import sys
import time
import requests

try:
    import RPi.GPIO as GPIO
    SIMULACION = False
except (ImportError, RuntimeError):
    print("[ADVERTENCIA] RPi.GPIO no disponible. Ejecutando en modo simulación.")
    SIMULACION = True

# -------------------------------
# CONFIGURACIÓN
# -------------------------------
LED_PIN = 18        # Pin GPIO para el LED (solo en Raspberry Pi)
BUTTON_PIN = 17     # Pin GPIO para el pulsador (solo en Raspberry Pi)

RASPBERRY_ID = 1    # ID de la Raspberry Pi
POSICION = "1"      # Posición asignada a la Raspberry Pi
API_URL = "http://127.0.0.1:8000/api/alumnos/actualizar_alumno/"  # URL del PUT

# -------------------------------
# CONFIGURACIÓN GPIO (solo en Raspberry Pi)
# -------------------------------
if not SIMULACION:
    GPIO.setmode(GPIO.BCM)
    GPIO.setup(LED_PIN, GPIO.OUT)
    GPIO.setup(BUTTON_PIN, GPIO.IN, pull_up_down=GPIO.PUD_DOWN)

# -------------------------------
# IMPLEMENTACIÓN DEL PATRÓN STATE
# -------------------------------
class EstadoLED:
    """Clase base para los estados del LED."""
    def encender(self, controlador):
        pass

    def apagar(self, controlador):
        pass

    def actualizar_api(self, estado):
        """Envía una actualización a la API con el estado 'activo' o 'inactivo'."""
        payload = {
            "id": RASPBERRY_ID,
            "posicion": POSICION,
            "estatus": estado
        }
        try:
            response = requests.put(API_URL, json=payload, timeout=5)
            if response.status_code == 200:
                print(f"[OK] Estado actualizado vía PUT: {payload}")
            else:
                print(f"[ERROR] Fallo en el PUT: Código {response.status_code}, Respuesta: {response.text}")
        except requests.exceptions.RequestException as e:
            print(f"[ERROR] No se pudo conectar a la API: {e}")

class EstadoActivo(EstadoLED):
    """Estado en el que el LED está encendido."""
    def encender(self, controlador):
        print("El LED ya está encendido.")

    def apagar(self, controlador):
        print("Apagando LED...")
        controlador.estado = EstadoInactivo()
        self.actualizar_api("inactivo")
        if not SIMULACION:
            GPIO.output(LED_PIN, False)

class EstadoInactivo(EstadoLED):
    """Estado en el que el LED está apagado."""
    def encender(self, controlador):
        print("Encendiendo LED...")
        controlador.estado = EstadoActivo()
        self.actualizar_api("activo")
        if not SIMULACION:
            GPIO.output(LED_PIN, True)

    def apagar(self, controlador):
        print("El LED ya está apagado.")

class LEDController:
    """Controlador que maneja los estados del LED."""
    def __init__(self):
        self.estado = EstadoInactivo()  # Estado inicial

    def alternar(self):
        """Alterna entre estados."""
        if isinstance(self.estado, EstadoInactivo):
            self.estado.encender(self)
        else:
            self.estado.apagar(self)

# -------------------------------
# BUCLE PRINCIPAL
# -------------------------------
try:
    print("\n[INFO] Programa iniciado. Presiona Ctrl+C para salir.")
    led_controller = LEDController()

    if SIMULACION:
        print("[INFO] Modo simulación activado. Pulsa 'Enter' para alternar el LED.")
        while True:
            input("\n[SIMULACIÓN] Presiona 'Enter' para simular pulsación de botón...")
            led_controller.alternar()
    else:
        print("[INFO] Monitoreando pulsaciones reales en la Raspberry Pi...\n")
        while True:
            if GPIO.input(BUTTON_PIN) == GPIO.HIGH:
                led_controller.alternar()
                while GPIO.input(BUTTON_PIN) == GPIO.HIGH:
                    time.sleep(0.05)  # Espera hasta que se suelte el botón
                time.sleep(0.2)       # Antirrebote
            time.sleep(0.05)          # Reduce carga en la CPU

except KeyboardInterrupt:
    print("\n[INFO] Programa detenido por el usuario.")

finally:
    if not SIMULACION:
        GPIO.cleanup()
        print("[INFO] GPIO liberado.")
    print("[INFO] Programa finalizado.")

En la API (Django o Flask)
La API sigue funcionando como intermediaria, almacenando los estados.

No necesita cambios en la estructura gracias a la abstracción del patrón State.

En la Interfaz Web (JavaScript)
El frontend simplemente solicita el estado actual cada cierto tiempo y lo refleja en la tabla:

async function obtenerAlumnos() {
    try {
        const response = await fetch('http://127.0.0.1:8000/api/alumnos/');
        const alumnos = await response.json();
        mostrarAlumnos(alumnos);
    } catch (error) {
        console.error('Error al obtener los alumnos:', error);
    }
}

// Llamar cada segundo para actualizar en tiempo real
setInterval(obtenerAlumnos, 1000);
Beneficio: Se mantiene la sincronización sin afectar el diseño modular del backend.

 Resultados Obtenidos
Código más modular y fácil de mantener.
Estados encapsulados en clases separadas.
Facilidad para agregar nuevos estados en el futuro.
Sincronización eficiente entre la Raspberry Pi, la API y la interfaz web.

El Patrón State ha permitido mejorar la estructura del código, facilitando futuras mejoras y evitando código repetitivo. 
