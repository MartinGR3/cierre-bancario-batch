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
