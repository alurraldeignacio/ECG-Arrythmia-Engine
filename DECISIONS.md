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

### [2] — Selección del canal MLII por nombre

**Contexto:** Fase 1. Necesito filtrar qué registros de la MIT-BIH usar. Sabemos que necesitamos las que tienen la derivación MLII (equivalente funcional a DII, la derivación objetivo del proyecto).

**Primera aproximación:** Filtrar por posición fija contemplando que la mayoría de las veces canal[0] contiene MLII.

**Qué aprendí:** Existen tres de los 48 registros que no tienen MLII en el primer canal. En particular, de estos 3, el 114 es el que nos preocupa debido a que forma parte del split de de Chazal et al. Para utilizar el split de de Chazal et al de forma metodologicamente correcta para comparaciones con otros trabajos, necesitamos incorporar todos sus registros, incluido el 114. Necesitamos entonces una forma de incorporar todos los registros de de Chazal et al a nuestros sets de validacion y entrenamiento independientemente de donde almacenan MLII.


**Decisión final:**
Buscar canales por nombre y no por ubicación (registro.sig_name.index('MLII')), contrario a asumir una posición fija para MLII. El dataset de registros exluye 102 y 104. De todos modos esos registros ya no formaban parte del split de de Chazal. Incluímos en nuestro registro, con este cambio de búsqueda, el registro 114, que sí estaba contenido en el split de de Chazal. De este modo, el split de la bibliografía puede implementarse sin modificaciones. 
