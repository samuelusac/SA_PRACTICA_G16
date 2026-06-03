# Vista de Procesos — Modelo 4+1

## 1. Objetivo
La Vista de Procesos describe los **hilos de ejecución**, **procesos del sistema** y los **flujos de comunicación asíncrona** entre los servicios SOA de FilmStars. Se enfoca en cómo la plataforma gestiona la **concurrencia crítica** (selección de asientos) y la **resiliencia** (pérdida de transacciones) mediante un middleware de mensajería, sin bloquear el hilo principal de la aplicación web.

---

## 2. Procesos Identificados

| Proceso | Tipo | Descripción | Servicio Asociado |
|:---|:---|:---|:---|
| **P1. API Gateway / Web Server** | Proceso Principal | Punto de entrada único. Expone REST API al frontend. Enruta peticiones síncronas y publica mensajes asíncronos. | `svc-gateway` |
| **P2. Gestor de Usuarios** | Proceso Independiente | Autenticación, sesiones multiperfil, validación de JWT. | `svc-users` |
| **P3. Catálogo y Cartelera** | Proceso Independiente | Consulta de películas, categorías (Estrenos, Pre-ventas, Re-estrenos), funciones por ciudad. | `svc-movies` |
| **P4. Motor de Reservas** | Proceso Independiente | Lógica de bloqueo temporal, mapa de asientos en tiempo real, expiración de reservas. | `svc-reservations` |
| **P5. Procesador de Pagos** | Proceso Independiente | Simulación de pasarela de pago, emisión de boletos, compensación financiera. | `svc-payments` |
| **P6. Broker de Mensajería** | Middleware | RabbitMQ (cluster de nodos). Gestiona colas, exchanges y routing de mensajes. | `rabbitmq` |
| **P7. Worker de Validación de Asientos** | Consumidor Asíncrono | Valida atomicidad de asientos seleccionados contra la base de datos. | `svc-reservations` |
| **P8. Worker de Confirmación Financiera** | Consumidor Asíncrono | Procesa pagos de forma asíncrona para evitar bloqueo del gateway. | `svc-payments` |
| **P9. Worker de Expiración** | Daemon/Timer | Revisa periódicamente reservas temporales vencidas y libera asientos. | `svc-reservations` |

---

## 3. Hilos de Ejecución por Proceso

### 3.1 Hilo Principal del Gateway (P1)
- **Hilo HTTP Listener:** Atiende peticiones REST del cliente (síncronas para consultas, asíncronas para operaciones críticas).
- **Hilo Publisher:** Publica mensajes en el broker sin esperar respuesta inmediata (fire-and-forget para resiliencia).
- **Hilo WebSocket Manager:** Mantiene conexiones activas para notificar al cliente cambios en el mapa de asientos.

### 3.2 Hilo del Motor de Reservas (P4)
- **Hilo REST API:** Atiende consultas de disponibilidad (síncrona, respuesta inmediata).
- **Hilo Seat Locker:** Ejecuta bloqueo temporal con TTL (Time-To-Live) en Redis o BD.
- **Hilo Consumer (P7):** Escucha la cola `queue.seat.validation` para validar atomicidad.

### 3.3 Hilo del Procesador de Pagos (P5)
- **Hilo REST API:** Expone endpoints para estado de transacción.
- **Hilo Consumer (P8):** Escucha `queue.payment.process` para ejecutar la pasarela de pago simulada.
- **Hilo Ticket Emitter:** Genera el boleto digital tras confirmación de pago.

### 3.4 Hilo Daemon de Expiración (P9)
- Ejecuta cada **30 segundos** (configurable).
- Consulta reservas con `status = 'PENDING'` y `created_at < NOW() - INTERVAL '10 minutes'`.
- Publica mensaje de liberación en `queue.seat.release`.

---

## 4. Comunicación Asíncrona: Productores y Consumidores

### 4.1 Topología del Broker (RabbitMQ)

Se utiliza un **Exchange de tipo Topic** (`filmstars.exchange`) para permitir routing flexible basado en patrones de clave.

| Exchange | Tipo | Routing Key Pattern | Descripción |
|:---|:---|:---|:---|
| `filmstars.exchange` | `topic` | `reservation.*` | Eventos del dominio de reservas |
| `filmstars.exchange` | `topic` | `payment.*` | Eventos del dominio de pagos |
| `filmstars.exchange` | `topic` | `notification.*` | Notificaciones al cliente |

### 4.2 Colas Definidas

| Cola | Binding Key | Productor | Consumidor | Durabilidad | TTL |
|:---|:---|:---|:---|:---|:---|
| `queue.seat.validation` | `reservation.validate` | `svc-gateway` | `Worker P7` (svc-reservations) | Durable | — |
| `queue.seat.confirm` | `reservation.confirm` | `Worker P7` | `Worker P8` (svc-payments) | Durable | — |
| `queue.payment.process` | `payment.process` | `Worker P7` | `Worker P8` (svc-payments) | Durable | — |
| `queue.payment.result` | `payment.result` | `Worker P8` | `svc-gateway` (WebSocket) | Durable | — |
| `queue.seat.release` | `reservation.release` | `Worker P9` / `Worker P8` (fallback) | `Worker P7` | Durable | — |
| `queue.ticket.emit` | `payment.success` | `Worker P8` | `svc-gateway` / Email Service | Durable | — |
| `queue.dead.letter` | `#` | Todas las colas (DLX) | Admin/Logging | Durable | — |

### 4.3 Flujo de Productores

| Evento | Productor | Routing Key | Payload (simplificado) |
|:---|:---|:---|:---|
| Usuario selecciona asientos | `svc-gateway` | `reservation.validate` | `{user_id, function_id, seats: ['A1','A2'], timestamp, correlation_id}` |
| Asientos validados exitosamente | `Worker P7` | `payment.process` | `{reservation_id, amount, payment_method, correlation_id}` |
| Pago procesado | `Worker P8` | `payment.result` | `{reservation_id, status: 'SUCCESS'/'FAILED', ticket_id?}` |
| Reserva expirada por tiempo | `Worker P9` | `reservation.release` | `{reservation_id, seats: ['A1','A2'], reason: 'TTL_EXPIRED'}` |
| Pago exitoso → Emitir boleto | `Worker P8` | `payment.success` | `{user_email, ticket_id, function_details, qr_code}` |

### 4.4 Flujo de Consumidores

| Consumidor | Cola Suscrita | Lógica de Procesamiento |
|:---|:---|:---|
| **Worker P7** (Validación de Asientos) | `queue.seat.validation` | 1. Inicia transacción BD con nivel de aislamiento `SERIALIZABLE` o usa `SELECT FOR UPDATE`.<<br>2. Verifica que los asientos estén en estado `AVAILABLE`.<<br>3. Si sí: cambia a `LOCKED`, crea reserva temporal, publica en `queue.seat.confirm`.<<br>4. Si no: publica `reservation.failed` con razón `SEATS_UNAVAILABLE`. |
| **Worker P8** (Confirmación Financiera) | `queue.payment.process` | 1. Simula pasarela de pago (proceso de 2-5 segundos).<<br>2. Si éxito: actualiza reserva a `CONFIRMED`, publica `payment.success` y `payment.result`.<<br>3. Si fallo: publica `reservation.release` para liberar asientos y `payment.result` con error. |
| **Worker P9** (Expiración) | `queue.seat.release` (también publica aquí) | 1. Cambia estado de asientos a `AVAILABLE`.<<br>2. Actualiza reserva a `EXPIRED`.<<br>3. Notifica al usuario vía WebSocket si está conectado. |
| **Gateway WebSocket** | `queue.payment.result` | Notifica al cliente en tiempo real: "Pago confirmado" o "Asiento liberado". |

---

## 5. Manejo de Concurrencia: Prevención de Condiciones de Carrera

### 5.1 Problema Crítico
Escenario: Dos usuarios (U1 y U2) seleccionan simultáneamente el asiento **A1** de la misma función. Sin control, ambos podrían recibir confirmación de disponibilidad, causando sobreventa.

### 5.2 Estrategia de Resolución

Se implementa una estrategia de **optimistic locking con fallback a cola serializada**:

#### Nivel 1: Bloqueo Optimista en Base de Datos
```sql
-- Tabla seats tiene columna version (integer) o status con control de transición
UPDATE seats 
SET status = 'LOCKED', locked_by = :user_id, locked_at = NOW() 
WHERE seat_id = 'A1' 
  AND function_id = 123 
  AND status = 'AVAILABLE';
-- Si rows_affected = 0, el asiento ya fue tomado.
```

#### Nivel 2: Serialización por Cola (Message Broker)
- Todos los mensajes de `reservation.validate` para la **misma función** se enrutan a una **cola particionada** (usando `function_id` como shard key o consistent hashing).
- RabbitMQ garantiza que los mensajes de una misma partición se procesen **secuencialmente** por un único consumidor, eliminando race conditions en la validación.

#### Nivel 3: TTL y Liberación Automática
- Un asiento bloqueado tiene un **TTL de 10 minutos** en Redis/BD.
- Si el usuario no completa el pago, `Worker P9` libera el asiento automáticamente.
- El frontend recibe heartbeat cada 30 segundos para refrescar el mapa de asientos.

### 5.3 Diagrama de Estados del Asiento

```
[AVAILABLE] --(usuario selecciona)--> [LOCKED] --(pago exitoso)--> [OCCUPIED]
     ^                                      |
     |                                      | (pago fallido / TTL expira)
     +----------[RELEASED]<------------------+
```

---

## 6. Diagrama de Flujo de Procesos (Texto para Draw.io)

Puedes usar esta descripción para construir tu diagrama en **Draw.io** o **LucidChart**:

```
[Cliente/Frontend]
       |
       | HTTP POST /api/reservations/validate
       v
[API Gateway (P1)] ----(síncrono)----> [svc-users] Validar JWT
       |
       | (asíncrono) Publish to exchange
       v
[RabbitMQ: filmstars.exchange]
       | routing_key: reservation.validate
       v
[Queue: queue.seat.validation] ----(consume)----> [Worker P7 (svc-reservations)]
       |                                              |
       | 1. SELECT FOR UPDATE / SERIALIZABLE            |
       | 2. UPDATE seat status = LOCKED                 |
       | 3. INSERT temporary reservation (TTL 10min)    |
       v
[Publish: payment.process] ----> [Queue: queue.payment.process]
                                      |
                                      v
                              [Worker P8 (svc-payments)]
                                      |
                                      | 1. Simular pasarela de pago
                                      | 2. IF success:
                                      |    - UPDATE reservation = CONFIRMED
                                      |    - UPDATE seat = OCCUPIED
                                      |    - Publish: payment.success
                                      | 3. IF fail:
                                      |    - Publish: reservation.release
                                      v
[Queue: payment.result] <----(Publish)----+
       |
       | (consume)
       v
[Gateway WebSocket] ----> Notifica al Cliente: "Boleto emitido" o "Pago rechazado"

[Daemon P9 (cada 30s)] ----> Revisa reservas expiradas
       |
       | Publish: reservation.release
       v
[Queue: queue.seat.release] ----> [Worker P7] Libera asiento a AVAILABLE
```

---

## 7. Resumen de Atributos de Calidad Cubiertos

| Atributo | Cómo se satisface en esta Vista |
|:---|:---|
| **Disponibilidad** | El gateway no se bloquea; las operaciones críticas se delegan a workers. |
| **Escalabilidad** | Los workers P7, P8 y P9 pueden replicarse horizontalmente (múltiples instancias consumiendo de la misma cola). |
| **Consistencia** | Serialización en la BD + particionamiento de colas por función evitan race conditions. |
| **Tolerancia a fallos** | Dead Letter Exchange (`queue.dead.letter`) captura mensajes fallidos para reintento o auditoría. |
| **Rendimiento** | Las consultas de catálogo (P3) permanecen síncronas y rápidas; solo la compra es asíncrona. |

---

## 8. Recomendaciones para tu Diagrama UML

Cuando dibujes el **Diagrama de Procesos** formal en UML (vista de procesos del 4+1):

1. **Usa notación de UML 2.5:** Procesos como "swimlanes" o "partitions" en un diagrama de actividades.
2. **Diferencia visualmente:**
   - **Líneas sólidas** para comunicación síncrona (REST).
   - **Líneas punteadas con flecha abierta** para comunicación asíncrona (mensajes al broker).
   - **Líneas onduladas** para eventos de tiempo (daemon P9).
3. **Incluye el broker como "central node"** en el centro del diagrama.
4. **Anota los routing keys** sobre las flechas asíncronas.
