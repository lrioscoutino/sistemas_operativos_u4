# Práctica guiada: Administración de entrada/salida, paso a paso

Tutorial práctico para la Unidad 4 (4.1–4.4 del temario AEC-1061). Vas a construir y correr tres simulaciones que muestran, con números reales, por qué el sistema operativo necesita buffers, por qué el orden en que atiende al disco importa, y por qué casi nadie usa *polling* puro para esperar un dispositivo.

Requisito: Python 3 (no se necesita ninguna librería externa).

---

## Paso 1 — Buffering: el colchón entre un dispositivo rápido y uno lento (4.1–4.3)

### El problema

Un dispositivo de entrada (teclado, red, disco) casi nunca produce datos al mismo ritmo en que el proceso los consume. Si el sistema operativo no tuviera dónde guardar temporalmente esos datos, se perderían cada vez que el productor fuera más rápido que el consumidor. Esa estructura de datos intermedia es el **buffer**, y la variante más común en manejadores de dispositivos es el **buffer circular**.

### Lo que vas a construir

```python
class BufferCircular:
    def __init__(self, capacidad):
        self.capacidad = capacidad
        self.datos = [None] * capacidad
        self.cabeza = 0  # siguiente lectura
        self.cola = 0    # siguiente escritura
        self.cuenta = 0

    def escribir(self, dato):
        if self.cuenta == self.capacidad:
            return False  # buffer lleno, el productor se bloquea/descarta
        self.datos[self.cola] = dato
        self.cola = (self.cola + 1) % self.capacidad
        self.cuenta += 1
        return True

    def leer(self):
        if self.cuenta == 0:
            return None  # buffer vacio, el consumidor se bloquea
        dato = self.datos[self.cabeza]
        self.cabeza = (self.cabeza + 1) % self.capacidad
        self.cuenta -= 1
        return dato

buf = BufferCircular(4)

# Dispositivo rapido (teclado) produce 6 eventos antes de que el proceso lea nada
print("=== Fase de escritura (productor rapido, consumidor aun no lee) ===")
for i in range(6):
    ok = buf.escribir(f"tecla_{i}")
    print(f"escribir tecla_{i}: {'OK' if ok else 'BUFFER LLENO, se pierde el dato'}")

print("\n=== Fase de lectura (consumidor procesa lo que si cupo) ===")
for _ in range(6):
    dato = buf.leer()
    print(f"leer -> {dato if dato else 'BUFFER VACIO'}")
```

### Salida real (verificada)

```
=== Fase de escritura (productor rapido, consumidor aun no lee) ===
escribir tecla_0: OK
escribir tecla_1: OK
escribir tecla_2: OK
escribir tecla_3: OK
escribir tecla_4: BUFFER LLENO, se pierde el dato
escribir tecla_5: BUFFER LLENO, se pierde el dato

=== Fase de lectura (consumidor procesa lo que si cupo) ===
leer -> tecla_0
leer -> tecla_1
leer -> tecla_2
leer -> tecla_3
leer -> BUFFER VACIO
leer -> BUFFER VACIO
```

### Qué acaba de pasar

El buffer tiene capacidad para 4 elementos. El "dispositivo" (en este caso, alguien tecleando muy rápido) metió 6 eventos antes de que nadie leyera nada — las primeras 4 teclas cupieron, pero `tecla_4` y `tecla_5` **se perdieron de verdad**: la función `escribir` devolvió `False` y esos datos nunca entraron a la estructura. Esto es exactamente lo que le pasa a un teclado real cuando el buffer del sistema operativo está lleno y nadie ha leído la entrada estándar: los eventos se descartan o el dispositivo se bloquea, dependiendo de cómo esté configurado el manejador.

En la fase de lectura, el consumidor sí logra leer las 4 que alcanzaron a entrar (`tecla_0` a `tecla_3`), y luego el buffer está vacío — por eso las últimas dos lecturas devuelven `BUFFER VACIO`, no un error: es el estado normal de "aquí no hay nada que leer todavía".

### Ejercicio para ti

Sube `capacidad` de 4 a 8 y vuelve a correr. ¿Se siguen perdiendo datos? Ahora baja `capacidad` a 2 — ¿en qué punto exacto empieza a perder datos? Relaciona esto con una pregunta real: ¿por qué un sistema operativo no simplemente usa un buffer "infinito" para nunca perder nada?

---

## Paso 2 — Planificación del disco: en qué orden atender las solicitudes (4.2–4.4)

### El problema

El disco (o cualquier dispositivo de acceso secuencial con una cabeza física que se mueve) recibe varias solicitudes de lectura/escritura en distintas posiciones casi al mismo tiempo. Moverse al azar o en el orden en que llegaron puede significar que el cabezal viaje de un lado al otro del disco innecesariamente. El sistema operativo decide el **orden de atención** con un algoritmo de planificación de disco.

### Lo que vas a construir

Tres algoritmos clásicos, sobre las mismas solicitudes y la misma posición inicial del cabezal:

- **FCFS** (*First-Come, First-Served*): atiende en el orden en que llegaron las solicitudes, sin optimizar nada.
- **SSTF** (*Shortest Seek Time First*): siempre atiende la solicitud más cercana a la posición actual del cabezal.
- **SCAN** (el "algoritmo del elevador"): el cabezal se mueve en una dirección atendiendo todo lo que encuentra a su paso, hasta el extremo del disco, y luego regresa.

```python
def fcfs(solicitudes, cabeza_inicial):
    orden = solicitudes[:]
    movimiento = 0
    pos = cabeza_inicial
    for s in orden:
        movimiento += abs(s - pos)
        pos = s
    return orden, movimiento

def sstf(solicitudes, cabeza_inicial):
    pendientes = solicitudes[:]
    orden = []
    movimiento = 0
    pos = cabeza_inicial
    while pendientes:
        mas_cercano = min(pendientes, key=lambda s: abs(s - pos))
        movimiento += abs(mas_cercano - pos)
        pos = mas_cercano
        orden.append(mas_cercano)
        pendientes.remove(mas_cercano)
    return orden, movimiento

def scan(solicitudes, cabeza_inicial, limite_max, direccion="derecha"):
    pendientes = sorted(solicitudes)
    orden = []
    movimiento = 0
    pos = cabeza_inicial

    mayores = [s for s in pendientes if s >= cabeza_inicial]
    menores = [s for s in pendientes if s < cabeza_inicial]

    if direccion == "derecha":
        secuencia = mayores + [limite_max] + list(reversed(menores))
    else:
        secuencia = list(reversed(menores)) + [0] + mayores

    for s in secuencia:
        movimiento += abs(s - pos)
        pos = s
        if s not in (0, limite_max):
            orden.append(s)
    return orden, movimiento

solicitudes = [98, 183, 37, 122, 14, 124, 65, 67]
cabeza_inicial = 53
limite_max = 199  # cilindro mas alto del disco (0 a 199)

print(f"Solicitudes: {solicitudes}")
print(f"Cabeza inicial: {cabeza_inicial}, disco de 0 a {limite_max}\n")

for nombre, resultado in [
    ("FCFS", fcfs(solicitudes, cabeza_inicial)),
    ("SSTF", sstf(solicitudes, cabeza_inicial)),
    ("SCAN (hacia la derecha)", scan(solicitudes, cabeza_inicial, limite_max, "derecha")),
]:
    orden, movimiento = resultado
    print(f"=== {nombre} ===")
    print(f"Orden de atencion: {orden}")
    print(f"Movimiento total del cabezal: {movimiento} cilindros\n")
```

### Salida real (verificada)

```
Solicitudes: [98, 183, 37, 122, 14, 124, 65, 67]
Cabeza inicial: 53, disco de 0 a 199

=== FCFS ===
Orden de atencion: [98, 183, 37, 122, 14, 124, 65, 67]
Movimiento total del cabezal: 640 cilindros

=== SSTF ===
Orden de atencion: [65, 67, 37, 14, 98, 122, 124, 183]
Movimiento total del cabezal: 236 cilindros

=== SCAN (hacia la derecha) ===
Orden de atencion: [65, 67, 98, 122, 124, 183, 37, 14]
Movimiento total del cabezal: 331 cilindros
```

### Qué acaba de pasar — la parte más importante de este paso

Las tres estrategias atienden **exactamente las mismas 8 solicitudes**, partiendo del mismo lugar (cilindro 53), y el cabezal termina moviéndose una cantidad muy distinta de cilindros:

| Algoritmo | Movimiento total | Qué significa en la vida real |
|---|---|---|
| FCFS | 640 | El cabezal brinca de 53→98→183→37→122→14... — va y viene por todo el disco sin ningún criterio. Es el "costo de no planificar nada". |
| SSTF | 236 | Mucho mejor: siempre va a lo más cercano. Pero tiene un problema que no se ve en este ejemplo — si llegan solicitudes constantemente cerca de donde ya está el cabezal, las solicitudes lejanas pueden **morir de inanición** (nunca les toca). |
| SCAN | 331 | Peor que SSTF en este caso puntual, pero **garantiza** que nadie se queda esperando para siempre — el cabezal siempre termina pasando por todas las posiciones en su camino de ida y vuelta. Es el balance que usan los sistemas reales: un poco menos óptimo que SSTF, pero justo. |

Esto es el mismo tipo de decisión que ya viste con los algoritmos de reemplazo de páginas en la Unidad 3: no existe un algoritmo que gane siempre en todo — hay que elegir entre "más rápido en promedio" (SSTF) y "más justo, sin inanición" (SCAN).

### Ejercicio para ti

Corre `scan(solicitudes, cabeza_inicial, limite_max, "izquierda")` y compara el movimiento total contra la versión "derecha". ¿Por qué cambia? Luego agrega una solicitud en el cilindro `5` a la lista original y vuelve a correr SSTF — ¿qué tan tarde la atiende comparado con SCAN?

---

## Paso 3 — Cómo espera el CPU a que el dispositivo esté listo: *polling* vs. interrupciones (4.2–4.4)

### El problema

Cuando un proceso pide leer un dispositivo, alguien tiene que esperar a que el dato esté listo. Hay dos formas fundamentalmente distintas de esperar: que el CPU **pregunte una y otra vez** ("¿ya? ¿ya? ¿ya?" — *polling*), o que el dispositivo **avise** cuando termine mediante una interrupción, dejando al CPU libre mientras tanto para hacer otra cosa.

### Lo que vas a construir

```python
import time

class DispositivoLento:
    """Simula un dispositivo que tarda un tiempo fijo en tener el dato listo."""
    def __init__(self, tiempo_listo):
        self.listo_en = time.monotonic() + tiempo_listo
        self.consultas = 0

    def esta_listo(self):
        self.consultas += 1
        return time.monotonic() >= self.listo_en

    def leer(self):
        return "dato_del_dispositivo"


def con_polling(tiempo_listo):
    dispositivo = DispositivoLento(tiempo_listo)
    inicio = time.monotonic()
    while not dispositivo.esta_listo():
        pass  # el CPU se queda aqui "preguntando" sin hacer nada util
    dato = dispositivo.leer()
    duracion = time.monotonic() - inicio
    return dato, duracion, dispositivo.consultas


def con_interrupcion(tiempo_listo):
    # En un SO real el CPU queda libre para otros procesos y el dispositivo
    # "avisa" con una interrupcion cuando termina. Aqui lo simulamos con
    # un solo sleep (cero consultas de CPU) para contrastar contra polling.
    inicio = time.monotonic()
    time.sleep(tiempo_listo)  # el CPU no hace nada por el dispositivo: puede hacer otro trabajo
    dato = "dato_del_dispositivo"
    duracion = time.monotonic() - inicio
    return dato, duracion, 0


TIEMPO_LISTO = 0.05  # 50 ms, valor didactico

dato, dur, consultas = con_polling(TIEMPO_LISTO)
print(f"Polling:      dato={dato!r}  duracion={dur:.4f}s  consultas_de_CPU={consultas}")

dato, dur, consultas = con_interrupcion(TIEMPO_LISTO)
print(f"Interrupcion: dato={dato!r}  duracion={dur:.4f}s  consultas_de_CPU={consultas}")
```

### Salida real (verificada)

```
Polling:      dato='dato_del_dispositivo'  duracion=0.0500s  consultas_de_CPU=442778
Interrupcion: dato='dato_del_dispositivo'  duracion=0.0501s  consultas_de_CPU=0
```

### Qué acaba de pasar — el número que importa

Ambas formas de esperar tardaron prácticamente lo mismo en tiempo real (0.05 segundos, porque el dispositivo simulado tarda lo mismo en estar listo sin importar cómo lo esperes). La diferencia brutal está en la columna `consultas_de_CPU`:

- **Polling:** el CPU preguntó **442,778 veces** "¿ya terminaste?" en esos 50 milisegundos. Cada una de esas preguntas es trabajo del procesador que no sirvió para nada más — durante ese tiempo, ningún otro proceso pudo usar el CPU.
- **Interrupción:** el CPU hizo **0 consultas**. Mientras el dispositivo "trabajaba", el procesador quedó completamente libre para ejecutar cualquier otro proceso, y solo fue interrumpido cuando el dato ya estaba listo.

Este número (442,778 consultas desperdiciadas) es la razón concreta, medible, de por qué casi ningún sistema operativo moderno usa *polling* puro para dispositivos lentos (disco, red, teclado) — se reserva para casos donde la espera es extremadamente corta y el costo de manejar una interrupción sería mayor que simplemente preguntar un par de veces (por ejemplo, ciertos buses de datos de muy alta velocidad).

### Ejercicio para ti

Cambia `TIEMPO_LISTO` a `0.5` (medio segundo) y corre de nuevo. ¿Cuántas consultas hace polling ahora? Con ese número en la mano, explica con tus propias palabras por qué un servidor que atiende miles de conexiones de red simultáneas **no podría funcionar** si cada una se esperara con polling.

---

## Resumen de lo que cubriste

| Tema del programa oficial | Dónde lo viste en esta práctica |
|---|---|
| 4.1 Dispositivos y manejadores de dispositivos | Introducción de cada paso — qué hace el manejador frente al dispositivo real |
| 4.2 Mecanismos y funciones de los manejadores | Paso 2 (decide el orden de atención) y Paso 3 (decide cómo esperar) |
| 4.3 Estructuras de datos para manejo de dispositivos | Paso 1 — el buffer circular |
| 4.4 Operaciones de entrada/salida | Paso 2 (planificación de disco) y Paso 3 (polling vs. interrupción) |
