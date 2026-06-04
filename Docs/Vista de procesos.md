# Vista de Procesos — FilmStars

---

## Índice

1. [Procesos y Hilos](#1-arquitectura-de-procesos-y-hilos)
2. [Topología del Broker de Mensajería](#2-topología-del-broker-de-mensajería)
3. [Flujos Productor-Consumidor Detallados](#3-flujos-productor-consumidor-detallados)
4. [Mecanismo de Control de Concurrencia](#4-mecanismo-de-control-de-concurrencia)
5. [Diagrama de Estados del Asiento](#5-diagrama-de-estados-del-asiento)
6. [Descripción del Diagrama de Secuencia](#6-descripción-del-diagrama-de-secuencia)
7. [Matriz de Responsabilidades de Procesos](#7-matriz-de-responsabilidades-de-procesos)
8. [Trazabilidad con Casos de Uso y Requerimientos](#8-trazabilidad-con-casos-de-uso-y-requerimientos)

---

## 1. Arquitectura de Procesos y Hilos

El sistema opera bajo un modelo **híbrido**: consultas de lectura son síncronas (REST) para mantener fluidez en la UI, mientras que todas las operaciones de escritura críticas (bloqueo de asientos, pagos, emisión de boletos) se desacoplan mediante procesos asíncronos en el broker.

### Procesos por Servicio

| Servicio | Proceso Principal | Hilos/Workers | Naturaleza |
|:---|:---|:---|:---|
| **User Service** | API REST de autenticación y perfiles | HTTP Handler (síncrono) | Síncrono |
| **Movie Service** | API REST de cartelera, funciones y horarios | HTTP Handler (síncrono) | Síncrono |
| **Reservation Service** | API REST de consulta + **Worker de Bloqueo** + **Worker de Liberación** + **Worker de Ticket** | HTTP Handler + 3 Consumer Threads | Híbrido |
| **Payment Service** | API REST de consulta + **Worker de Validación Financiera** | HTTP Handler + 1 Consumer Thread | Híbrido |
| **Notification Service** | **Worker de Envío de Correos** | 2 Consumer Threads (alta disponibilidad) | Asíncrono |

---

## 2. Topología del Broker de Mensajería

Se usará **Exchange tipo `topic`** central (`filmstars.direct`) para enrutar eventos por dominio, garantizando que los mensajes críticos persistan en disco (`durable: true`) y que las colas sean resilientes a reinicios.

### Colas y Routing Keys

| Cola | Routing Key | Productor(es) | Consumidor(es) | Propósito |
|:---|:---|:---|:---|:---|
| `queue.seat.blocking` | `reservation.seat.block` | Reservation Service (API) | **Reservation Service Worker** (1 instancia activa por `function_id`) | Serializar bloqueos de asientos y evitar condiciones de carrera |
| `queue.seat.release` | `reservation.seat.release` | Reservation Service (Scheduler/TTL) + Payment Service (compensación) | **Reservation Service Worker** | Liberar asientos expirados o por pago fallido |
| `queue.payment.validate` | `payment.validate` | Reservation Service (tras confirmar bloqueo) | **Payment Service Worker** | Procesar pago simulado de forma segura |
| `queue.ticket.generate` | `ticket.generate` | Payment Service (tras pago exitoso) | **Reservation Service Worker** | Emitir boleto y marcar asiento como OCUPADO |
| `queue.notification.email` | `notification.email` | Reservation Service, Payment Service, Ticket Worker | **Notification Service Worker** | Enviar confirmaciones, boletos y alertas |

---

## 3. Flujos Productor-Consumidor Detallados

### Flujo A: Bloqueo Temporal de Asientos (Anti-condición de carrera)

**Lógica del Worker de Bloqueo:**
1. Recibe mensaje con `user_id`, `function_id`, `seat_id`, `version`, `timestamp`.
2. Ejecuta **CAS (Compare-And-Swap)** atómico sobre la fila del asiento.
3. Si `affected_rows=1` (CAS exitoso): actualiza a `BLOQUEADO`, guarda `blocked_by` y `expires_at = NOW() + 5min`.
4. Si `affected_rows=0` (CAS fallido): el asiento ya fue tomado. Publica mensaje a `queue.notification.email` con routing key `notification.email.error`.
5. Confirma **ACK** al broker solo si la transacción de BD fue exitosa. En caso de error, usa **NACK** con requeue para reintento.


---

### Flujo B: Expiración de Reserva (Liberación Automática)

1. El Scheduler interno del Reservation Service se autodispara cada **30 segundos** (Self-Message / cron interno).
2. Ejecuta query sobre la BD Reservation: `SELECT seat_id, blocked_by, version, expires_at FROM seats WHERE status = 'BLOQUEADO' AND expires_at < NOW()`.
3. Si el resultado es vacío: no hay asientos expirados. El Scheduler duerme 30 segundos y vuelve al paso 1.
4. Si hay registros expirados: por cada asiento encontrado, construye un mensaje con `seat_id`, `function_id`, `blocked_by` (para auditoría), `version` actual y `reason = 'TTL_EXPIRED'`.
5. Publica cada mensaje en la cola `queue.seat.release` del Message Broker con `persistent = true` y `delivery_mode = 2`.
6. El Broker encola los mensajes y los entrega al **Worker de Liberación** (Consumer del Reservation Service).
7. El Worker de Liberación consume el mensaje de `queue.seat.release` con `prefetch_count = 1`.
8. Verifica que el asiento siga en estado `BLOQUEADO` en la fuente de verdad: `SELECT status, version FROM seats WHERE seat_id = X FOR UPDATE`.
9. Si el estado ya cambió (ej. a `VENDIDO` o `OCUPADO` por una compra concurrente): el Worker descarta el mensaje, hace **ACK** al broker y termina sin modificar nada.
10. Si el estado sigue siendo `BLOQUEADO`: ejecuta **CAS atómico**: `UPDATE seats SET status = 'DISPONIBLE', blocked_by = NULL, version = version + 1 WHERE seat_id = X AND version = N`.
11. Si `affected_rows = 1` (CAS exitoso): la liberación fue exitosa. Registra el evento en tabla `audit_logs` (RNF-20) con `action = 'LIBERACION_TTL'`, `seat_id`, `timestamp` y `reason`.
12. Si `affected_rows = 0` (CAS fallido): otro proceso modificó el asiento entre el SELECT y el UPDATE. El Worker hace **NACK** con `requeue = false` (para evitar loop infinito) y registra el conflicto en logs.
13. El Worker hace **ACK** al broker solo si la transacción de BD fue exitosa.
14. El Scheduler espera 30 segundos y vuelve al paso 1.

---

### Flujo C: Proceso de Pago Simulado (Transacción Segura)

**Mecanismo de Idempotencia (anti-pérdida ante caídas):**
- El mensaje lleva header `idempotency-key: <order_id>`.
- El Payment Service mantiene tabla `processed_payments(idempotency_key PK, status, created_at)`.
- Si recibe un mensaje con key ya existente y estado `EXITOSO`: reenvía confirmación sin reprocesar.
- Si estado es `PROCESANDO`: espera o consulta estado actual.
- La cola usa `delivery_mode=persistent` y `acknowledgement=manual` (ACK solo tras commit en BD).

---

## 4. Mecanismo de Control de Concurrencia: Asientos

Este es el requisito crítico del sistema. Se implementa una **estrategia híbrida de tres capas**:

### Capa 1: Serialización por Cola (Aplicación)
`queue.seat.blocking` tiene **prefetch_count=1** y es consumida por un único worker activo por partición (particionamos por `function_id` si necesitamos escalar). Esto garantiza que dos mensajes para la misma función no se procesen simultáneamente en el mismo worker.

### Capa 2: Lock Optimista en Base de Datos (Persistencia)

Tabla `seats` en la BD de Reservation Service:

```sql
CREATE TABLE seats (
    seat_id       UUID PRIMARY KEY,
    function_id   UUID NOT NULL,
    status        ENUM('DISPONIBLE', 'BLOQUEADO', 'VENDIDO', 'OCUPADO') DEFAULT 'DISPONIBLE',
    blocked_by    UUID NULL,
    version       INT DEFAULT 0,
    expires_at    TIMESTAMP NULL,
    updated_at    TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
);
```

El Worker de Bloqueo:
1. Lee fila: `SELECT * FROM seats WHERE seat_id=X AND function_id=Y`
2. Verifica `status='DISPONIBLE'`
3. Actualiza: `UPDATE seats SET status='BLOQUEADO', blocked_by=USER, version=version+1 WHERE seat_id=X AND version=Z`
4. Si `affected_rows=0`: otro proceso lo modificó. Rechaza y notifica.

### Capa 3: TTL y Compensación (Negocio)
- Un asiento `BLOQUEADO` tiene TTL de **5 minutos exactos** (RNF-09).
- Si el usuario no completa el pago, el scheduler publica liberación (Flujo B).
- Si el Payment Service falla, publica mensaje de compensación a `queue.seat.release`.

---

## 5. Estados de Asientos

**Eventos disparadores:**

| Transición | Evento | Proceso Responsable | CU |
|:---|:---|:---|:---|
| DISPONIBLE → BLOQUEADO | Usuario selecciona asiento (CAS exitoso) | Reservation Service Worker (por cola) | CU-302, CU-305 |
| BLOQUEADO → DISPONIBLE | TTL expira (5 min) o pago fallido | Scheduler / Payment Service (compensación) | CU-307, CU-410 |
| BLOQUEADO → OCUPADO | Pago validado exitosamente | Payment Service → Ticket Worker | CU-404, CU-407 |
| OCUPADO → LIBERADO | Cancelación administrativa o reembolso | Proceso manual/admin | — |
| LIBERADO → DISPONIBLE | Liberación completada (compensación finalizada) | Scheduler / Worker automático | CU-307 |

**Invariante de Consistencia (RNF-10):** En cualquier instante t, un asiento tiene exactamente UN estado. No existen estados superpuestos ni transiciones parciales.

---

## 6. Descripción del Diagrama de Secuencia

### Participantes (11 líneas de vida)

| # | Participante | Tipo | Rol |
|:---|:---|:---|:---|
| 1 | **Usuario** | Actor | Cliente final |
| 2 | **Frontend / API Gateway** | Servicio | Punto de entrada único, enrutamiento, WebSocket/SSE |
| 3 | **Movie Service** | Servicio | Catálogo, ciudades, cines, funciones |
| 4 | **Reservation Service** | Servicio | Core: asientos, bloqueos, órdenes, boletos |
| 5 | **BD Reservation** | Base de datos | Persistencia de asientos, bloqueos, órdenes |
| 6 | **Message Broker** | Middleware | RabbitMQ/Kafka: colas de mensajería |
| 7 | **Consumer Reservations** | Worker | Consumidor de colas de asientos/boletos |
| 8 | **Payment Service** | Servicio | Procesamiento de pagos simulados |
| 9 | **BD Payment** | Base de datos | Persistencia de transacciones de pago |
| 10 | **Consumer Payments** | Worker | Consumidor de colas de validación de pagos |
| 11 | **Notification Service** | Servicio | Envío de emails con boletos y alertas |

### Escenario: Compra exitosa concurrente de 2 usuarios que intentan el mismo asiento; solo uno gana.

| Paso | Actor | Acción | Tipo de mensaje |
|:---|:---|:---|:---|
| 1 | Usuario A y B | `GET /movies/functions` (consulta cartelera) | ⬛ Síncrono |
| 2 | Usuario A | `POST /seats/hold` (asiento F5, función 12) | ⬛ Síncrono |
| 3 | Reservation Service | Verifica disponibilidad: `SELECT status, version` | ⬛ Síncrono |
| 4 | BD Reservation | Retorna: `DISPONIBLE, version=N` | --- Retorno |
| 5 | Reservation Service | **Publish** `queue.seat.blocking` | ⬜ Asíncrono |
| 6 | Broker | Encola mensaje A | — |
| 7 | Consumer Reservations | Consume msg A, ejecuta **CAS** `UPDATE ... version=N` | ⬛ Síncrono |
| 8 | BD Reservation | Éxito. `affected_rows=1`. Estado F5 → BLOQUEADO. | — |
| 9 | Consumer Reservations | **ACK** al broker | --- Retorno |
| 10 | Reservation Service | Responde HTTP 202 a Usuario A: "Retenido 5 min" | --- Retorno |
| 11 | Usuario B | `POST /seats/hold` (mismo F5, función 12) | ⬛ Síncrono |
| 12 | Reservation Service | Verifica disponibilidad: ya está BLOQUEADO | ⬛ Síncrono |
| 13 | Reservation Service | Responde HTTP 409 a Usuario B: "Asiento no disponible" | --- Retorno |
| 14 | Usuario A | `POST /order/confirm` | ⬛ Síncrono |
| 15 | Reservation Service | Valida bloqueos vigentes, crea orden `idempotency_key=reserva_001` | ⬛ Síncrono |
| 16 | Usuario A | `POST /payment/process` | ⬛ Síncrono |
| 17 | Reservation Service | Verifica bloqueo vigente. **Publish** `queue.payment.validate` | ⬜ Asíncrono |
| 18 | Frontend | HTTP 202 Accepted: "Procesando compra..." | --- Retorno |
| 19 | Consumer Payments | Consume, procesa pago simulado | ⬛ Síncrono |
| 20 | Payment Service | `INSERT` en BD Payment con `idempotency_key` | ⬛ Síncrono |
| 21 | Payment Service | Commit en BD Payment. **Publish** `queue.ticket.generate` | ⬜ Asíncrono |
| 22 | Consumer Reservations | Consume, **UPDATE** BD Reservation: estado=OCUPADO, genera `ticket_code` | ⬛ Síncrono |
| 23 | Consumer Reservations | **Publish** `queue.notification.email` con `ticket_code` | ⬜ Asíncrono |
| 24 | Notification Service | Consume, genera PDF/QR, envía email | ⬛ Síncrono |
| 25 | Frontend | SSE Push: `order.status='COMPLETADA'` | ⬜ Asíncrono |
| 26 | Usuario A | Recibe boleto en pantalla + email | — |

---

## 7. Matriz de Responsabilidades de Procesos

| Proceso/Worker | Servicio | Cola(s) que consume | Cola(s) que produce | BD que modifica | Naturaleza |
|:---|:---|:---|:---|:---|:---|
| HTTP API Auth | User Service | — | — | `db_users` | Síncrono |
| HTTP API Movies | Movie Service | — | — | `db_movies` | Síncrono |
| HTTP API Reservas | Reservation Service | — | `queue.seat.blocking`, `queue.payment.validate` | `db_reservations` (lectura) | Síncrono |
| **Frontend / API Gateway** | **Frontend** | — | `queue.seat.blocking` (vía API), `queue.payment.validate` (vía API) | — | **Síncrono** |
| **Worker Bloqueo** | Reservation Service | `queue.seat.blocking` | `queue.notification.email` | `db_reservations` | **Asíncrono** |
| **Worker Liberación** | Reservation Service | `queue.seat.release` | `queue.notification.email` | `db_reservations` | **Asíncrono** |
| **Worker Ticket** | Reservation Service | `queue.ticket.generate` | `queue.notification.email` | `db_reservations` | **Asíncrono** |
| **Scheduler TTL** | Reservation Service | *(interno: cron)* | `queue.seat.release` | `db_reservations` | **Asíncrono** |
| HTTP API Pagos | Payment Service | — | — | `db_payments` (lectura) | Síncrono |
| **Worker Pagos** | Payment Service | `queue.payment.validate` | `queue.ticket.generate`, `queue.seat.release`, `queue.notification.email` | `db_payments` | **Asíncrono** |
| **Worker Email** | Notification Service | `queue.notification.email` | — | `db_notifications` (logs) | **Asíncrono** |

---

## 8. Trazabilidad con Casos de Uso y Requerimientos

| Sección de Vista de Procesos | Casos de Uso asociados | Requerimientos Funcionales | Requerimientos No Funcionales |
|:---|:---|:---|:---|
| Flujo A (Bloqueo) | CU-301, CU-302, CU-304, CU-305 | RF-11, RF-12, RF-13, RF-14, RF-15 | RNF-02 (≤1s), RNF-05 (50 usuarios), RNF-09 (5min TTL), RNF-10 (sin doble compra) |
| Flujo B (Expiración) | CU-307 | RF-17 | RNF-09 (5min TTL) |
| Flujo C (Pago) | CU-401, CU-402, CU-403, CU-404, CU-405, CU-406 | RF-19, RF-20, RF-21, RF-22, RF-23, RF-24, RF-25, RF-26 | RNF-07 (entorno aislado), RNF-08 (mensajes no se pierden), RNF-14 (≤3s en cola) |
| Flujo D (Boleto) | CU-407, CU-408, CU-409, CU-410, CU-411 | RF-21, RF-22, RF-23, RF-25, RF-27 | RNF-17 (1M transacciones), RNF-20 (auditoría) |
| Diagrama de Estados | CU-302, CU-305, CU-307, CU-404 | RF-12, RF-14, RF-15, RF-17 | RNF-10 (consistencia) |
| Diagrama de Secuencia | Todos los anteriores | Todos los RF críticos | Todos los RNF de concurrencia y tolerancia |

---

*Documento de Vista de Procesos — FilmStars · Práctica 1, Software Avanzado (USAC)*
