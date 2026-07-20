# ECG Arrhythmia Engine

Motor de detección de arritmias sobre ECG single-lead (DII), de la señal cruda ya adquirida a la clasificación. Pipeline: acondicionamiento → detección de latidos (Pan-Tompkins) → clasificación (CNN 1D) → servicio de inferencia vía API. No incluye la etapa de adquisición (electrodos/hardware) — eso lo resuelve el dispositivo de origen de la señal.

Entrenado sobre [MIT-BIH Arrhythmia Database](https://physionet.org/content/mitdb/1.0.0/), con mapeo a taxonomía clínica [AAMI EC57](https://www.aami.org/) y validación inter-paciente.

## Estado actual

**En desarrollo — fase 1 Completa (dataset descargado, canal MLII obtenido, split de de Chazal sin leakage). Comenzando Fase 2.**

Ver el [plan completo del proyecto](docs/PLAN.md) para el detalle de todas las fases, decisiones de diseño y checklist de avance, y [docs/DECISIONS.md](docs/DECISIONS.md) para la bitácora de los puntos donde el criterio de diseño se revisó en el camino.

## Por qué este proyecto

Me interesa el cruce entre procesamiento de señales biomédicas y machine learning aplicado a cardiología, y quería un proyecto propio que fuera más allá de un notebook de clasificación aislado: armar la cadena completa, desde la señal cruda hasta un servicio que la consuma. Es también una forma de aplicar con rigor metodológico (validación inter-paciente, taxonomía clínica estándar) un problema que en el fondo es simple de plantear pero fácil de resolver mal.

## Estructura del repo

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
