Squema
Ejercicio del modulo 5 de Data Analytics de Coderhouse


Ejercicio SQL: Diferencias entre UNION y UNION ALL

Este documento explica en detalle el comportamiento, diferencias de rendimiento, casos de uso reales y manejo de errores al consolidar conjuntos de datos en SQL mediante `UNION` y `UNION ALL`.

---

Cantidad de filas devueltas y eliminación de duplicados

¿Cuántas filas devuelve cada consulta y por qué son distintas?
UNION ALL` devuelve la suma directa de las filas de ambas consultas. Mantiene la totalidad de los registros devueltos por cada instrucción `SELECT`, sin verificar ni eliminar repeticiones.
`UNION` (de forma predeterminada, equivalente a `UNION DISTINCT`) consolida el resultado combinando los registros y **eliminando las filas completamente duplicadas** (aquellas donde los valores de *todas* las columnas coinciden exactamente).

Ejemplo concreto de eliminación con `UNION`
Imaginemos que ejecutamos dos consultas sobre las tablas `Clientes_Sucursal_A` y `Clientes_Sucursal_B`:

Resultado de `Consulta 1` (Sucursal A):
  | id_cliente | nombre | email |
  | :--- | :--- | :--- |
  | 101 | Ana Gómez | ana@email.com |
  | 102 | Carlos Ruiz | carlos@email.com |

Resultado de `Consulta 2` (Sucursal B):
  | id_cliente | nombre | email |
  | :--- | :--- | :--- |
  | 102 | Carlos Ruiz | carlos@email.com |
  | 103 | María López | maria@email.com |

Resultados obtenidos:

Con `UNION ALL` (4 filas):
  Conserva la fila de *Carlos Ruiz* proveniente de ambas sucursales.
  1. `101 | Ana Gómez | ana@email.com`
  2. `102 | Carlos Ruiz | carlos@email.com`
  3. `102 | Carlos Ruiz | carlos@email.com` *(duplicado conservado)*
  4. `103 | María López | maria@email.com`

Con `UNION` (3 filas):
  La fila idéntica `102 | Carlos Ruiz | carlos@email.com` se detecta como duplicada. Se elimina una de las repeticiones y solo se devuelve una fila para Carlos:
  1. `101 | Ana Gómez | ana@email.com`
  2. `102 | Carlos Ruiz | carlos@email.com`
  3. `103 | María López | maria@email.com`

---

2. Rendimiento y operación interna

¿Por qué `UNION ALL` es más eficiente que `UNION`?
`UNION ALL` es sensiblemente más rápido y consume menos recursos de memoria y CPU porque simplemente concatena los flujos de datos recibidos de cada consulta y los devuelve directamente al usuario.

 Operaciones adicionales que realiza `UNION`
Para identificar y eliminar duplicados, el motor de base de datos debe procesar todo el conjunto de datos combinado ejecutando una de las siguientes operaciones internas:

1. Ordenamiento (`Sort / Distinct Sort`): Junta todas las filas e intenta ordenarlas por todas sus columnas en memoria (o en disco temporal/TempDB si el volumen de datos es muy grande) para detectar cuáles son idénticas.
2. Tablas de dispersión (`Hash Aggregate / Hash Match`): Crea una estructura Hash en memoria para comparar cada fila entrante contra las ya existentes.

Ambas operaciones incrementan el uso de memoria RAM, generan sobrecarga de CPU y pueden requerir lectura/escritura en disco en grandes volúmenes de datos.

---

3. Casos de negocio reales para cada operador

 Casos para usar `UNION ALL`

1. Consolidación de transacciones o registros históricos independización de fuentes:
   Ejemplo: Generar un reporte financiero consolidado combinando las tablas `Ventas_2024` y `Ventas_2025`. Cada venta representa un evento único en el tiempo con su propio ID primario. No existen duplicados entre tablas y se requiere procesar millones de registros con el menor tiempo de respuesta posible.
2. Generación de logs o auditoría de eventos multicanal:
   Ejemplo: Combinar el historial de interacciones de un cliente proveniente de la tabla `Logs_Web_App` y `Logs_Mobile_App` para una línea de tiempo. Si un usuario realizó dos clics idénticos en ambas plataformas a la misma hora, nos interesa conservar ambos eventos para auditar la actividad exacta.

Casos para usar `UNION`

1. Padrón o directorio unificado de contactos/clientes:
   Ejemplo: Crear una lista maestra para una campaña de email marketing combinando la lista de `Compradores_Online` y `Asistentes_Eventos_Presenciales`. Se utiliza `UNION` para asegurar que ningún cliente reciba correos duplicados ni se sobrecarguen los costos del proveedor de correos masivos.
2. Consolidación de catálogos o maestros de productos:
   Ejemplo: Unificar las listas de proveedores de insumos (ej. `Catalogo_Proveedor_A` y `Catalogo_Proveedor_B`) para mostrar un catálogo interno de productos disponibles. Si dos proveedores ofrecen exactamente el mismo ítem con los mismos atributos, se utiliza `UNION` para consolidarlos en una única vista limpia.

---

 4. Incompatibilidad de columnas y errores de SQL

Para utilizar `UNION` o `UNION ALL`, la sintaxis de SQL exige cumplir estrictamente dos reglas en las consultas consolidadas:
1. Las consultas deben devolver el mismo número de columnas.
2. Los tipos de datos de las columnas coincidentes en posición deben ser compatibles entre sí (o convertibles implícitamente).

 ¿Qué ocurre cuando no coinciden?

Diferencia en el número de columnas:
  Si una consulta selecciona 3 columnas y la otra selecciona 2, la ejecución se interrumpe inmediatamente antes de procesar los datos.
  Error generado (según el motor de base de datos):
    PostgreSQL / MySQL: `ERROR: EACH UNION query must have the same number of columns.`
    SQL Server: `Msg 205, Level 16, State 1: All queries combined using a UNION, INTERSECT or EXCEPT operator must have an equal number of expressions in their target lists.`

Incompatibilidad en los tipos de datos:
  Si la primera columna del primer `SELECT` es de tipo `INTEGER` y la primera columna del segundo `SELECT` es de tipo `VARCHAR` (con texto no numérico como `"Activo"`), la conversión implícita fallará.
Error generado (según el motor de base de datos):
    SQL Server: `Msg 245, Level 16: Conversion failed when converting the varchar value 'Activo' to data type int.`
    PostgreSQL: `ERROR: UNION types integer and text cannot be matched.`
