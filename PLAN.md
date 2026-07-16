# Plan del proyecto — ECG Arrhythmia Engine

Plan completo de trabajo, desde la descarga de MIT-BIH hasta el modelo servido vía API. Este documento es la referencia de diseño; el estado de avance real se trackea en el [README](../README.md) y en los issues/checklist de cada fase.

## Contexto y objetivo

Motor de detección de arritmias sobre ECG single-lead (DII), pensado como pipeline "de señal cruda a clasificación": acondicionamiento → detección de latidos → clasificación de arritmias, sobre una señal ya adquirida (no incluye la etapa de adquisición/hardware en sí — eso lo resuelve el dispositivo de origen, ej. un Holter o un ergómetro portátil). El modelo se entrena sobre MIT-BIH Arrhythmia Database y se sirve como servicio de inferencia vía FastAPI.

Proyecto personal para profundizar en procesamiento de señales biomédicas y machine learning aplicado a cardiología, combinando DSP clásico (filtrado, detección de QRS) con un clasificador entrenado desde cero, y llevándolo hasta un servicio de inferencia real.

## Decisiones de diseño ya tomadas

| Decisión | Elegida | Motivo |
|---|---|---|
| Dataset | MIT-BIH Arrhythmia Database, vía `wfdb` | Estándar de la industria, anotado por cardiólogos; 48 registros de 47 pacientes con diversidad real de tipos de arritmia — necesario para entrenar un clasificador (a diferencia de la Noise Stress Test DB, que en el fondo es solo 2 de estos mismos registros con ruido real agregado, sin la diversidad necesaria para entrenamiento; ver nota en sección de alcance sobre su uso como augmentation) |
| Derivación | MLII (equivalente funcional a DII) | Coincide con el caso de uso single-lead Holter |
| Taxonomía de clasificación | Mapeo AAMI EC57 (5 superclases: N/S/V/F/Q) | Estándar en literatura, evita el ruido de 15+ clases originales |
| Estrategia de split | Inter-paciente (train/val/test por paciente, no por latido) | Evita data leakage; es el error #1 en proyectos con MIT-BIH |
| Detección de picos R | Pan-Tompkins implementado desde cero, validado contra anotaciones reales | Algoritmo estándar y bien documentado en la literatura de detección de QRS; implementarlo desde cero (en vez de usar una librería) da control total sobre el pipeline y entendimiento profundo del método; entrenamiento usa anotaciones ground-truth, detector se valida aparte |
| Arquitectura del modelo | CNN 1D en PyTorch | Adecuada para morfología de latidos de longitud fija; buen balance entre capacidad de captar patrones locales y simplicidad de entrenamiento frente a arquitecturas recurrentes o de atención |
| Entrenamiento | Google Colab | GPU gratuita, iteración rápida |
| Exportación | `state_dict` de PyTorch + metadata de preprocesamiento (+ ONNX opcional) | Portabilidad, reproducibilidad de inferencia |
| Serving | FastAPI | Framework moderno, tipado, con documentación automática (OpenAPI/Swagger) y buen soporte para servir modelos de ML |
| Deploy | Render (free tier, sin tarjeta) | Zero costo, deploy directo desde Git, Docker soportado |
| Contenerización | Docker (imagen slim, PyTorch CPU-only) | Portabilidad, buena práctica de despliegue |

## FASE 0 — Definición de alcance

- [ ] Confirmar tamaño de ventana por latido (muestras antes/después del pico R)
- [ ] Confirmar criterio de split inter-paciente (split estándar de De Chazal DS1/DS2 vs. split propio documentado)
- [ ] Confirmar nivel de granularidad: clasificación por latido individual (beat-by-beat)

## FASE 1 — Obtención y organización del dataset

- [ ] Descargar MIT-BIH completa vía `wfdb`
- [ ] Verificar qué registros tienen canal 1 = MLII, descartar/separar el resto
- [ ] Definir y documentar el split de pacientes (train/val/test)

## FASE 2 — Acondicionamiento de la señal

- [ ] Implementar filtro pasa-altos / remoción de baseline wander (<0.5 Hz)
- [ ] Implementar filtro notch (ruido de línea, ~60Hz en MIT-BIH)
- [ ] Implementar filtro pasa-bajos (ruido EMG, corte ~35-45 Hz)
- [ ] Combinar en pasa-banda con `filtfilt` (fase cero) usando `scipy.signal`
- [ ] Validar visualmente señal cruda vs. filtrada en varios registros
- [ ] Validación cruzada opcional con NeuroKit2
- [ ] Normalización de amplitud (z-score o min-max por ventana)

## FASE 3 — Detección de latidos (Pan-Tompkins)

- [ ] Implementar Pan-Tompkins desde cero (derivada → cuadrado → integración → umbral adaptativo)
- [ ] Validar picos detectados contra anotaciones `.atr` reales
- [ ] Reportar sensibilidad y precisión de detección
- [ ] Documentar como módulo separado del clasificador (aislar error de detección del error de clasificación)

## FASE 4 — Armado del dataset de latidos

- [ ] Segmentar ventanas fijas centradas en cada pico R anotado
- [ ] Mapear símbolos de anotación MIT-BIH → 5 superclases AAMI
- [ ] Analizar y documentar distribución/desbalance de clases
- [ ] Definir estrategia de balance (class weights / oversampling / augmentation)
- [ ] Data augmentation opcional: time shift, escalado de amplitud, y ruido — evaluar usar las grabaciones de ruido real de la MIT-BIH Noise Stress Test DB (`bw`, `ma`, `em`) en vez de ruido gaussiano genérico, ya que son grabaciones reales de los tipos de artefacto más comunes en ECG ambulatorio (ver nota en sección de alcance/ergómetro)
- [ ] Guardar dataset procesado con split ya aplicado (tensores listos para entrenar)

## FASE 5 — Arquitectura del modelo (CNN 1D)

- [ ] Definir arquitectura (bloques Conv1D → BatchNorm → ReLU → Pool, GAP, FC final)
- [ ] Definir loss (cross-entropy con class weights)
- [ ] Definir optimizer + scheduler (Adam + ReduceLROnPlateau)
- [ ] Definir regularización (dropout, weight decay, early stopping por F1 de validación)

## FASE 6 — Entrenamiento en Colab

- [ ] Notebook de preprocesamiento (produce el dataset de Fase 4)
- [ ] Notebook de entrenamiento (consume el dataset, no reprocesa)
- [ ] Checkpointing del mejor modelo según métrica de validación
- [ ] Trackeo de curvas de loss y F1 por época

## FASE 7 — Validación y testeo

- [ ] Reportar F1 por clase y F1 macro
- [ ] Matriz de confusión completa
- [ ] Sensibilidad específica de clase V (ventricular ectópico)
- [ ] Reportar métricas en split inter-paciente
- [ ] Comparar contra split intra-paciente (aleatorio) para evidenciar el efecto de leakage
- [ ] Comparar contra valores de referencia en literatura (si se usó split De Chazal)
- [ ] Análisis de errores (inspección de latidos mal clasificados)

## FASE 8 — Exportación del modelo

- [ ] Exportar `state_dict` + config de arquitectura + parámetros de preprocesamiento
- [ ] Exportación opcional a ONNX
- [ ] Validar el modelo exportado con un script de inferencia simple, fuera de Colab, antes de tocar la API

## FASE 9 — API (FastAPI)

- [ ] Setup del proyecto (routers, schemas Pydantic, carga del modelo en `startup`)
- [ ] Portar pipeline de preprocesamiento (Fases 2-4) a modo inferencia — mismos parámetros exactos que en train
- [ ] `POST /predict/beat` — clasifica un latido ya segmentado
- [ ] `POST /predict/segment` — recibe tira de señal cruda, corre detección + clasificación
- [ ] `GET /health`
- [ ] Validaciones Pydantic (longitud de señal, sample rate, tipos)

## FASE 10 — Deploy

- [ ] Dockerfile (base slim, PyTorch CPU-only)
- [ ] Deploy en Render (free tier)
- [ ] Verificar cold start / comportamiento tras inactividad
- [ ] Frontend mínimo de demo (opcional — Streamlit o React/Canvas)

## FASE 11 — Portfolio

- [ ] README de nivel case-study (decisiones, resultados, limitaciones)
- [ ] Gráficos de presentación (señal filtrada, picos detectados, matriz de confusión, curvas de entrenamiento, latidos coloreados por clase)
- [ ] Limpieza de repo (`requirements.txt`, `.gitignore`, docstrings)
- [ ] Demo en GIF/video (opcional)

## Alcance: post-análisis, no tiempo real

El sistema está diseñado como **análisis post-grabación (batch)**, no como clasificación en streaming durante el esfuerzo. La señal se graba completa (o por segmentos) y se envía a `POST /predict/segment` para su análisis — el mismo modelo de uso que un Holter enviado a laboratorio.

Esto es una decisión de diseño consciente, no una limitación de cómputo: Pan-Tompkins + una CNN 1D de este tamaño son livianos y correrían perfectamente en tiempo real. La razón de ir por batch es de **validez de dominio** (ver abajo), no de performance.

### Sobre la integración con el Ergómetro Portátil (ver proyecto relacionado)

Existe una motivación clínica real para esta integración: la ergometría (prueba de esfuerzo) es una herramienta estándar de cardiología precisamente porque hay arritmias que solo se manifiestan bajo esfuerzo físico y no aparecen en un registro de reposo. Conectar el motor de clasificación a la señal del ergómetro tiene sentido de caso de uso.

Sin embargo, hay un desajuste de dominio a documentar: **MIT-BIH es señal de reposo / Holter ambulatorio**, con un perfil de ruido distinto al de una sesión de esfuerzo físico (artefacto de movimiento, ruido muscular por el esfuerzo, línea de base afectada por respiración acelerada). Un modelo entrenado exclusivamente sobre MIT-BIH puede degradar su desempeño sobre señal de esfuerzo real, y esto no está validado en el alcance actual del proyecto.

**Por lo tanto:**
- La integración ergómetro → motor de arritmias se plantea como análisis post-sesión (se graba la sesión de esfuerzo, se analiza después), no como alerta en tiempo real durante el ejercicio.
- Cualquier claim de desempeño del modelo aplica al dominio de entrenamiento (reposo/ambulatorio), no a señal de esfuerzo, hasta validar lo contrario.
- Extensión futura posible: validar robustez del modelo contra datasets de señal ruidosa (ej. [MIT-BIH Noise Stress Test Database](https://physionet.org/content/nstdb/1.0.0/)) antes de considerar cualquier escenario de tiempo real. Nótese que la NSTDB, en rigor, es en sí misma una forma de data augmentation: toma solo 2 registros limpios de la Arrhythmia Database (118 y 119) y les agrega ruido **real grabado** (no sintético) de tres tipos — deriva de línea base, ruido muscular y artefacto de movimiento de electrodo, obtenidos de voluntarios activos — en distintos niveles de SNR. Por eso no reemplaza a la Arrhythmia Database como fuente de entrenamiento (carece de diversidad de pacientes y arritmias, al basarse en solo 2 registros), pero sus grabaciones de ruido (`bw`, `ma`, `em`) sí son un insumo mejor que ruido gaussiano genérico para la estrategia de augmentation de la Fase 4 — ver nota ahí.

## Limitaciones a declarar explícitamente

- MIT-BIH es una base de datos de laboratorio; no representa completamente el ruido de un Holter ambulatorio real, y menos aún el de una sesión de esfuerzo físico.
- El modelo no está validado sobre señal de ejercicio/esfuerzo — ver sección de alcance arriba.
- Prototipo de clasificación, no dispositivo médico validado clínicamente.
- No reemplaza revisión médica.
