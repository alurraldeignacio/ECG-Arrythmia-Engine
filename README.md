# ECG Arrhythmia Engine

Motor de detección de arritmias sobre ECG single-lead (DII), de la señal cruda ya adquirida a la clasificación. 

Pipeline: acondicionamiento → detección de latidos con Pan-Tomkins → Clasificación mediante CNN 1-D → servicio de inferencia por API. No incluye la etapa de adquisición (electrodos/hardware) — eso lo resuelve el dispositivo de origen de la señal.

Sistema inicialmente pensado como complemento de ergómetro portátil desarrollado en otro proyecto.

Entrenado sobre [MIT-BIH Arrhythmia Database](https://physionet.org/content/mitdb/1.0.0/), con mapeo a taxonomía clínica [AAMI EC57](https://www.aami.org/). 

## Estado actual

**En desarrollo — fase de diseño y setup inicial.**

Ver el [plan completo del proyecto](docs/PLAN.md) para el detalle de todas las fases, decisiones de diseño y el checklist de avances.

## Por qué este proyecto

Dos razones. Punto 1, quiero complementar mi proyecto universitario sobre el diseño y desarrollo de un sistema de ergometría portátil con un clasificador srevido via API del que este ergómetro pueda consumir. Punto 2, Me interesa el cruce entre procesamiento de señales biomédicas y machine learning. Ya encaré proyectos de esta índole en mi tesis universitaria, orientado a la detección de Intención de movimiento de mano en pacientes con Hemiparesia y neuropatías utilizando técnicas de aprendizaje automático sobre señales de EMG. Pero en este escenario, quería encarar un proyecto propio que fuera más allá de un notebook de clasificación aislado. El objetivo es armar la cadena completa, desde la señal cruda hasta un servicio que la consuma. Es también una forma de aplicar con rigor metodológico, incluyendo validación inter paciente y taxonomía clínica estándar, un problema que en el fondo es simple de plantear pero fácil de resolver mal. El desafío fundamental que más me entusiasma radica en la investigación de fondo sobre como se resolvieron problemas similares en la práctica. Esto me llevó a leer trabajos fundacionales sobre el tema (ej. Automatic Classification of Heartbeats Using ECG Morphology and Heartbeat Interval Features, De Chazal et. al.). 

## Estructura del repo propuesto

```
ecg-arrhythmia-engine/
├── docs/
│   └── PLAN.md          # Plan completo, fase por fase, con checklist de avance
├── training/             # Notebooks y scripts de preprocesamiento + entrenamiento (Colab)
└── api/                  # Backend FastAPI que sirve el modelo entrenado
```

## Stack

**Señal / ML:** Python, NumPy, SciPy, `wfdb`, PyTorch (CNN 1D), scikit-learn
**Backend:** FastAPI, Docker
**Deploy:** Render
**Datos:** MIT-BIH Arrhythmia Database (PhysioNet)

## Limitaciones

MIT-BIH es una base de datos de laboratorio y no representa completamente el ruido de un Holter ambulatorio real. Este es un proyecto de portfolio/prototipo — no un dispositivo médico validado clínicamente, y no reemplaza revisión médica.

## Autor

Ignacio Alurralde — [LinkedIn](https://linkedin.com/in/ignacio-alurralde)
