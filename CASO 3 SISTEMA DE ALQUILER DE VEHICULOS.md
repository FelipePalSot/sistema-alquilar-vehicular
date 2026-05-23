# 🚗 Caso 3: Sistema de Alquiler de Vehículos

> Análisis completo del sistema: requisitos, reglas de negocio, arquitectura, eventos, diccionario de dominio y modelo de datos.

---

## 1. 📋 Contexto y Requisitos

> Una empresa de alquiler de autos gestiona la renta de vehículos controlando disponibilidad, plazos, estados y facturación.

### Requisitos Funcionales

| # | Módulo | Funciones |
|---|--------|-----------|
| 1 | 🚙 **Gestión de Vehículos** | Registrar vehículo, consultar disponibles por categoría/fecha, consultar estado |
| 2 | 🔑 **Operación de Alquiler** | Alquilar, validar disponibilidad, devolver, calcular costo, registrar estado al devolver |
| 3 | 🔔 **Notificaciones** | Confirmar alquiler, avisar devolución pendiente, alertar devolución tardía (canal: consola) |

### Reglas de Negocio

| ⚙️ Regla | 💡 Descripción |
|----------|---------------|
| **Exclusividad** | Un vehículo solo puede estar alquilado por UN cliente a la vez |
| **Plazos** | Mínimo 1 día — Máximo 30 días |
| **Horario Laboral** | Solo entre 8:00 AM y 6:00 PM |
| **Límite por Cliente** | Máximo 2 vehículos alquilados simultáneamente |
| **Categorización** | Sedan / SUV / Camioneta — tarifa diaria distinta por categoría |
| **Penalización** | **1.5x** la tarifa diaria por cada día extra de retraso |
| **Combustible** | Devolver con el mismo nivel recibido (tanque lleno) |
| **Daños** | Cargo adicional obligatorio si el vehículo se devuelve dañado |

---

## 2. 🧩 Metodología: "Divide y Vencerás"

| Fase | Nombre | Descripción |
|------|--------|-------------|
| 1️⃣ | **Descomposición** (拆分) | Dividir el problema en 3 sub-espacios: Vehículos, Alquileres, Notificaciones |
| 2️⃣ | **Resolución Individual** (分别求解) | Solucionar cada sub-espacio de forma desacoplada |
| 3️⃣ | **Fusión** (合并解) | Integrar todas las sub-soluciones en un flujo unificado |

> **¿Cómo guiar la división?** Mediante **Domain-Driven Design (DDD)** — Bounded Contexts, Descomposición Funcional o Arquitectura de Microservicios.

---

## 3. ⚡ Eventos del Sistema (Event Storming)

| # | Evento | Descripción |
|---|--------|-------------|
| 1 | `VehículoRegistrado` | Alta de auto en catálogo (marca, modelo, placa, categoría, tarifa) |
| 2 | `EstadoDeVehículoConsultado` | Verificación de disponibilidad o asignación actual |
| 3 | `AlquilerSolicitado` | Intento de reserva — activa validaciones de tiempo y límites |
| 4 | `AlquilerConfirmado` | Reserva aprobada — vehículo bloqueado para otros usuarios |
| 5 | `VehículoDevuelto` | Entrega física — se registra timestamp, combustible y condición |
| 6 | `CostoTotalCalculado` | Procesamiento de tarifa base + recargos (mora, combustible, daños) |
| 7 | `ConfirmaciónDeAlquilerNotificada` | Mensaje de confirmación enviado a consola |
| 8 | `DevoluciónPendienteNotificada` | Aviso preventivo de vencimiento próximo |
| 9 | `DevoluciónTardíaNotificada` | Alerta crítica de retraso — inicia flujo de multas |

---

## 4. 📖 Diccionario de Términos

### A. Penalización por Retraso

| Aspecto | Detalle |
|---------|---------|
| **Definición** | Cargo extra por entregar el vehículo fuera del horario o fecha pactada |
| **Estado: No Aplicada** | Entrega realizada a tiempo |
| **Estado: Aplicada** | Multa calculada e integrada al costo total |
| **Cálculo** | 1.5x tarifa diaria por cada día excedido |
| **Activación** | Automática si la entrega es después de las 6:00 PM o supera el día pactado |

### B. Estado de Devolución

| Estado | Condición | Consecuencia |
|--------|-----------|--------------|
| ✅ `Conforme` | A tiempo, tanque lleno, sin daños | Cierre normal del contrato |
| ⏰ `Tardío` | Fuera del plazo o del horario comercial | Penalización 1.5x por día extra |
| 🔨 `Dañado` | Abolladuras, averías o golpes | Recargo obligatorio por reparación |
| ⛽ `Combustible Incompleto` | Tanque por debajo del nivel original | Recargo por reabastecimiento |

> Al registrar la devolución, el vehículo cambia automáticamente a estado **`Disponible`**.

---

## 5. 🗄️ Modelo de Datos

### Atributos por Entidad

| Entidad | Atributo | Tipo | Descripción |
|---------|----------|------|-------------|
| **Moneda** | Nombre | string | Ej. "Sol Peruano" |
| | Símbolo | string | Ej. "S/" |
| | CódigoISO | string | Ej. "PEN" |
| **Precio** | MontoBase | decimal | Valor antes de impuestos o multas |
| | IdMoneda | string | Clave de asociación con Moneda |
| | EsModificable | boolean | Flexibilidad ante promociones |
| **Marca** | NombreFabricante | string | Ej. "Toyota" |
| | PaísOrigen | string | País de manufactura |
| | CódigoProveedor | string | Identificador interno del catálogo |
| **Modelo** | NombreComercial | string | Ej. "Corolla" |
| | AñoFabricación | int | Año de fabricación |
| | IdMarca | int | FK → Marca |
| **Placa** | NúmeroRegistro | string | Código alfanumérico único oficial |
| | Jurisdicción | string | Región/ciudad de inscripción |
| | FechaVencimientoRevisión | date | Próximo control técnico obligatorio |
| **Alquiler** | FechaHoraInicio | datetime | Retiro del auto (8 AM – 6 PM) |
| | FechaHoraFinPactada | datetime | Límite contractual de devolución |
| | TotalDíasContratados | int | Días pactados (mín. 1, máx. 30) |
| **EstadoVehículo** | NivelCombustible | string | Ej. "Lleno" |
| | CondiciónCarrocería | string | Ej. "Impecable" |
| | IdAlquilerAsociado | int | FK → Contrato de alquiler |

### Relaciones y Cardinalidades

| Origen | Destino | Cardinalidad | Regla |
|--------|---------|:------------:|-------|
| **Cliente** | Alquiler Vehículo | **1 : N** | Máx. 2 alquileres simultáneos por cliente (control por software) |
| **Vehículo** | Alquiler Vehículo | **1 : N** | Solo un cliente activo por vez; historial ilimitado |
| **Categoria** | Vehículo | **1 : N** | Cada vehículo pertenece a una única categoría |
| **Categoria** | Tarifa Diaria | **1 : 1** | Tarifa fija y exclusiva por categoría |
| **Alquiler Vehículo** | Tarifa Diaria | **N : 1** | Múltiples alquileres usan la misma tarifa para calcular base y multas (1.5x) |
