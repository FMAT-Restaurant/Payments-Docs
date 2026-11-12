# Diagramas de Flujo y Secuencia del Microservicio de Cuenta y Pagos

> **Microservicio:** Cuenta y Pagos (`Payment Microservice` - Jira: `PMC`)  
> **Versión:** 2.1 (Sintaxis Mermaid estricta, basada en HUs de Jira y Acuerdos Técnicos del PO)  
> **Documento Complementario a:** [ERS Cuenta y Pagos v3.0](../Docs/ERS_Cuenta_Pagos.md)  
> **Formato de Diagramas:** Mermaid (Renderizable nativamente en GitHub, IDEs y visores Markdown)

---

## Índice de Contenidos

1. [Arquitectura Operativa Global del Microservicio](#1-arquitectura-operativa-global-del-microservicio)
2. [Módulo 1: Sincronización Asíncrona y Bandeja de Caja (PMC-5, PMC-26, PMC-27, PMC-28)](#2-módulo-1-sincronización-asíncrona-y-bandeja-de-caja)
   - [1.1 Diagrama de Flujo: Ingesta y Limpieza de Pre-Cuentas](#11-diagrama-de-flujo-ingesta-y-limpieza-de-pre-cuentas)
   - [1.2 Diagrama de Secuencia: Sincronización y Polling de Pre-Cuentas](#12-diagrama-de-secuencia-sincronización-y-polling-de-pre-cuentas)
3. [Módulo 2: Procesamiento Transaccional de Cobros y Pagos Mixtos (PMC-6, PMC-7, PMC-13)](#3-módulo-2-procesamiento-transaccional-de-cobros-y-pagos-mixtos)
   - [2.1 Diagrama de Flujo: Lógica de Cobro, Validación Financiera y Cálculo de Cambio](#21-diagrama-de-flujo-lógica-de-cobro-validación-financiera-y-cálculo-de-cambio)
   - [2.2 Diagrama de Secuencia: Transacción Atómica de Cobro y Evento Outbox](#22-diagrama-de-secuencia-transacción-atómica-de-cobro-y-evento-outbox)
4. [Módulo 3: Ciclo de Vida del Turno de Caja y Arqueo (PMC-8, PMC-9, PMC-29)](#4-módulo-3-ciclo-de-vida-del-turno-de-caja-y-arqueo)
   - [3.1 Diagrama de Flujo: Apertura, Arqueo en Tiempo Real y Cierre Operativo](#31-diagrama-de-flujo-apertura-arqueo-en-tiempo-real-y-cierre-operativo)
   - [3.2 Diagrama de Secuencia: Arqueo sin Bloqueos y Cierre de Caja](#32-diagrama-de-secuencia-arqueo-sin-bloqueos-y-cierre-de-caja)
5. [Módulo 4: Supervisión de Cajas y Cierre Forzado (PMC-16, PMC-30, PMC-32)](#5-módulo-4-supervisión-de-cajas-y-cierre-forzado)
   - [4.1 Diagrama de Flujo: Panel de Supervisión y Cierre Administrativo](#41-diagrama-de-flujo-panel-de-supervisión-y-cierre-administrativo)
   - [4.2 Diagrama de Secuencia: Auditoría y Cierre Forzado de Turno](#42-diagrama-de-secuencia-auditoría-y-cierre-forzado-de-turno)
6. [Módulo 5: Formatos Post-Venta: Tickets Térmicos y Exportación Contable (PMC-14, PMC-15)](#6-módulo-5-formatos-post-venta-tickets-térmicos-y-exportación-contable)
   - [5.1 Diagrama de Secuencia: Generación e Impresión de Ticket Térmico (80 mm)](#51-diagrama-de-secuencia-generación-e-impresión-de-ticket-térmico-80-mm)
   - [5.2 Diagrama de Secuencia: Exportación Contable Masiva en Streaming CSV](#52-diagrama-de-secuencia-exportación-contable-masiva-en-streaming-csv)
7. [Matriz Comparativa: Cambios Respecto a los Diagramas Originales](#7-matriz-comparativa-cambios-respecto-a-los-diagramas-originales)

---

## 1. Arquitectura Operativa Global del Microservicio

Este diagrama ilustra el flujo integral del microservicio adaptado a las decisiones técnicas de Jira: persistencia dual en memoria distribuida (**Redis**) y base de datos relacional (**SQL Server / PostgreSQL**), comunicación asíncrona mediante **MassTransit** y desacoplamiento de pasarelas bancarias y fiscalización para el MVP.

```mermaid
flowchart TD
    %% Estilos de Nodos
    classDef external fill:#f9f9f9,stroke:#666,stroke-width:1px,color:#333;
    classDef broker fill:#fff3e0,stroke:#e65100,stroke-width:2px,color:#e65100;
    classDef memory fill:#e8f5e9,stroke:#2e7d32,stroke-width:2px,color:#2e7d32;
    classDef logic fill:#e1f5fe,stroke:#0277bd,stroke-width:2px,color:#0277bd;
    classDef storage fill:#ede7f6,stroke:#4527a0,stroke-width:2px,color:#4527a0;
    classDef client fill:#fce4ec,stroke:#c2185b,stroke-width:2px,color:#c2185b;

    subgraph Salon_Comandas["Salón y Comandas"]
        Orders["Microservicio de Órdenes"]:::external
    end

    subgraph Bus_Eventos["Bus de Eventos MassTransit"]
        EvtReady["Evento Asíncrono: DinningSessionReadyEvent"]:::broker
        EvtCancel["Evento Asíncrono: DinningSessionCancelledEvent"]:::broker
        EvtPaid["Evento Asíncrono: PaymentCompletedEvent"]:::broker
    end

    subgraph Memoria_Redis["Capa de Memoria Distribuida Redis"]
        BillCache[("Snapshot Pre-cuenta: payments:bill:id")]:::memory
        IndexPending[("Índice de Pendientes: payments:pending_bills_index")]:::memory
    end

    subgraph Microservicio_Pagos["Microservicio de Cuenta y Pagos"]
        ConsumerSync["Consumidor MassTransit: Sincronización Pre-cuentas"]:::logic
        ConsumerCleanup["Consumidor MassTransit: Limpieza por Anulación"]:::logic
        ApiGateway["Kong API Gateway / Extracción JWT Claims"]:::logic
        PaymentService["Servicio de Cobro y Validación Financiera"]:::logic
        ShiftService["Servicio de Turnos y Arqueo"]:::logic
        ExportService["Servicio de Streaming CSV"]:::logic
    end

    subgraph Persistencia_SQL["Persistencia Transaccional ACID"]
        DBSQL[("Base de Datos SQL: TurnoCaja, TransaccionPago, Detalle, Desgloses")]:::storage
    end

    subgraph Puntos_Venta["Puntos de Venta y Supervisión"]
        UIPending["UI: Bandeja de Caja - Polling 10s"]:::client
        UICobro["UI: Pantalla de Cobro y Pagos Mixtos"]:::client
        UITicket["UI: Impresión Ticket 80mm media print"]:::client
        UIShift["UI: Turno, Arqueo y Cierre"]:::client
        UIAdmin["UI: Supervisión y Cierre Forzado"]:::client
        UIExport["UI: Historial y Exportación CSV"]:::client
    end

    %% Flujos de Comandas hacia Redis
    Orders -->|"Comanda lista para pago"| EvtReady
    Orders -->|"Mesa cancelada en salón"| EvtCancel
    EvtReady --> ConsumerSync
    EvtCancel --> ConsumerCleanup
    ConsumerSync -->|"Guarda snapshot TTL 24h"| BillCache
    ConsumerSync -->|"Agrega ID a Set"| IndexPending
    ConsumerCleanup -->|"Elimina clave"| BillCache
    ConsumerCleanup -->|"Remueve ID de Set"| IndexPending

    %% Flujos de la Bandeja de Caja
    UIPending -->|"GET /api/v1/payments/pendings"| ApiGateway
    ApiGateway --> ShiftService
    ShiftService -->|"Consulta rápida sub-20ms"| IndexPending
    ShiftService -->|"Lee snapshots"| BillCache

    %% Flujos de Cobro
    UICobro -->|"POST /api/v1/payments/charge"| ApiGateway
    ApiGateway --> PaymentService
    PaymentService -->|"Verifica turno abierto y saldos"| DBSQL
    PaymentService -->|"Lee snapshot de platillos"| BillCache
    PaymentService -->|"Inserta transacción, desgloses y folio"| DBSQL
    PaymentService -->|"Elimina pre-cuenta pagada"| BillCache
    PaymentService -->|"Remueve de índice"| IndexPending
    PaymentService -->|"Publica evento Outbox"| EvtPaid

    %% Flujos Post-Cobro y Turnos
    PaymentService -->|"201 Created"| UICobro
    UICobro -->|"GET /payments/folio/ticket"| UITicket
    UIShift -->|"Apertura, Arqueo y Cierre"| ShiftService
    UIAdmin -->|"Supervisión y Cierre Forzado"| ShiftService
    ShiftService --> DBSQL
    UIExport -->|"GET /payments/export"| ExportService
    ExportService -->|"IAsyncEnumerable streaming"| DBSQL
```

---

## 2. Módulo 1: Sincronización Asíncrona y Bandeja de Caja

### 1.1 Diagrama de Flujo: Ingesta y Limpieza de Pre-Cuentas
Este diagrama detalla cómo el microservicio procesa eventos provenientes de Órdenes (PMC-5 y PMC-28) asegurando idempotencia y manteniendo actualizado el índice de pre-cuentas en Redis.

```mermaid
flowchart TD
    StartEvent(["Llega Mensaje desde RabbitMQ / MassTransit"]):::broker --> CheckType{"¿Tipo de Evento?"}

    %% Flujo DinningSessionReadyEvent
    CheckType -->|"DinningSessionReadyEvent"| ValidateReady{"¿Payload Válido?<br/>totalEsperado > 0 y items no vacío"}
    ValidateReady -->|"No"| SendDLQ["Enviar a DLQ: payments_dinning_ready_error"]:::logic --> EndDLQ(["Fin: Mensaje en Cuarentena"])
    ValidateReady -->|"Sí"| CheckIdempotent{"¿EventId ya procesado en Inbox?"}
    CheckIdempotent -->|"Sí"| AckIgnore["Hacer ACK y descartar duplicado"]:::logic --> EndAck(["Fin: ACK"])
    CheckIdempotent -->|"No"| CheckTimestamp{"¿Existe snapshot previo y timestamp es menor?"}
    CheckTimestamp -->|"Sí"| DropOld["Descartar por timestamp obsoleto y hacer ACK"]:::logic --> EndAck
    CheckTimestamp -->|"No"| SaveRedis["Guardar snapshot en Redis: payments:bill:id<br/>TTL 24h, incluye table_name, waiter_name, folio"]:::memory
    SaveRedis --> AddIndex["Agregar dinningSessionId a Set:<br/>payments:pending_bills_index"]:::memory
    AddIndex --> AckReady["Confirmar mensaje ACK a RabbitMQ"]:::logic --> EndReady(["Fin: Pre-cuenta Lista"])

    %% Flujo DinningSessionCancelledEvent
    CheckType -->|"DinningSessionCancelledEvent"| CheckCancelExists{"¿Existe snapshot en Redis?"}
    CheckCancelExists -->|"No"| AckCancelIgnore["Hacer ACK silencioso"]:::logic --> EndAck
    CheckCancelExists -->|"Sí"| DeleteSnapshot["Eliminar clave: payments:bill:id"]:::memory
    DeleteSnapshot --> RemoveIndex["Remover id de Set:<br/>payments:pending_bills_index"]:::memory
    RemoveIndex --> AckCancel["Confirmar mensaje ACK a RabbitMQ"]:::logic --> EndCancel(["Fin: Pre-cuenta Removida"])

    classDef broker fill:#fff3e0,stroke:#e65100,stroke-width:2px,color:#e65100;
    classDef logic fill:#e1f5fe,stroke:#0277bd,stroke-width:2px,color:#0277bd;
    classDef memory fill:#e8f5e9,stroke:#2e7d32,stroke-width:2px,color:#2e7d32;
```

---

### 1.2 Diagrama de Secuencia: Sincronización y Polling de Pre-Cuentas
Ilustra la arquitectura reactiva basada en Polling periódico acordada con el Product Owner (PMC-26 y PMC-27) en lugar de conexiones persistentes por WebSockets.

```mermaid
sequenceDiagram
    autonumber
    actor Cajero as Cajero (Frontend)
    participant UI as Bandeja de Caja (UI)
    participant Gateway as API Gateway (Kong)
    participant PaymentsAPI as Controlador Payments
    participant Redis as Caché Redis
    participant Broker as MassTransit / Bus
    participant Orders as Microservicio Órdenes

    %% Evento inicial
    Orders->>Broker: Publicar DinningSessionReadyEvent (totales, platillos, mesa)
    Broker->>PaymentsAPI: Consumidor recibe DinningSessionReadyEvent
    PaymentsAPI->>Redis: SADD payments:pending_bills_index dinningSessionId
    PaymentsAPI->>Redis: SET payments:bill:id (JSON, TTL 24h)
    PaymentsAPI-->>Broker: ACK Mensaje procesado

    %% Ciclo de Polling del Frontend
    loop Polling cada 10 segundos (Pestaña activa)
        Cajero->>UI: Visualiza Bandeja de Cuentas
        UI->>Gateway: GET /api/v1/payments/pendings (Bearer JWT)
        Gateway->>PaymentsAPI: Proxy Request (Claims validados)
        PaymentsAPI->>Redis: SMEMBERS payments:pending_bills_index
        Redis-->>PaymentsAPI: Lista de IDs de sesiones activas
        PaymentsAPI->>Redis: MGET payments:bill:id1, payments:bill:id2...
        Redis-->>PaymentsAPI: Payloads de snapshots
        PaymentsAPI-->>Gateway: 200 OK (Lista de cuentas pendientes)
        Gateway-->>UI: 200 OK JSON
        UI->>UI: Renderiza tarjetas de mesas por cobrar
    end

    %% Anulación de cuenta en salón
    Note over Orders,PaymentsAPI: Caso de Anulación de Orden en Salón
    Orders->>Broker: Publicar DinningSessionCancelledEvent
    Broker->>PaymentsAPI: Consumidor recibe DinningSessionCancelledEvent
    PaymentsAPI->>Redis: DEL payments:bill:id
    PaymentsAPI->>Redis: SREM payments:pending_bills_index id
    PaymentsAPI-->>Broker: ACK Mensaje procesado

    %% Siguiente ciclo de polling refleja la eliminación
    UI->>Gateway: GET /api/v1/payments/pendings
    Gateway->>PaymentsAPI: Proxy Request
    PaymentsAPI->>Redis: Consulta índices activos
    Redis-->>PaymentsAPI: Lista sin la cuenta anulada
    PaymentsAPI-->>UI: 200 OK (Lista actualizada)
    UI->>UI: Remueve visualmente la mesa cancelada
```

---

## 3. Módulo 2: Procesamiento Transaccional de Cobros y Pagos Mixtos

### 2.1 Diagrama de Flujo: Lógica de Cobro, Validación Financiera y Cálculo de Cambio
Representa las validaciones de negocio estrictas acordadas en PMC-6, PMC-7 y PMC-13 para cobros con un solo método o combinados, reglas de tarjetas, referencias de terminal y persistencia del ingreso real.

```mermaid
flowchart TD
    StartCobro(["Inicio: Cajero envía petición de cobro"]):::client --> ValidateTurno{"¿Cajero tiene TurnoCaja ABIERTO?"}
    ValidateTurno -->|"No"| ErrTurno["HTTP 403: CASH_SHIFT_NOT_OPEN<br/>Debe abrir caja antes de cobrar"]:::error
    ValidateTurno -->|"Sí"| GetPrecuenta{"¿Existe Pre-cuenta en Redis?"}
    GetPrecuenta -->|"No"| ErrNotFound["HTTP 404: PENDING_BILL_NOT_FOUND<br/>Pre-cuenta no disponible o ya pagada"]:::error
    GetPrecuenta -->|"Sí"| CalcSaldo["Calcular Saldo Total:<br/>Saldo = Subtotal + Impuestos + Propina"]:::logic

    CalcSaldo --> CheckCardMethods{"¿Incluye pagos con Tarjeta?"}
    CheckCardMethods -->|"Sí"| CheckCardRef{"¿Métodos con tarjeta incluyen<br/>authorization_reference?"}
    CheckCardRef -->|"No"| ErrCardRef["HTTP 422: AUTHORIZATION_REFERENCE_REQUIRED<br/>Referencia obligatoria para tarjetas"]:::error
    CheckCardRef -->|"Sí"| CheckCardExceed{"¿Suma de Tarjetas > Saldo Total?"}
    CheckCardExceed -->|"Sí"| ErrCardExceed["HTTP 422: CARD_AMOUNT_EXCEEDS_BALANCE<br/>Pagos con tarjeta no pueden exceder total"]:::error
    CheckCardExceed -->|"No"| CheckCardFullAndCash{"¿Tarjeta cubre el total y<br/>se envía Efectivo > 0?"}
    CheckCardFullAndCash -->|"Sí"| ErrCardFull["HTTP 422: CARD_COVERS_FULL_AMOUNT<br/>Tarjeta cubre total; no se admite efectivo"]:::error
    CheckCardFullAndCash -->|"No"| CheckSumTotal
    CheckCardMethods -->|"No"| CheckSumTotal

    CheckSumTotal{"¿Suma de Pagos Total >= Saldo Total?"}
    CheckSumTotal -->|"No"| ErrFunds["HTTP 422: INSUFFICIENT_FUNDS<br/>Retorna déficit restante requerido"]:::error
    CheckSumTotal -->|"Sí"| CalcChange["Calcular Cambio Vuelto:<br/>change_given = Efectivo - saldo_restante"]:::logic

    CalcChange --> AdjustCashBreakdown["Ajuste de Desglose de Efectivo:<br/>Monto persistido = min efectivo, saldo_restante<br/>Garantiza sum Desgloses == Total Transacción"]:::logic

    AdjustCashBreakdown --> GenFolio["Generar Folio Único Diario Atómico:<br/>TX-yyyyMMdd-000001"]:::logic

    GenFolio --> SqlBegin["Iniciar Transacción SQL IDbContextTransaction"]:::storage
    SqlBegin --> SqlInsert["1. Insertar TransaccionPago COMPLETADO<br/>2. Insertar DetalleTransaccion snapshot<br/>3. Insertar DesglosePago montos netos<br/>4. Registrar Outbox: PaymentCompletedEvent"]:::storage
    SqlInsert --> SqlCommit{"¿COMMIT Exitoso?"}
    SqlCommit -->|"No / Excepción"| SqlRollback["ROLLBACK Transacción Completa"]:::error --> Err500["HTTP 500: INTERNAL_SERVER_ERROR"]:::error
    SqlCommit -->|"Sí"| CleanRedis["Eliminar pre-cuenta y desindexar de Redis"]:::memory
    CleanRedis --> EmitEvent["Disparar evento asíncrono PaymentCompletedEvent"]:::broker
    EmitEvent --> Resp201["HTTP 201 Created Folio, Total, change_given"]:::client --> EndCobro(["Fin del Cobro"])

    classDef client fill:#fce4ec,stroke:#c2185b,stroke-width:2px,color:#c2185b;
    classDef logic fill:#e1f5fe,stroke:#0277bd,stroke-width:2px,color:#0277bd;
    classDef memory fill:#e8f5e9,stroke:#2e7d32,stroke-width:2px,color:#2e7d32;
    classDef storage fill:#ede7f6,stroke:#4527a0,stroke-width:2px,color:#4527a0;
    classDef broker fill:#fff3e0,stroke:#e65100,stroke-width:2px,color:#e65100;
    classDef error fill:#ffebee,stroke:#c62828,stroke-width:2px,color:#c62828;
```

---

### 2.2 Diagrama de Secuencia: Transacción Atómica de Cobro y Evento Outbox
Ilustra el aislamiento transaccional estricto, la consulta de datos en memoria y la publicación atómica hacia el broker de eventos.

```mermaid
sequenceDiagram
    autonumber
    actor Cajero as Cajero
    participant UI as Pantalla de Cobro (FE)
    participant Gateway as API Gateway (Kong)
    participant CobroService as Servicio de Cobro
    participant CalcDomain as Motor Financiero (Dominio)
    participant Redis as Caché Redis
    participant SQL as SQL Server / Postgres (EF Core)
    participant Bus as MassTransit (RabbitMQ)

    Cajero->>UI: Ingresa desglose de pagos y propina
    UI->>Gateway: POST /api/v1/payments/charge (Bearer JWT, payload)
    Gateway->>CobroService: Valida claims y delega request
    CobroService->>SQL: SELECT TurnoCaja WHERE CashierId == JWT.Id AND Status == 'ABIERTO'
    alt Sin Turno Abierto
        SQL-->>CobroService: null
        CobroService-->>UI: 403 Forbidden (CASH_SHIFT_NOT_OPEN)
    end

    CobroService->>Redis: GET payments:bill:dinningSessionId
    Redis-->>CobroService: Snapshot de cuenta (Platillos, Subtotal, IVA)

    CobroService->>CalcDomain: Evaluar reglas financieras (Saldos, Tarjetas, Cambio)
    alt Regla Violada (exceso tarjeta / falta referencia / fondos insuficientes)
        CalcDomain-->>CobroService: Error de Dominio
        CobroService-->>UI: 422 Unprocessable Entity (Error de validación)
    end
    CalcDomain-->>CobroService: Cálculos validados (change_given, desgloses netos)

    CobroService->>SQL: BEGIN TRANSACTION (IDbContextTransaction)
    CobroService->>SQL: Obtener siguiente correlativo diario para folio
    CobroService->>SQL: INSERT TransaccionPago (TX-20261108-000001, COMPLETADO, change_given)
    CobroService->>SQL: INSERT DetalleTransaccion (Platillos snapshot)
    CobroService->>SQL: INSERT DesglosePago (Efectivo ajustado, Tarjetas con ref)
    CobroService->>SQL: INSERT OutboxMessage (PaymentCompletedEvent)
    CobroService->>SQL: COMMIT TRANSACTION

    CobroService->>Redis: DEL payments:bill:id & SREM payments:pending_bills_index id
    CobroService->>Bus: Publicar PaymentCompletedEvent (hacia Órdenes y Cocina)
    CobroService-->>Gateway: 201 Created (Folio, Total, change_given)
    Gateway-->>UI: 201 Created JSON
    UI->>UI: Muestra confirmación de pago y cambio al cajero
```

---

## 4. Módulo 3: Ciclo de Vida del Turno de Caja y Arqueo

### 3.1 Diagrama de Flujo: Apertura, Arqueo en Tiempo Real y Cierre Operativo
Muestra los estados del turno de caja del operador (`ABIERTO`, `CERRADO`, `DESCUADRADO`), la prevención de aperturas simultáneas y la comparación ciega de efectivo físico (PMC-8, PMC-9, PMC-29).

```mermaid
flowchart TD
    StartOperacion(["Inicio de Jornada del Cajero"]):::client --> CheckTurnoMe{"GET /cash-shifts/me<br/>¿Tiene turno activo?"}
    CheckTurnoMe -->|"Sí (200 OK)"| OperarCaja["Turno ABIERTO: Habilitado para Cobrar"]:::logic
    CheckTurnoMe -->|"No (404 Not Found)"| SolicitarApertura["UI Bloqueada: Requiere Apertura"]:::client

    SolicitarApertura --> PostOpen["POST /cash-shifts/open<br/>Body: opening_float"]:::client
    PostOpen --> ValidaFondo{"¿Fondo Inicial >= 0?"}
    ValidaFondo -->|"No"| ErrFondo["HTTP 400: INVALID_AMOUNT"]:::error
    ValidaFondo -->|"Sí"| CheckDoubleOpen{"¿Ya tiene turno ABIERTO?"}
    CheckDoubleOpen -->|"Sí"| Err409["HTTP 409: CASH_SHIFT_ALREADY_OPEN"]:::error
    CheckDoubleOpen -->|"No"| CreateTurno["Crear TurnoCaja en SQL<br/>Estado: ABIERTO<br/>Registrar cashier_name desde JWT"]:::storage --> OperarCaja

    OperarCaja --> DuranteTurno{"Operación Durante la Jornada"}
    DuranteTurno -->|"Cobrar Cuentas"| ChargeOps["Procesar Transacciones RF-04"]:::logic
    DuranteTurno -->|"Arqueo en Tiempo Real"| GetSummary["GET /cash-shifts/summary<br/>Lectura no bloqueante RCSI"]:::client
    GetSummary --> CalcExpected["Sistema calcula en SQL:<br/>expected_cash = fondo + efectivo neto<br/>card_total, card_count, retained_tips"]:::logic
    CalcExpected --> ReturnSummary["200 OK: Métricas de Arqueo para el Cajero"]:::client

    DuranteTurno -->|"Fin de Jornada"| SolicitarCierre["Cajero solicita Cierre de Turno"]:::client
    SolicitarCierre --> ConteoFisico["Cajero realiza conteo físico de dinero en gaveta"]:::client
    ConteoFisico --> PostClose["POST /cash-shifts/close<br/>Body: declared_cash, reason"]:::client
    PostClose --> CompararEfectivo{"¿declared_cash == expected_cash?"}

    CompararEfectivo -->|"Sí (Cuadre Exacto)"| SetCerrado["Actualizar TurnoCaja en SQL:<br/>Estado: CERRADO<br/>difference = 0.00<br/>closed_at = UtcNow"]:::storage
    CompararEfectivo -->|"No (Diferencia)"| SetDescuadrado["Actualizar TurnoCaja en SQL:<br/>Estado: DESCUADRADO<br/>difference = declared - expected<br/>closed_reason = reason<br/>closed_at = UtcNow"]:::error

    SetCerrado --> FinTurno(["Fin: Caja Cerrada Exitosamente"]):::logic
    SetDescuadrado --> FinTurno

    classDef client fill:#fce4ec,stroke:#c2185b,stroke-width:2px,color:#c2185b;
    classDef logic fill:#e1f5fe,stroke:#0277bd,stroke-width:2px,color:#0277bd;
    classDef storage fill:#ede7f6,stroke:#4527a0,stroke-width:2px,color:#4527a0;
    classDef error fill:#ffebee,stroke:#c62828,stroke-width:2px,color:#c62828;
```

---

### 3.2 Diagrama de Secuencia: Arqueo sin Bloqueos y Cierre de Caja
Ilustra el uso de Read Committed Snapshot Isolation (RCSI) para permitir que los reportes de arqueo no detengan los cobros concurrentes en otras cajas.

```mermaid
sequenceDiagram
    autonumber
    actor Cajero as Cajero
    participant UI as Pantalla de Arqueo / Cierre
    participant Gateway as API Gateway (Kong)
    participant ShiftAPI as Controlador CashShifts
    participant SQL as SQL Server (Snapshot Isolation)

    %% Arqueo en Tiempo Real
    Cajero->>UI: Solicita Arqueo Parcial / Pre-corte
    UI->>Gateway: GET /api/v1/payments/cash-shifts/summary (Bearer JWT)
    Gateway->>ShiftAPI: Identifica CajeroId por Claims
    ShiftAPI->>SQL: SELECT con AsNoTracking() sobre TurnoCaja y Transacciones
    Note over ShiftAPI,SQL: RCSI: Lectura de versión de fila sin Shared Locks.<br/>No bloquea escrituras concurrentes.
    SQL-->>ShiftAPI: Sumatorias de efectivo neto, tarjetas y propinas
    ShiftAPI-->>Gateway: 200 OK (expected_cash, card_total, card_count, retained_tips)
    Gateway-->>UI: 200 OK JSON
    UI->>UI: Despliega totales para verificación

    %% Cierre de Caja
    Cajero->>UI: Declara efectivo físico contado y confirma cierre
    UI->>Gateway: POST /api/v1/payments/cash-shifts/close (declared_cash, reason)
    Gateway->>ShiftAPI: Proxy Request
    ShiftAPI->>SQL: Re-calcula expected_cash final del turno
    SQL-->>ShiftAPI: Total final esperado
    alt Cuadre Exacto
        ShiftAPI->>SQL: UPDATE TurnoCaja SET Status='CERRADO', ClosedAt=NOW
    else Descuadre (Sobrante / Faltante)
        ShiftAPI->>SQL: UPDATE TurnoCaja SET Status='DESCUADRADO', Difference=diff, ClosedAt=NOW
    end
    SQL-->>ShiftAPI: Filas actualizadas
    ShiftAPI-->>Gateway: 200 OK (Status CERRADO o DESCUADRADO, difference)
    Gateway-->>UI: 200 OK JSON
    UI->>UI: Emite comprobante de corte de turno e inhabilita caja
```

---

## 5. Módulo 4: Supervisión de Cajas y Cierre Forzado

### 5.1 Diagrama de Flujo: Panel de Supervisión y Cierre Administrativo
Describe las capacidades exclusivas de roles `Administrador` y `Gerente` para auditar cualquier caja y forzar el cierre administrativo en caso de abandono o emergencia (PMC-16, PMC-30, PMC-32).

```mermaid
flowchart TD
    StartAdmin(["Administrador / Gerente accede a Supervisión"]):::client --> CheckRole{"¿Rol == Administrador o Gerente?"}
    CheckRole -->|"No"| Err403["HTTP 403: FORBIDDEN_ROLE"]:::error
    CheckRole -->|"Sí"| GetOpenShifts["GET /cash-shifts/open<br/>Lista todas las cajas operativas"]:::client

    GetOpenShifts --> SelectShift["Selecciona un turno específico por ID"]:::client
    SelectShift --> InspectSummary["GET /cash-shifts/:id/summary<br/>Audita el arqueo del turno ajeno"]:::client

    InspectSummary --> EvaluaCierre{"¿Requiere Cierre Forzado?<br/>ej. abandono de puesto o corte administrativo"}
    EvaluaCierre -->|"No"| KeepOpen["Mantiene caja operando normalmente"]:::logic
    EvaluaCierre -->|"Sí"| PostForceClose["POST /cash-shifts/:id/force-close<br/>Body: declared_cash, force_close_reason"]:::client

    PostForceClose --> ValidaReason{"¿force_close_reason >= 10 caracteres?"}
    ValidaReason -->|"No"| ErrReason["HTTP 400: Validacion FluentValidation fallida"]:::error
    ValidaReason -->|"Sí"| CheckTurnoStatus{"¿Estado del Turno?"}
    CheckTurnoStatus -->|"No existe"| Err404["HTTP 404: CASH_SHIFT_NOT_FOUND"]:::error
    CheckTurnoStatus -->|"Ya CERRADO o DESCUADRADO"| Err409["HTTP 409: CASH_SHIFT_CLOSED"]:::error
    CheckTurnoStatus -->|"ABIERTO"| ExecForceClose["Actualizar TurnoCaja en SQL:<br/>Status = CERRADO o DESCUADRADO<br/>closed_by_admin_id = JWT.Id<br/>force_close_reason = motivo<br/>closed_at = UtcNow"]:::storage

    ExecForceClose --> ReturnAdmin200["200 OK: Turno Finalizado Forzadamente con Rastro de Auditoría"]:::client

    classDef client fill:#fce4ec,stroke:#c2185b,stroke-width:2px,color:#c2185b;
    classDef logic fill:#e1f5fe,stroke:#0277bd,stroke-width:2px,color:#0277bd;
    classDef storage fill:#ede7f6,stroke:#4527a0,stroke-width:2px,color:#4527a0;
    classDef error fill:#ffebee,stroke:#c62828,stroke-width:2px,color:#c62828;
```

---

### 5.2 Diagrama de Secuencia: Auditoría y Cierre Forzado de Turno

```mermaid
sequenceDiagram
    autonumber
    actor Admin as Administrador / Gerente
    participant UI as Panel de Supervisión (FE)
    participant Gateway as API Gateway (Kong)
    participant ShiftAPI as Controlador CashShifts
    participant SQL as Base de Datos SQL

    Admin->>UI: Ingresa a Panel de Supervisión de Cajas
    UI->>Gateway: GET /api/v1/payments/cash-shifts/open (JWT Admin)
    Gateway->>ShiftAPI: Autoriza rol Administrador/Gerente
    ShiftAPI->>SQL: SELECT Turnos WHERE Status == 'ABIERTO'
    SQL-->>ShiftAPI: Lista de turnos con cajeros y horas de inicio
    ShiftAPI-->>UI: 200 OK (Lista de turnos abiertos)

    Admin->>UI: Selecciona caja con irregularidad para auditar
    UI->>Gateway: GET /api/v1/payments/cash-shifts/:cash_shift_id/summary
    Gateway->>ShiftAPI: Proxy Request
    ShiftAPI->>SQL: Calcula balance financiero del turno indicado
    SQL-->>ShiftAPI: expected_cash, fondos y totales
    ShiftAPI-->>UI: 200 OK (Métricas de arqueo del turno)

    Admin->>UI: Ingresa conteo físico y justificación de cierre forzado
    UI->>Gateway: POST /api/v1/payments/cash-shifts/:id/force-close
    Note over UI,Gateway: Payload: declared_cash: 850.00, force_close_reason: 'Cajero ausente...'
    Gateway->>ShiftAPI: Proxy Request
    ShiftAPI->>ShiftAPI: Valida longitud mínima de 10 caracteres en motivo
    ShiftAPI->>SQL: BEGIN TRANSACTION
    ShiftAPI->>SQL: UPDATE TurnoCaja SET Status='DESCUADRADO', ClosedByAdminId=JWT.Id, ForceCloseReason=reason, ClosedAt=NOW
    ShiftAPI->>SQL: COMMIT TRANSACTION
    SQL-->>ShiftAPI: Confirmación de actualización
    ShiftAPI-->>Gateway: 200 OK (Cierre forzado registrado exitosamente)
    Gateway-->>UI: 200 OK JSON
    UI->>UI: Refresca lista eliminando el turno cerrado
```

---

## 6. Módulo 5: Formatos Post-Venta: Tickets Térmicos y Exportación Contable

### 6.1 Diagrama de Secuencia: Generación e Impresión de Ticket Térmico (80 mm)
Detalla la obtención de datos inmutables para el renderizado e impresión desde el navegador mediante `@media print` en lugar de binarios propietarios ESC/POS (PMC-14, PMC-21).

```mermaid
sequenceDiagram
    autonumber
    actor Cajero as Cajero / Cliente
    participant UI as Modal de Cobro / Historial (FE)
    participant Gateway as API Gateway (Kong)
    participant TicketAPI as Controlador Payments
    participant SQL as Base de Datos SQL
    participant Impresora as Impresora Térmica 80mm

    Cajero->>UI: Solicita imprimir o reimprimir comprobante de pago
    UI->>Gateway: GET /api/v1/payments/:folio/ticket (Bearer JWT)
    Gateway->>TicketAPI: Proxy Request
    TicketAPI->>SQL: SELECT TransaccionPago JOIN Detalle JOIN Desgloses WHERE Folio == folio
    alt Folio Inexistente
        SQL-->>TicketAPI: null
        TicketAPI-->>UI: 404 Not Found (TRANSACTION_NOT_FOUND)
    end
    SQL-->>TicketAPI: Datos completos (Mesa, Cajero, Platillos, Totales, Referencias)
    TicketAPI->>TicketAPI: Mapea DTO con restaurant_name institucional y paid_at inmutable
    TicketAPI-->>Gateway: 200 OK JSON (DTO TicketImpresion)
    Gateway-->>UI: 200 OK JSON

    UI->>UI: Renderiza componente visual formateado a 80mm
    UI->>UI: Dispara window.print() aplicando estilos CSS @media print
    UI->>Impresora: Envía trabajo de impresión al driver del sistema operativo
    Impresora-->>Cajero: Emite ticket físico térmico impreso
```

---

### 6.2 Diagrama de Secuencia: Exportación Contable Masiva en Streaming CSV
Demuestra la arquitectura de streaming asíncrono con memoria plana $O(1)$ para descargas de hasta 31 días acordada en PMC-15.

```mermaid
sequenceDiagram
    autonumber
    actor Admin as Administrador
    participant UI as Vista Historial de Ventas (FE)
    participant Gateway as API Gateway (Kong)
    participant ExportAPI as Controlador Payments
    participant SQL as SQL Server (IAsyncEnumerable)

    Admin->>UI: Selecciona rango de fechas y presiona 'Descargar CSV'
    UI->>Gateway: GET /api/v1/payments/export?start_date=2026-11-01T00:00:00Z&end_date=2026-11-20T00:00:00Z
    Gateway->>ExportAPI: Verifica rol Administrador (JWT Claims)
    alt Rol no es Administrador
        ExportAPI-->>UI: 403 Forbidden (FORBIDDEN_ROLE)
    end

    ExportAPI->>ExportAPI: Valida ventana semiabierta [start, end) y rango <= 31 días
    alt Rango > 31 días o Fechas Inválidas
        ExportAPI-->>UI: 400 Bad Request (INVALID_DATE_RANGE)
    end

    ExportAPI->>SQL: Abre cursor SQL con AsNoTracking() e índice IX_Transactions_PaidAt_Folio
    ExportAPI-->>Gateway: HTTP 200 OK Headers:
    Note over ExportAPI,Gateway: Headers HTTP 200:<br/>Content-Type: text/csv (UTF-8)<br/>Transfer-Encoding: chunked<br/>Content-Disposition: attachment - reporte_ventas.csv
    Gateway-->>UI: 200 OK (Inicio de descarga de archivo en navegador)

    ExportAPI->>Gateway: Escribe encabezados CSV (Folio,Fecha,Subtotal,Impuestos,Propina,Total,Efectivo,Tarjetas)
    loop Lectura en Chunks de 500 filas
        SQL-->>ExportAPI: Stream de siguientes 500 registros
        ExportAPI->>ExportAPI: Formatea filas a estándar RFC 4180
        ExportAPI->>Gateway: Escribe bloque directamente a Response.Body
        Gateway->>UI: Transfiere chunk de datos al archivo descargado
    end

    ExportAPI->>Gateway: Cierra stream HTTP (EOF)
    Gateway-->>UI: Descarga finalizada exitosamente
    Note over ExportAPI: Consumo de memoria RAM constante O(1) independientemente del número de filas.
```

---

## 7. Matriz Comparativa: Cambios Respecto a los Diagramas Originales

| Diagrama Original (.png) | Limitación o Discrepancia del Diagrama Anterior | Corrección y Enfoque en los Nuevos Diagramas |
| :--- | :--- | :--- |
| **`Payment microservice diagram flow operational.png`** | Incluía integración con **PAC SAT (CFDI)** y tablas como `CorteCajaDiario` que no existen en el modelo de base de datos final. Además asumía WebSockets para la bandeja de caja. | **Actualizado en Sección 1:** Se elimina el PAC SAT (post-MVP), se unifica el control de caja en `TurnoCaja` (`CashShift`), se añade **Redis** como caché de pre-cuentas y se adopta **Polling HTTP** cada 10 s. |
| **`Payment microservice sequence diagram.png`** | Contemplaba llamadas de autorización síncrona a "Pasarela Externa" que bloqueaban el backend, así como rollbacks en caso de fallo de red bancario. | **Actualizado en Sección 3.2:** Se elimina la llamada HTTP a pasarelas bancarias. El cobro es local y se valida obligatoriamente la `authorization_reference` física. Se incluye el Outbox Pattern para MassTransit. |
| **`Flujo de Cobro y Pagos Mixtos.png`** | No especificaba el algoritmo de cálculo de cambio ni las restricciones cuando la tarjeta excede el total o cubre el saldo completo. | **Actualizado en Sección 3.1:** Se modela el cálculo exacto de `change_given`, las reglas `CARD_AMOUNT_EXCEEDS_BALANCE` y `CARD_COVERS_FULL_AMOUNT`, y el ajuste con `Math.Min` para el ingreso real. |
| **`Flujo de Arqueo y Corte de Caja.png`** | Mostraba un "Corte de Caja Diario" separado y no contemplaba la lectura no bloqueante (RCSI) ni la posibilidad de consultar turnos de supervisión. | **Actualizado en Secciones 4.1 y 4.2:** Se formaliza el ciclo de vida sobre `TurnoCaja` con estados `CERRADO` o `DESCUADRADO`, fondo inicial, cálculo de métricas en tiempo real y lectura Snapshot. |
| **`Flujo de Facturación Electrónica (Post-Venta).png`** | Situaba la facturación como un flujo del MVP con endpoints de timbrado inmediatos. | **Aclarado en Arquitectura:** La facturación CFDI 4.0 queda formalmente despriorizada para la fase post-MVP. |
| **`Flujo de Cancelación y Extorno (Excepción).png`** | Detallaba la anulación y extornos en pasarelas bancarias con autorización gerencial. | **Aclarado en Arquitectura:** La cancelación de ventas salió del alcance del MVP por decisión del Product Owner (PMC-12 en *Won't Do*, PR #11 cerrada). |
| *(No existía en diagramas .png)* | No existían diagramas para el **Cierre Forzado** de supervisión, la sincronización de pre-cuentas en Redis, ni la exportación contable en streaming. | **Nuevos Diagramas Creados:** Secciones 1.1, 1.2, 5.1, 5.2 y 6.2 cubren la totalidad de estas Historias de Usuario de Jira (PMC-5, PMC-15, PMC-16, PMC-26, PMC-27, PMC-28, PMC-30, PMC-32). |
