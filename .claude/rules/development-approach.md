# Development Approach

Principios que rigen todas las decisiones técnicas en este proyecto.

## Principios fundamentales

### 1. Enfoque crítico permanente
Antes de aceptar cualquier propuesta —propia o del usuario— evaluarla activamente buscando:
- Fallos técnicos o inconsistencias
- Supuestos no verificados
- Riesgos no considerados
- Alternativas más simples

No dar por válido algo solo porque parece razonable.

### 2. Análisis profundo antes de implementar
Ninguna implementación comienza sin haber respondido:
- ¿Qué problema resuelve exactamente?
- ¿Es la solución más simple para este contexto?
- ¿Qué puede salir mal?
- ¿Cómo se deshace si falla?

### 3. Preferencia por herramientas consolidadas
Orden de preferencia al elegir una solución:
1. Herramienta existente, ampliamente adoptada por la comunidad, con documentación activa
2. Combinación de herramientas estándar con configuración mínima
3. Script propio solo si las opciones anteriores no cubren el caso

**Nunca** proponer una solución custom cuando existe una herramienta madura que hace lo mismo.

### 4. Creatividad reducida — estabilidad máxima
Este proyecto prioriza fiabilidad y bajo mantenimiento sobre originalidad.
- Preferir lo aburrido y probado sobre lo nuevo y elegante
- Evitar patrones experimentales o poco documentados
- Si hay dos opciones equivalentes, elegir la que tiene más adopción y más casos reales documentados

### 5. No inventar
Si se desconoce el comportamiento exacto de una herramienta, configuración o API:
- **No suponer** ni extrapolar
- Indicar explícitamente la incertidumbre
- Consultar la documentación oficial antes de proponer

### 6. Preguntar antes que asumir
Ante ambigüedad en un requisito o duda técnica real:
- Preguntar al usuario antes de implementar con supuestos
- Es preferible una pregunta directa a una implementación incorrecta que hay que rehacer

### 7. Documentación oficial como fuente de verdad
Para cualquier herramienta del stack:
- La fuente primaria es siempre la documentación oficial
- Las respuestas de foros, blogs o tutoriales se contrastan con la doc oficial
- Si la doc oficial y un tutorial discrepan, prevalece la doc oficial

## Aplicación práctica

Ante cualquier tarea de implementación, el flujo es:

```
1. Entender el requisito exacto
2. Identificar herramientas consolidadas que lo resuelvan
3. Consultar documentación oficial de la herramienta elegida
4. Evaluar críticamente la propuesta antes de presentarla
5. Señalar explícitamente riesgos, limitaciones o puntos de incertidumbre
6. Implementar solo tras validación del usuario si hay dudas relevantes
```

## Contexto del proyecto

Asociación española pequeña, servidor único, bajo mantenimiento.
La complejidad técnica debe ser proporcional a este contexto.
Una solución que funciona bien para este caso es mejor que una solución teóricamente óptima pero difícil de mantener.
