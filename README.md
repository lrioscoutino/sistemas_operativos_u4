# Unidad 4 — Administración de entrada/salida

**Materia:** Sistemas Operativos (AEC-1061)
**Grupo:** S5C — Aula E10
**Horario:** Martes 18:00–20:00 · Jueves 17:00–19:00
**Enfoque:** teórico-conceptual, con simulaciones ejecutables en Python

Continuación directa de la [Unidad 3](../sistemas_operativos_u3/README.md): ahí viste cómo el núcleo administra la memoria. Esta unidad entra a otro recurso crítico: los dispositivos de entrada/salida — cómo el sistema operativo amortigua la diferencia de velocidad entre CPU y dispositivos (buffers), en qué orden atiende las solicitudes a un disco, y por qué espera eventos con interrupciones en vez de preguntar todo el tiempo (polling).

## Contenido

- [02_practica_entrada_salida.md](02_practica_entrada_salida.md) — Práctica guiada paso a paso: 4.1-4.3 buffering con buffer circular (pérdida de datos cuando se llena), 4.2 y 4.4 planificación de disco (FCFS/SSTF/SCAN comparados), 4.2 y 4.4 polling vs. interrupciones (consultas de CPU desperdiciadas medidas en tiempo real)

## Nota sobre el código

Las tres simulaciones de la práctica son Python **ejecutado y verificado** antes de publicarse — las salidas que aparecen en el documento (incluido el conteo de 442,778 consultas de CPU en el ejemplo de polling) son exactamente las que produce el código al correrlo, no transcripciones de un libro.

## Fuentes

- Programa sinóptico oficial AEC-1061 — TecNM
- Silberschatz, A. — *Fundamentos de Sistemas Operativos* (9na ed.) — ejemplo clásico de planificación de disco (FCFS/SSTF/SCAN)
- Tanenbaum, A. — *Sistemas Operativos Modernos* (3ra ed.) — manejadores de dispositivos, E/S por interrupción vs. consulta
- Stallings, W. — *Sistemas Operativos* (5ta ed.)
