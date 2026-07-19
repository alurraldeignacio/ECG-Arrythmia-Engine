# Decisiones de diseño — bitácora de aprendizaje

Este documento registra los puntos donde mi entendimiento del problema cambió de forma real durante el desarrollo — no cada elección menor, sino los momentos donde la primera aproximación resultó insuficiente y la revisé con literatura o análisis más profundo. Se escribe en el momento, no reconstruido después.

---

### [1] — Definición de la ventana de segmentación por latido

**Contexto:** Para clasificar cada latido con la CNN 1D, hace falta extraer una ventana de señal fija centrada en cada pico R (Fase 4 del plan).

**Primera aproximación:** Ventana fija simétrica-asimétrica arbitraria, por ejemplo 200ms antes del pico R y 400ms después, aplicada igual a todos los latidos del dataset.

**Qué aprendí:** Un RR interval normal ronda 600-1000ms, pero en presencia de arritmia (extrasístoles muy pegadas, bigeminismo, corridas de taquicardia) puede caer a 300-400ms. Con una ventana fija de ~600ms totales, esos casos generan solapamiento con el latido vecino — exactamente los casos donde la clasificación es clínicamente más relevante y donde más urge que la segmentación sea correcta.

Revisando la literatura (de Chazal et al. 2004; Kachuee et al. 2018) encontré tres enfoques usados en la práctica:
- **Ventana fija simple**, aceptando el solapamiento ocasional — común en papers que priorizan simplicidad y comparabilidad.
- **Ventana adaptativa según el RR real** del segmento (ej. Kachuee et al. definen el largo como una fracción del RR mediano y después normalizan a longitud fija).
- **Estrategia "R-R-R"**: la ventana se define entre el pico R anterior y el posterior al latido actual — por construcción nunca hay solapamiento, con el costo de tener que padear/truncar después para una longitud fija de entrada a la CNN.

También aprendí que de Chazal et al. no clasifican solo por morfología de la ventana: combinan la forma de onda con **features explícitos de intervalo** (RR previo, RR posterior, RR promedio local) como entrada adicional al clasificador — porque hay clases (particularmente V, por la pausa compensatoria típica tras un latido ectópico ventricular) que se distinguen tanto por timing como por forma.

**Decisión final:**
- Ventana con **clip adaptativo**: se define un tamaño objetivo fijo (a determinar en la Fase 0/4, orientativamente ~250ms antes / ~400ms después), pero si el RR hacia el latido vecino es menor que eso, la ventana se recorta hasta la mitad de la distancia al pico vecino más cercano. Evita el solapamiento sin la complejidad de resamplear ventanas de largo variable.
- El modelo va a recibir, además de la ventana de señal, **features explícitos de intervalo RR** (RR previo, RR posterior, RR promedio local) como input adicional — no solo la forma cruda del latido.
- Limitación a documentar: este diseño (clasificación AAMI a nivel de latido individual, con features de intervalo local) está pensado para arritmias que se manifiestan como forma de latido anómala y/o timing puntual (ectopías ventriculares/auriculares). No está diseñado para detectar arritmias que son patrones de ritmo a través de muchos latidos (ej. fibrilación auricular, donde el QRS individual suele ser morfológicamente normal y lo anómalo es la irregularidad del ritmo a lo largo de una secuencia larga) — eso requeriría un enfoque de clasificación de secuencia/ritmo sobre ventanas más largas, fuera del alcance actual.
