# Drivers de Requisitos Funcionales (DRF)
---

## Selección de Ubicación y Funciones
| ID       |Requerimiento  | Prioridad |
|--------  |-------------------------------------------------------------------------------|-----------|
| RF-01    | El sistema de permitir al usuario seleccionar una ciudad de listado disponible | Alta      |
| RF-02    | El sistema de ser capaz de comstrar los cindes disponibles en la ciudad seleccionada por el usuario | Alta      |
| RF-03    | EEl sistema debe ser capa de mostrar mostrar la funcion, con su horario y numero sala, disponibles para la cuidad seleccionada por el usuario  | Alta      |
| RF-04    | El sistema de permitir filtrar las pelicuas por fecha y horario    | Media     |
| RF-05   | El sistema debe ser capaz de mostrar el genero y los datos de la pelicula segun la fucncion | Media     |

---

## Visualización de películas por categoría
| ID     | Requerimiento| Prioridad |
|--------|-------------------------------------------------------------------------------|-----------|
| RF-06  | El sistema debe ser capaz de clasificar su cartelera en tres categorias segun su tipo de proyección  Estrenos, Pre-ventas, Re-Estrenos| Alta      |
| RF-07  | El sistea debe ser capaz de mostra todas las pelicuasl segun su clasificaion seleccionada | Alta      |
| RF-08  | El sistema debe ser capaz de mostrar un banner con los Estrenos de las peliculas | Media     |
| RF-09  | El sistema de se capaz de mostrar al usuario las caracterisiticas la pelicula (titulo, sinopsis, duracion, genero)  | Media     |
| RF-10  | El sistema debe ser capaz de cambiar la categoria de una pelicula autiomaticamente segun cambie el estado | Baja      |

---

## Selección de Asientos en Tiempo Real

| ID     | Requerimiento  | Prioridad |
|--------|-------------------------------------------------------------------------------|-----------|
| RF-11  | El sistema debe ser capaz de mostrar un mapa de la sala el cual sea interactivo  | Alta      |
| RF-12  | El mapa interactivo de mostrar los tres estados de un asiento (Disponible, ocupado, Seleccionadao, bloqueado temporalmente), debe mostrar de maneara agradabel para el usuario | Alta      |
| RF-13  | El sistema debe permitir al usuario seleccionar y desleccionar los asientos disponibles| Alta      |
| RF-14  | El sistema debe ser capaz de bloquear temporalmente los asientos seleccionados del cliente durante el proceso de compra | Alta      |
| RF-15  | El sistema debe impedir que dos usuarios seleccionen el mismo asiento simultáneamente  | Alta      |
| RF-16  | El sistema debe mostrar un mensaje de error si el usuario intenta seleccionar un asiento ya ocupado o bloqueado | Media     |
| RF-17  | El sistema debe liberar automáticamente los asientos bloqueados si la reserva temporal expira sin completar la compra | Media     |
| RF-18  | El sistema debe mostrar un temporizador visual durante el bloqueo temporal de asientos | Baja      |

---

## Flujo de Compra de Boletos

| ID     | Requerimiento   | Prioridad |
|--------|-------------------------------------------------------------------------------|-----------|
| RF-19  | El sistema debe ser capaz de enviar solicitudes de validación de los asientos a una cola de mensajeria  | Alta      |
| RF-20  | El sistema debe procesar un pago simulado después de la validación exitosa de los asientos | Alta      |
| RF-21  | El sistema debe emitir un boleto digital solo después de completar el pago y la validación | Alta      |
| RF-22  | El sistema debe notificar al usuario si la validación de asientos falla (ej. asientos ya no disponibles) | Alta      |
| RF-23  | El sistema debe notificar al usuario si el pago  es rechazado.  | Media     |
| RF-24  | El sistema debe permitir reintentar el pago hasta 3 veces en caso de error transitorio | Media     |
| RF-25  | El sistema debe registrar cada transacción de compra (exitosa o fallida) en un historial para auditoría | Media     |
| RF-26  | El sistema debe mostrar un resumen de la compra (película, función, asientos, total) antes de confirmar el pago | Alta      |
| RF-27  | El sistema debe generar un código único para cada boleto emitido     | Media     |
