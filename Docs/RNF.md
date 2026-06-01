# Requerimientos No Funcionales (RNF)

| ID     | Atributo              | Especificación Cuantitativa                                                                 |
|--------|-----------------------|---------------------------------------------------------------------------------------------|
| RNF-01 | Tiempo de respuesta   | El mapa interactivo de asientos debe cargarse y mostrar los estados actualizados en ≤ 2 segundos. |
| RNF-02 | Tiempo de respuesta   | La confirmación de selección de asientos (bloqueo temporal) debe ser ≤ 1 segundo.            |
| RNF-03 | Disponibilidad        | El sistema debe tener una disponibilidad del 99.5% durante horas de alta demanda.            |
| RNF-04 | Concurrencia          | El sistema debe soportar al menos 1000 usuarios simultáneos seleccionando asientos.          |
| RNF-05 | Concurrencia crítica  | No deben ocurrir condiciones de carrera al seleccionar el mismo asiento por hasta 50 usuarios simultáneos. |
| RNF-06 | Seguridad             | Todas las comunicaciones entre servicios deben usar autenticación JWT o cookies seguras.     |
| RNF-07 | Seguridad de pago     | El pago simulado debe procesarse en un entorno aislado sin exponer datos sensibles en logs. |
| RNF-08 | Tolerancia a fallos   | Los mensajes en cola no deben perderse ante caídas de servicios.                             |
| RNF-09 | Tiempo de bloqueo     | El tiempo de reserva temporal de un asiento debe ser de 5 minutos exactos.                   |
| RNF-10 | Consistencia          | El sistema no debe permitir que dos usuarios compren el mismo asiento.                       |
| RNF-11 | Carga máxima          | El sistema debe manejar picos de hasta 2000 peticiones por segundo.                          |
| RNF-12 | Tiempo de inactividad | El tiempo máximo de inactividad planificada no debe exceder 2 horas al mes.                  |
| RNF-13 | Escalabilidad         | Los servicios críticos deben poder escalar horizontalmente a 5 réplicas.                     |
| RNF-14 | Tiempo de procesamiento asíncrono | El mensaje de validación debe procesarse en ≤ 3 segundos desde la cola.            |
| RNF-15 | Error humano          | El sistema debe mostrar un mensaje claro en ≤ 0.5 segundos si el asiento ya está ocupado.    |
| RNF-16 | Latencia de red       | Las peticiones entre servicios internos deben tener una latencia promedio ≤ 100 ms.          |
| RNF-17 | Capacidad de almacenamiento | El sistema debe almacenar al menos 1 millón de transacciones sin pérdida de rendimiento. |
| RNF-18 | Tiempo de recuperación | En caso de fallo crítico, el sistema debe recuperarse en ≤ 15 minutos.                      |
| RNF-19 | Punto único de fallo  | Ningún componente crítico debe ser un punto único de fallo.                                  |
| RNF-20 | Seguridad de asientos | El sistema debe auditar cualquier cambio de estado en asientos.                              |
