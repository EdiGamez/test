# Redirección de Inventarios en el Sistema RFR

Este documento explica el funcionamiento de la redirección de inventarios en el sistema RFR, desde su configuración en base de datos hasta su uso en los proyectos que consumen los inventarios.

---

## 1. Estructura General del Sistema

El sistema de inventarios consta de 4 componentes principales:

1. **Base de Datos (USER_RFR.TRFR_INVENTORIES)**
2. **Librería `librfrinventory`**
3. **Proyecto `svcrfrinventory`** (servicio de publicación en Kafka)
4. **Proyecto `unixbatchrfr`** (generación de reportes tipo CSV)

---

## 2. Tabla de Inventarios: USER_RFR.TRFR_INVENTORIES

Esta tabla define los inventarios y sus posibles redirecciones. Las columnas principales son:

| Columna                 | Descripción                                                                 |
|-------------------------|------------------------------------------------------------------------------|
| `TIPO`                  | Perímetro (por ejemplo, `TAYLOR`, `TRIM`).                                   |
| `ASSET_CLASS`           | Clase de activo (por ejemplo, `FX`, `IR`, `INF`).                             |
| `FACTOR_TYPE`           | Tipo de factor (por ejemplo, `SPT`, `CRV`, `VOL`).                           |
| `UNIT`                  | Unidad (por ejemplo, `ES`, `NY`, o `null` para global).                      |
| `JAVA_ANNOTATION`       | Implementación concreta (enum de Java).                                      |
| `JAVA_ANNOTATION_ORIGEN`| Implementación original (usada para saber de dónde se redirige).              |
| `IS_COPY_FR`            | Si está en `YES`, se trata de una redirección.                               |

> **Ejemplo de contenido:**  
> ![Ejemplo de tabla](**[agregar imagen de tabla aquí]**)

---

## 3. Librería `librfrinventory`

### 📂 Package: `com.sgt.rfr.rfrinventorylib.database.userfr.repository`

- **Clase:** `TrfrUnderlyingsRepository`  
  - Ejecuta una query que recibe:
    - Unidad (`UNIT`)
    - Perímetro (`TIPO`)
    - Clase de activo (`ASSET_CLASS`)
    - Tipo de factor (`FACTOR_TYPE`)
    - Fecha (`dataDatePart`)
  - Devuelve un `List<String>` con los underlyings que cumplen con esos parámetros.

- **Clase:** `TrfrInventoryService`
  - Métodos principales:
    - `getInventoriesAvailable()`: usa `TrfrInventoriesRepository` para obtener un `List<TrfrInventoriesDTO>`
    - `getUnderlyingsNames()`: devuelve un array de `String` con underlyings filtrados por los parámetros.

---

## 4. Proyecto `svcrfrinventory`

### 📂 Package: `com.sgt.rfr.rfrinventory.service`

- **Clase:** `StrategyService`
  - Método: `redirectFactor`
    - Recibe lista de factores y `InventoryRequest`
    - Pide underlyings del request original y compara con los underlyings de la redirección.
    - Filtra factores por coincidencia de underlyings.
    - Actualiza `HeaderCtrl` con la nueva cantidad de factores.

  - Método: `findStrategy`
    - Llama a `getJavaAnnotationsFromInventory(InventoryRequest)`
    - Lógica:
      - Busca inventario que coincida exactamente.
      - Si no hay coincidencia por `UNIT`, busca uno con `UNIT = null`.

---

## 5. Proyecto `unixbatchrfr`

### 📂 Package: `com.sgt.rfr.unixbatchrfr.reports`

- **Clase:** `TaylorReportService`
  - Método: `buildReport`
    - Internamente llama a `processInventory()`:
      - Crea `InventoryRequest`
      - Verifica si hay implementación válida para TRIM y TAYLOR
      - Si ambas existen, compara y redirige según configuración en `TRFR_INVENTORIES`
      - Publica solo los factores coincidentes
      - Actualiza `HeaderCtrl` y construye el CSV final.

---

## 🔧 6. Configuración de Redirección

### 🖊️ Paso a paso

1. Insertar un nuevo registro en `USER_RFR.TRFR_INVENTORIES`.
2. Usar los mismos valores de `TIPO`, `ASSET_CLASS`, `FACTOR_TYPE` que el inventario original.
3. Asignar la `UNIT` de la unidad que quieres redirigir.
4. Poner `IS_COPY_FR = 'YES'`
5. En `JAVA_ANNOTATION`, indicar la implementación **destino**.
6. En `JAVA_ANNOTATION_ORIGEN`, indicar la implementación **origen** (la original).

### 📈 Ejemplo de redirección

| TIPO   | ASSET_CLASS | FACTOR_TYPE | UNIT | JAVA_ANNOTATION     | JAVA_ANNOTATION_ORIGEN | IS_COPY_FR |
|--------|-------------|-------------|------|----------------------|-------------------------|------------|
| TAYLOR | FX          | SPT         | ES   | SCM_TRIM_FX_SPT      | SCM_TAYLOR_FX_SPT       | YES        |

Este registro indica que para la unidad `ES`, cuando se solicite el inventario `TAYLOR FX SPT`, debe ser redirigido a la implementación de TRIM (`SCM_TRIM_FX_SPT`).

> **Importante:** se siguen usando los nombres de underlyings de `TAYLOR`, por eso no se elimina el registro con `UNIT = null`.

### 🚫 Sin redirección:
Si no hay un registro que coincida con la unidad, se utiliza el inventario definido con `UNIT = null`.

---

## 📜 7. Puntos clave

- La redirección se maneja sólo desde base de datos.
- El filtrado por underlyings garantiza que se publiquen factores consistentes.
- Hay lógica repetida en ambos proyectos para validar y aplicar la redirección.

---

## 🔍 8. Recursos sugeridos

- **Imagen de tabla de ejemplo**: [Agregar captura de la tabla USER_RFR.TRFR_INVENTORIES]
- **Ejemplo de `InventoryRequest`**: [Agregar snippet JSON o Java]
- **Ejemplo de enum con anotaciones Java**: [Agregar enum de implementaciones Java]

---

> Documento generado como plantilla técnica para desarrolladores. Actualiza los espacios entre corchetes donde se indica "[Agregar...]" con ejemplos reales.

