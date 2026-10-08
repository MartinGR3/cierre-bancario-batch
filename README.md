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

## Día 4 · De MySQL a MongoDB

### Boleto de salida

1. ¿Qué hace cada uno de los tres steps de tu Job, y de qué tipo es cada uno?

    - verificarArchivoStep: revisa que exista el archivo CSV del día. Es un Tasklet.
    - cargarMovimientosStep: lee los movimientos del archivo, los procesa y los guarda en MySQL. Es un Chunk y trabaja de 10 en 10.
    - publicarSaldosStep: lee los saldos desde MySQL y los guarda en MongoDB. También es un Chunk y trabaja de 3 en 3.

2. ¿Por qué el cierre del 9 no duplicó los saldos, y el del 10 (sin `@Id`) sí?

    El cierre del 9 no duplicó porque usamos @Id y la cuenta se convierte en el _id de MongoDB. Entonces, si la cuenta ya existe, se actualiza el documento en lugar de crear otro.
    El cierre del 10 sí duplicó porque quitamos @Id. MongoDB creó un _id diferente para cada documento, por eso volvió a guardar las mismas cuentas como documentos nuevos.

3. Al reiniciar el cierre del 11, ¿por qué no se cargó otra vez el archivo?

    Porque los primeros dos steps ya estaban en COMPLETED antes de que fallara el tercer step. Spring Batch sabe que ya terminaron y cuando reiniciamos solo vuelve a ejecutar el step que faltaba, que era publicarSaldosStep. Por eso no volvió a cargar el archivo.     

4. ¿Qué diferencia hay entre `spring-boot-starter-data-mongodb` y «Spring Batch MongoDB» (`batch-data-mongodb`)?

    - spring-boot-starter-data-mongodb sirve para que nuestra aplicación pueda trabajar con MongoDB, en este caso para guardar los saldos.
    - batch-data-mongodb sirve para que Spring Batch guarde sus propias tablas de control en MongoDB en lugar de MySQL. En este caso no usamos ese, porque las tablas BATCH_* siguen estando en MySQL.

## Lo que aprendí esta semana

Esta semana aprendí que un proceso batch sirve para procesar datos de forma automática. Un Job es el trabajo completo y puede tener varios steps. Los steps pueden ser de tipo Tasklet o Chunk. En un Chunk tenemos un lector(Reader), un procesador(Processor) y un escritor(Writter). También aprendí que Spring Batch guarda información de las ejecuciones, si un step falla, podemos reiniciar el Job, cuando lo reiniciamos, los steps que ya terminaron no se vuelven a ejecutar esto ayuda a que los datos no se carguen otra vez.  

