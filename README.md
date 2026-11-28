# GA-TL-02 · Taller de diseño de la arquitectura de microservicios del PFC

![Asignatura](https://img.shields.io/badge/Asignatura-Aplicaciones%20Distribuidas%20(ISR--701)-1f6feb)
![Actividad](https://img.shields.io/badge/Actividad-GA--TL--02%20%C2%B7%20FORM--TL--02-2da44e)
![LaTeX](https://img.shields.io/badge/LaTeX-pdflatex-008080)
![Sistema](https://img.shields.io/badge/Sistema-RTV-orange)

Universidad Técnica Estatal de Quevedo (UTEQ), Facultad de Ciencias de la Computación, Carrera de Software. Séptimo nivel, paralelo A. Grupo GLMR.

Taller grupal de diseño de la arquitectura de microservicios del Proyecto Fin de Curso: servicios, propiedad de datos, API Gateway, comunicaciones, consistencia y resiliencia, con diagramas C4 de contenedores y componentes.

## Integrantes

| Integrante |
|---|
| Lombeida Escaleras Bryan Humberto |
| García Bazurto Kevin Leonel |
| Manotoa Alava Carlos Alexander |
| Reinoso Vélez Eduardo David |

## Sistema analizado

Sistema de Revisión Técnica Vehicular (RTV) del Centro de Revisión Técnica Vehicular del Municipio de Valencia, tomado de la entrega E1 de [NixonPiguave/TA-PFC-E1-PFC-Entrega-1](https://github.com/NixonPiguave/TA-PFC-E1-PFC-Entrega-1).

La autorización de inicio de inspección entre Inspecciones y Turnos y Pagos es el único flujo implementado y probado en E1. El equipo confirmó para el taller la descomposición en cuatro servicios y sus bases privadas; el API Gateway, los servicios MS-03 y MS-04 y los demás canales son diseño propuesto, sin implementación, y así se marcan en cada documento. Los requisitos RF y RNF y el contrato de E1 están en [`PFC-RTV-referencia-completa-para-GLMR.md`](PFC-RTV-referencia-completa-para-GLMR.md).

## Estructura

```
.
├── README.md
├── PFC-RTV-referencia-completa-para-GLMR.md   requisitos y contrato de E1 que citan los documentos
├── docs/
│   ├── arquitectura/    01 dominio y servicios, 02 propiedad de datos, 03 API Gateway,
│   │                    04 comunicaciones y contratos, 05 consistencia y resiliencia
│   ├── diagramas/       C4 nivel 2 y nivel 3 (.drawio editable y .pdf)
│   ├── ga-tl-02/        informe en LaTeX (informe.tex) y su PDF compilado (informe.pdf)
│   └── adr/             registros de decisiones de arquitectura
└── evidencias/
    └── ga-tl-02/        bosquejos iniciales dibujados a mano
```

## Compilación

- **Compilador:** `pdflatex`, con TeX Live o MiKTeX.
- **Archivo principal:** `docs/ga-tl-02/informe.tex`.
- **Bibliografía:** el informe no cita fuentes, así que no usa `biber` ni `bibtex`.
- **Orden de comandos**, dentro de `docs/ga-tl-02/`:

```
pdflatex informe
pdflatex informe
```

La segunda pasada resuelve las referencias a tablas. El resultado es `docs/ga-tl-02/informe.pdf`, que se incluye en el repositorio. Los archivos auxiliares de la compilación (`.aux`, `.log`, `.out`) están en `.gitignore`.

- **Dependencias:** paquetes `babel` (español), `fontenc`, `inputenc`, `lmodern`, `geometry`, `booktabs`, `longtable`, `array` e `hyperref`. En TeX Live hacen falta `texlive-lang-spanish` y `texlive-latex-extra`.

## Diagramas

Los `.drawio` se abren con [diagrams.net](https://app.diagrams.net) o con la extensión de draw.io para VS Code. Los `.pdf` son su exportación.
