# Cierre bancario con Spring Batch

**Autor:** Martin Gonzalez Rico

## Cómo correrlo

    docker compose up -d --wait
    ./correr.sh 2026-09-30 prueba
    ./ver-batch.sh

## Día 1 · Mi primer Job

### Boleto de salida

1. ¿Qué diferencia hay entre un proceso batch y la API REST de la Semana 3? Da dos.

   - La API REST se queda prendida esperando peticiones y nunca termina, a menos que yo lo termine.
     El batch arranca, hace su trabajo y termina solo. 
   - La API atiende un dato por petición (un empleado) y hay alguien esperando la respuesta.
     El batch procesa muchos datos de una vez (todos los movimientos del día) y nadie
     interactúa con él mientras corre. 

2. ¿Qué es un Job, qué es un Step y qué es un Tasklet?
    - Job (trabajo): es el proceso completo, en mi caso cierreDelDiaJob. Es el contenedor
     de los steps.
    - Step (paso): es una fase del Job. En este caso se tiene dos: saludoStep y
     verificarArchivoStep.
    - Tasklet (task): es una sola operación que corre dentro de un step. 

3. Con tus tablas: ¿qué diferencia hay entre una **JobInstance** y una **JobExecution**?

    - La JobInstanc* es el Job para unos parámetros, en este caso es "el cierre de una fecha".
    - La JobExecution es cada intento de correr JobInstance.

4. ¿Por qué Spring Batch no deja correr dos veces el cierre del 28?

    Porque ya existe una instancia de cierreDelDiaJob con fecha=2026-09-28 y está completa. Spring Batch lo hace para evitar problema como: si el cierre del 28 se corriera otra vez, el banco cobraría dos veces las comisiones de ese día. 

5. (MP-4, paso 6) Si mañana llega el archivo del 25 y corres otra vez el cierre del 25, ¿será otra instancia u
   otra ejecución de la misma? ¿Por qué lo crees?

    Creo que va a ser otra ejecución de la misma instancia.Lo creo porque la primera vez terminó FAILED (no existía el archivo) y no COMPLETED, sí me debería dejar volver a correrla.


## Día 2 · El primer chunk

### Boleto de salida

1. ¿Qué diferencia hay entre un step de tipo Tasklet y uno de tipo chunk?

    Un Tasklet sirve para hacer una tarea específica, mientras que un Chunk procesa los datos por grupos, leyendo, procesando y escribiendo.

2. ¿Qué hace cada una de las tres piezas de un chunk? ¿Cuál es opcional?

    - Reader: lee los datos.
    - Processor: procesa o transforma los datos.
    - Writer: escribe los datos.
    El Processor es opcional, porque los datos pueden pasar directamente del Reader al Writer.

3. Con 45 movimientos y chunks de 10, ¿cuántos commits habría? ¿Y con chunks de 50?

    Con chunks de 10 habría 5 commits: 10 + 10 + 10 + 10 + 5.
    Con chunks de 50 habría 1 commit, porque los 45 movimientos caben en un solo chunk.

4. ¿Por qué el Escritor recibe el chunk completo y no un movimiento a la vez?
    
    Porque así puede escribir varios registros juntos y hacer menos operaciones, lo que hace que el procesamiento sea más eficiente.

5. Mi predicción de la MP-3, paso 1: ¿qué habría pasado sin el Procesador?
    Los movimientos pasarían directamente del Reader al Writer, sin ninguna transformación o validación intermedia.

## Día 3 · Parámetros, fallas y reinicio

### Boleto de salida

1. ¿Qué diferencia hay entre una JobInstance y una JobExecution? Usa como ejemplo el cierre del 25.

Una JobInstance es el trabajo de un día específico, por ejemplo, el cierre del 25 de diciembre.
Una JobExecution es cada intento de ejecutar ese trabajo.
En el cierre del 25 tuvo dos ejecuciones: la 4 falló y la 8 (en mi caso 10) terminó correctamente.


2. ¿En qué caso Spring Batch se niega a correr un cierre, y en qué caso lo reinicia?

Spring Batch se niega a correrlo cuando la misma instancia ya terminó en COMPLETED.
Si la ejecución anterior terminó en FAILED, Spring Batch puede reiniciar esa misma instancia desde donde se quedó.

3. En el reinicio del día 5, ¿por qué el step de carga leyó 10 movimientos y no 20?

Porque el primer chunk de 10 movimientos ya había sido procesado y confirmado. Cuando el step falló en el segundo chunk, esos 10 ya quedaron guardados.
Al reiniciar, Spring Batch no vuelve a hacer los 10 que ya confirmó, sino que continúa con los que faltan. Por eso en el reinicio leyó 10, y al final quedaron los 20 movimientos en la tabla.

4. ¿Qué diferencia hay entre un movimiento **filtrado** y uno **omitido**?

Un movimiento filtrado es uno que el Procesador decide no guardar. Por ejemplo, si el tipo no es DEPOSITO ni RETIRO, el procesador devuelve null y Spring Batch lo cuenta en FILTER_COUNT.
Un movimiento omitido (skip) es uno que tiene un error al leerlo, pero Spring Batch está configurado para tolerarlo y saltarlo hasta llegar a un límite.

5. ¿Por qué importa el código de salida, si el estado ya queda en las tablas?

El código de salida importa porque el planificador lo usa para saber si el proceso terminó bien o falló. El 0 significa bien y otro número significa error.