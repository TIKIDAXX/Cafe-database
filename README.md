## Descripcion de la base de datos

La base de datos está estructurada bajo un modelo relacional, donde cada entidad principal (Sucursal, Empleado, Proveedor, Producto, Cliente y Venta) se representa como una tabla con su correspondiente clave primaria, y las relaciones se implementan mediante claves foráneas para garantizar la integridad referencial. La relación entre sucursal y empleado es de tipo 1:N, ya que una sucursal puede tener varios empleados, pero cada empleado pertenece a una única sucursal. De igual forma, un empleado puede registrar múltiples ventas (1:N). Los proveedores mantienen una relación 1:N con los productos, ya que un proveedor puede suministrar varios productos, pero cada producto está asociado a un único proveedor. Las ventas se vinculan con empleados, sucursales y clientes mediante relaciones N:1, permitiendo identificar quién realizó la operación, dónde y para qué cliente. Además, la relación entre ventas y productos es de tipo N:M, resuelta mediante una tabla intermedia (detalle_venta) que almacena los productos incluidos en cada venta junto con información adicional como cantidad y precio unitario. Este diseño permite realizar consultas analíticas complejas manteniendo coherencia, escalabilidad y consistencia en los datos.

<details> <summary><strong>Función: la_propina_del_mesero</strong></summary>

Esta función calcula el total de propina que le corresponde a un mesero en un día específico, basándose en el 10% de las ventas que realizó en efectivo. La función valida que el empleado exista y que efectivamente sea mesero antes de realizar el cálculo, y devuelve el resultado redondeado a pesos enteros.

# Código de la Función
```sql
DELIMITER $$

CREATE FUNCTION la_propina_del_mesero(
    id_del_mesero INT,
    el_dia DATE
)
RETURNS INT
DETERMINISTIC
READS SQL DATA
BEGIN
    DECLARE total_propina INT;
    DECLARE el_nombre VARCHAR(100);
    DECLARE el_cargo VARCHAR(50);
    
    -- Validar que el empleado existe
    SELECT cargo INTO el_cargo
    FROM empleados 
    WHERE id_empleado = id_del_mesero;
    
    -- Si no existe, mostrar error
    IF el_cargo IS NULL THEN
        SIGNAL SQLSTATE '45000' 
        SET MESSAGE_TEXT = 'uy parce ese empleado no existe, revisa bien el ID';
    END IF;
    
    -- Validar que sea mesero
    IF el_cargo != 'Mesero' THEN
        SIGNAL SQLSTATE '45000' 
        SET MESSAGE_TEXT = 'este man no es mesero, los meseros son los que reciben propina';
    END IF;
    
    -- Obtener el nombre (opcional)
    SELECT CONCAT(nombre, ' ', apellido) 
    INTO el_nombre
    FROM empleados 
    WHERE id_empleado = id_del_mesero;
    
    -- Calcular propina: 10% de ventas en efectivo
    SELECT IFNULL(SUM(total_venta * 0.10), 0)
    INTO total_propina
    FROM ventas
    WHERE id_empleado = id_del_mesero
      AND DATE(fecha_venta) = el_dia
      AND metodo_pago = 'Efectivo';
    
    -- Redondear a pesos enteros
    SET total_propina = ROUND(total_propina, 0);
    
    RETURN total_propina;
    
END$$

DELIMITER ;
```
# Explicación técnica de las líneas clave

**IF el_cargo IS NULL THEN**
Validación de existencia: Evalúa si la variable el_cargo contiene NULL, lo que indica que el SELECT previo no encontró ningún empleado con el ID proporcionado. La consulta SELECT cargo INTO el_cargo FROM empleados WHERE id_empleado = id_del_mesero asigna NULL cuando no hay coincidencias. Este IF evita que la función continúe con un empleado inexistente.

**IF el_cargo != 'Mesero' THEN**
Validación de cargo: Compara el valor de el_cargo con el string 'Mesero' usando el operador de desigualdad !=. Si el empleado existe pero su cargo no es 'Mesero', la función lanza un error porque solo los meseros reciben propina directamente por atención en mesa.

**SELECT IFNULL(SUM(total_venta * 0.10), 0)**
Cálculo con manejo de nulos:
total_venta * 0.10 calcula el 10% de cada venta (porcentaje estimado de propina).
SUM() agrega todos esos valores del conjunto de resultados.
IFNULL(..., 0) reemplaza cualquier resultado NULL (cuando no hay ventas que cumplan las condiciones) con 0, evitando que la función retorne NULL.

**SET total_propina = ROUND(total_propina, 0)**
Redondeo a entero: La función ROUND() con segundo parámetro 0 redondea el valor decimal al entero más cercano. Esto es necesario porque las propinas se pagan en pesos completos, sin manejar fracciones de peso. Ejemplo: 12,345.67 → 12,346.

</details>
<details> <summary><strong>Procedimiento: repartir_propinas_entre_meseros</strong></summary>

Este procedimiento calcula automáticamente la propina que le corresponde a cada mesero en una fecha específica y guarda los resultados en una tabla llamada propinas_diarias. Lo que hace es bien sencillo: agarra todos los meseros que trabajaron ese día, mira cuánto vendió cada uno en efectivo, les saca el 10% y guarda esa información ordenada de mayor a menor propina, para que el administrador sepa rápidamente cuánto darle a cada uno sin tener que estar ejecutando la función para cada mesero por separado.

# Código de la Prcedimiento
```sql
DELIMITER $$

CREATE PROCEDURE repartir_propinas(
    IN fecha_reparto DATE
)
BEGIN
    
    DELETE FROM propinas_diarias WHERE fecha = fecha_reparto;
    
    INSERT INTO propinas_diarias (fecha, id_mesero, nombre_mesero, total_ventas_efectivo, propina_a_pagar)
    SELECT 
        fecha_reparto,
        e.id_empleado,
        CONCAT(e.nombre, ' ', e.apellido),
        COALESCE(SUM(v.total_venta), 0),
        SUM(la_propina_del_mesero(e.id_empleado, fecha_reparto))
    FROM empleados e
    LEFT JOIN ventas v ON e.id_empleado = v.id_empleado 
        AND DATE(v.fecha_venta) = fecha_reparto
        AND v.metodo_pago = 'Efectivo'
    WHERE e.cargo = 'Mesero'
    GROUP BY e.id_empleado
    HAVING COALESCE(SUM(v.total_venta), 0) > 0;
    
    SELECT CONCAT('Propina repartida para ', ROW_COUNT(), ' meseros el ', fecha_reparto) AS mensaje;
    
END$$

DELIMITER ;
```
# Explicación técnica de las líneas clave

**COALESCE(SUM(v.total_venta), 0)**
SUM(v.total_venta) suma las ventas de cada mesero. Si no hay ventas, devuelve NULL. COALESCE() reemplaza ese NULL por 0. Así evitamos que aparezcan valores vacíos y siempre tenemos un número para mostrar o usar en otros cálculos.

**DELETE FROM propinas_diarias WHERE fecha = fecha_reparto**
Elimina registros previos de la misma fecha para evitar duplicados antes de insertar los nuevos cálculos.

**SUM(la_propina_del_mesero(e.id_empleado, fecha_reparto))**
Reutiliza la función creada anteriormente para calcular la propina de cada mesero, evitando repetir la lógica de cálculo del 10% y redondeo.

**HAVING COALESCE(SUM(v.total_venta), 0) > 0**
Filtra solo los meseros que tuvieron ventas en efectivo ese día, los que no tuvieron ventas no se insertan en la tabla.

**SELECT CONCAT(...) AS mensaje**
Muestra un mensaje simple indicando cuántos meseros recibieron propina usando ROW_COUNT() que cuenta las filas insertadas.

</details>
<details> <summary><strong>Evento: repartir_propinas_automatico</strong></summary>

Este evento ejecuta automáticamente el procedimiento repartir_propinas cada día a las 8:00 PM, calculando y guardando las propinas del día actual sin necesidad de que alguien lo haga manualmente.

# Código del Evento
```sql
DELIMITER $$

CREATE EVENT repartir_propinas_automatico
ON SCHEDULE EVERY 1 DAY
STARTS CONCAT(CURDATE(), ' 20:00:00')
DO
BEGIN
    CALL repartir_propinas(CURDATE());
END$$

DELIMITER ;
```
# Explicación técnica

**ON SCHEDULE EVERY 1 DAY**
El evento se ejecuta una vez al día, todos los días.

**STARTS CONCAT(CURDATE(), ' 20:00:00')**
La primera ejecución será hoy a las 8:00 PM. CONCAT pega la fecha actual con la hora 20:00:00.

**CALL repartir_propinas(CURDATE())**
Dentro del evento se ejecuta el procedimiento creado anteriormente, pasándole la fecha actual con CURDATE().

</details>
<details> <summary><strong>Trigger: evitar_ventas_sin_stock</strong></summary>

Este trigger se activa antes de insertar una venta en la tabla ventas y verifica que los productos que se van a vender tengan suficiente stock en inventario. Si no hay suficiente, no deja hacer la venta y muestra un mensaje de error. Así evitamos vender cosas que no tenemos.

# Código del Trigger
```sql
DELIMITER $$

CREATE TRIGGER evitar_ventas_sin_stock
BEFORE INSERT ON detalle_venta
FOR EACH ROW
BEGIN
    DECLARE v_stock_actual INT;
    DECLARE v_nombre_producto VARCHAR(100);
    
    -- Verificar el stock actual del producto que se quiere vender
    SELECT stock_actual, nombre INTO v_stock_actual, v_nombre_producto
    FROM productos
    WHERE id_producto = NEW.id_producto;
    
    -- Si no hay suficiente stock, cancelar la venta
    IF v_stock_actual < NEW.cantidad THEN
        SIGNAL SQLSTATE '45000' 
        SET MESSAGE_TEXT = CONCAT('No hay suficiente ', v_nombre_producto, '. Solo quedan ', v_stock_actual, ' unidades');
    END IF;

END$$

DELIMITER ;
```
# Explicación técnica de las líneas clave

**BEFORE INSERT ON detalle_venta**
El trigger se ejecuta antes de insertar cada línea del detalle de venta. Así podemos validar y evitar que se guarde si algo anda mal.
  
**FOR EACH ROW**
Se ejecuta una vez por cada producto que se está vendiendo en la factura.

**SELECT stock_actual, nombre INTO v_stock_actual, v_nombre_producto**
Consulta el stock actual y el nombre del producto que se quiere vender y guarda esos valores en las variables v_stock_actual y v_nombre_producto.

**IF v_stock_actual < NEW.cantidad THEN**
Compara el stock disponible con la cantidad que se quiere vender. Si el stock es menor, entra al IF y cancela la operación.

**SIGNAL SQLSTATE '45000'**
Lanza un error personalizado que detiene la inserción. El mensaje muestra cuántas unidades quedan y de qué producto.

**CONCAT('No hay suficiente ', v_nombre_producto, '. Solo quedan ', v_stock_actual, ' unidades')**
Arma el mensaje de error pegando texto fijo con las variables que obtuvimos. Ejemplo: "No hay suficiente Café Tinto pequeño. Solo quedan 3 unidades".

</details>
