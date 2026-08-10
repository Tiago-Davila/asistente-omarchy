---
type: "query"
date: "2026-08-09T19:18:45.526073+00:00"
question: "Por que 'Restricciones que condicionan toda decision tecnica' (CLAUDE.md) es el puente entre los principios de arquitectura de Eva y su gobernanza?"
contributor: "graphify"
outcome: "useful"
source_nodes: ["Restricciones que condicionan toda decision tecnica", "Restricciones Tecnicas y Presupuestos (tabla de limites duros)", "Flujo de Desarrollo y Puertas de Calidad", "VI. Presupuesto de Recursos Explicito y Verificable", "XI. Cien por Ciento Local"]
---

# Q: Por que 'Restricciones que condicionan toda decision tecnica' (CLAUDE.md) es el puente entre los principios de arquitectura de Eva y su gobernanza?

## Answer

Expanded from original query via vocab: [restricciones, decisi, cnica, arquitectura, principios, governance, presupuestos, capas, puertas, flujo, calidad, duros]. Then traversed BFS depth=2 (39 nodes) plus explain on the bridge node. El nodo (CLAUDE.md L30-L60) tiene 14 aristas salientes: 11 dentro de 'Principios de Arquitectura de Eva' y solo 3 cruzando a 'Gobernanza y Presupuestos de Eva' (VI Presupuesto de Recursos, XI Cien por Ciento Local, y la tabla de limites duros de constitution.md L210-L230). Esas 3 aristas son todo el puente; betweenness 0.044. CORRECCION IMPORTANTE: el grafo sobreestima el rol de CLAUDE.md. La lista de puertas de calidad de la constitucion (constitution.md L251-L252) tambien cita VI y XI explicitamente, pero el extractor capturo solo VII de L251 y omitio VI y XI. Con esas dos aristas EXTRACTED presentes, 'Flujo de Desarrollo y Puertas de Calidad' tambien seria puente. Lo que si es real y accionable: la arista semantically_similar_to entre la seccion de CLAUDE.md y la tabla de limites duros de la constitucion documenta que los mismos numeros (3% core, 250MB RSS, 3GB, whisper small, 4B params) estan escritos dos veces en fuentes independientes, sin mecanismo de sincronizacion. Recomendacion: que CLAUDE.md apunte a la tabla en vez de repetir los numeros.

## Outcome

- Signal: useful

## Source Nodes

- Restricciones que condicionan toda decision tecnica
- Restricciones Tecnicas y Presupuestos (tabla de limites duros)
- Flujo de Desarrollo y Puertas de Calidad
- VI. Presupuesto de Recursos Explicito y Verificable
- XI. Cien por Ciento Local