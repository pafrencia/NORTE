# NORTE – OrderManager / Picking Isla  
## Control de Armado por Consolidado y Comprobante  
### Lock persistente con **IdArmado** (compatible SQL Server 2008)

---

## 1. Contexto y problema actual

En la tarea **Picking Isla**, el botón **“Comenzar armado”** inicia un proceso **local**, sin registrar información en la base de datos hasta el cierre final.

El sistema **permite comenzar el armado de varios comprobantes juntos del mismo cliente**, lo que genera que:

- Dos operarios, desde distintas estaciones,
- puedan comenzar el armado del **mismo conjunto de comprobantes**,
- provocando duplicación de trabajo, errores y reprocesos.

---

## 2. Objetivo

Implementar un **mecanismo simple y persistente de control de armado**, que permita:

- Registrar quién comienza un armado y cuándo.
- Agrupar **varios comprobantes** dentro de un mismo armado mediante un **IdArmado**.
- Detectar si un comprobante:
  - está **COMENZADO**, o
  - está **FINALIZADO**.
- Mostrar mensajes claros al operario.
- Permitir **tomar control del armado** (pisar el registro existente) incluso si estaba FINALIZADO.
- Evitar armados duplicados por falta de refresco de pantalla.

El lock **NO debe expirar automáticamente** ni eliminarse al finalizar.

---

## 3. Concepto clave: IdArmado

El **IdArmado** representa una **sesión de armado** iniciada por un operario.

- Un **IdArmado** puede contener **uno o varios comprobantes**.
- Cada comprobante se guarda como **una fila**, pero todas las filas del mismo armado comparten el mismo `IdArmado`.
- Al cancelar o finalizar, se opera **por IdArmado**, afectando todas las filas asociadas.

---

## 4. Estados del armado

El campo **Estado** puede tomar los siguientes valores:

- **COMENZADO** (estado por defecto al crear el registro)
- **FINALIZADO**

---

## 5. Identificación única de cada comprobante

Cada fila queda identificada de forma única por:

- `CodigoConsolidado`
- `NumeroConsolidado`
- `CodigoCliente`
- `Comprobante`

> El control es **por comprobante**, pero la operación de cancelar/finalizar es **por IdArmado**.

---

## 6. Flujo funcional detallado

### 6.1. Al presionar “Comenzar armado”

1. El sistema determina la **lista de comprobantes seleccionados**.
2. Consulta la tabla de control para verificar si **alguno** de esos comprobantes ya existe.

---

### 6.2. No existe ningún registro previo

- El sistema genera un nuevo:
  - `IdArmado = NEWID()`
- Inserta **una fila por cada comprobante**, todas con:
  - el mismo `IdArmado`
  - `FechaInicio = GETDATE()`
  - `Estado = COMENZADO`
  - `Usuario = usuario actual`
- El armado continúa normalmente.

---

### 6.3. Existe al menos un comprobante registrado  
(Estado COMENZADO o FINALIZADO)

El sistema muestra un cartel informativo:

#### a) Si el estado es COMENZADO
> “El comprobante **{Comprobante}** fue comenzado a armar por el usuario **{Usuario}** el **{FechaInicio}**.”

#### b) Si el estado es FINALIZADO
> “El comprobante **{Comprobante}** figura como **FINALIZADO**, armado por el usuario **{Usuario}** el **{FechaInicio}**.”

En ambos casos, el cartel ofrece **dos opciones**:

##### Opción 1: Continuar con el armado (tomar control / rearmar)
- Se eliminan los registros existentes de los comprobantes en conflicto.
- Se genera un **nuevo IdArmado**.
- Se insertan nuevamente **todas las filas** de los comprobantes seleccionados:
  - con el nuevo `IdArmado`
  - `Estado = COMENZADO`
- El usuario actual **toma control total del armado**.

##### Opción 2: Cancelar
- No se modifica ningún registro.
- Se cancela la operación.
- No se inicia el armado.

---

## 7. Cancelación del armado

Si el operario **cancela el armado en medio del proceso**:

```sql
DELETE dbo.Manager_PickingIsla_ControlArmado
WHERE IdArmado = @IdArmado;
```

Esto elimina **todas las filas del armado**, dejando los comprobantes libres.

---

## 8. Finalización del armado

Cuando el operario **finaliza el armado**, justo **antes de imprimir la etiqueta**:

```sql
UPDATE dbo.Manager_PickingIsla_ControlArmado
SET Estado = 'FINALIZADO'
WHERE IdArmado = @IdArmado;
```

- El registro **NO se elimina**.
- El armado queda marcado como finalizado.
- Otro operario verá el estado FINALIZADO y deberá decidir explícitamente si desea rearmar.

---

## 9. Diseño de Base de Datos (compatible SQL Server 2008)

### 9.1. Tabla propuesta

Nombre sugerido:  
`Manager_PickingIsla_ControlArmado`

### Campos

- `IdArmado` UNIQUEIDENTIFIER NOT NULL
- `CodigoConsolidado` NVARCHAR(20) NOT NULL
- `NumeroConsolidado` NUMERIC(18,0) NOT NULL
- `CodigoCliente` NVARCHAR(20) NOT NULL
- `Comprobante` NVARCHAR(50) NOT NULL
- `Usuario` NVARCHAR(50) NOT NULL
- `FechaInicio` DATETIME NOT NULL
- `Estado` NVARCHAR(20) NOT NULL

---

### 9.2. Script SQL sugerido

```sql
CREATE TABLE dbo.Manager_PickingIsla_ControlArmado
(
    IdArmado            UNIQUEIDENTIFIER NOT NULL,
    CodigoConsolidado   NVARCHAR(20) NOT NULL,
    NumeroConsolidado   NUMERIC(18,0) NOT NULL,
    CodigoCliente       NVARCHAR(20) NOT NULL,
    Comprobante         NVARCHAR(50) NOT NULL,
    Usuario             NVARCHAR(50) NOT NULL,
    FechaInicio         DATETIME NOT NULL,
    Estado              NVARCHAR(20) NOT NULL
);

-- Unicidad por comprobante
CREATE UNIQUE INDEX UX_PickingIsla_ControlArmado_Comprobante
ON dbo.Manager_PickingIsla_ControlArmado
(
    CodigoConsolidado,
    NumeroConsolidado,
    CodigoCliente,
    Comprobante
);

-- Índice para operar por IdArmado
CREATE INDEX IX_PickingIsla_ControlArmado_IdArmado
ON dbo.Manager_PickingIsla_ControlArmado (IdArmado);
```

---

## 10. Criterios de aceptación

1. Un armado puede incluir uno o varios comprobantes.
2. Todos los comprobantes del mismo armado comparten el mismo `IdArmado`.
3. Cancelar elimina todas las filas del `IdArmado`.
4. Finalizar marca todas las filas del `IdArmado` como FINALIZADO.
5. Un comprobante FINALIZADO puede rearmarse solo con acción explícita del operario.
6. El control funciona aunque otra estación no refresque pantalla.

---

## 11. Beneficios

- Soporte real para armado múltiple por cliente.
- Cancelación y finalización simples y seguras.
- Total compatibilidad con SQL Server 2008.
- Persistencia clara del estado del pedido.
- Evita armados duplicados y conflictos operativos.

---

**Documento funcional – Distribuidora NORTE**  
OrderManager – Picking Isla
