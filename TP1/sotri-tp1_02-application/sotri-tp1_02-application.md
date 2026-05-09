# Actividad 2 - Paso 2

## ¿Cómo asigna tiempo de procesamiento a cada tarea?

Lo hace con el SysTick timer interrumpiendo las tareas y verificando las prioridades para delegar el control de la CPU a la tarea correspondiente.

## ¿Cómo FreeRTOS elige qué Tarea debe ejecutarse en un momento dado?

Lo hace en base al nivel de prioridad de cada tarea.

## ¿Cómo la prioridad relativa de cada Tarea afecta el comportamiento del sistema?

Afecta haciendo que aquella de mayor prioridad se ejecute idefinidamente si no suelta el recurso (starvation).
En cambio en caso de tener igual prioridad iran cambiando entre las dos dependiendo a la interrupcion del PendSV.

## ¿Cuáles son los estados en los que puede encontrarse una Tarea?

Running, ready, suspend y blocked.

## ¿Cómo implementar Tareas?

Cada tarea se debe implementar con una funcion en C.

## ¿Cómo crear una o más instancias de una Tarea?

Para crear una tarea se debe usar la funcion:

```c
xTaskCreate(<Pointer to the task function>,
            <Task string name id>,
            <Reserved memory for the stack>,
            <Pointer to the parameters>,
            <Priority level>,
            <Pointer to a handler function>);
```

En caso de crear mas de una tarea, se debe llamar repetidas veces a esta funcion con diferentes parametros.

## ¿Cómo eliminar una Tarea? 

Para eliminar una tarea se hace lo siguiente:


```c
vTaskDelete(<task handler>);
vTaskDelete(NULL); // Elimina la tarea que llama a la funcion
```

# Actividad 2 - Paso 3

Se aumento la prioridad a la tarea del boton por lo que se observo que la tarea del led nunca se ejecuto:

```
[info] Task BTN is running - Tick [mS] =   0
[info]  Task BTN - BTN PRESSED
[info]  Task BTN - BTN HOVER
[info]  Task BTN - BTN PRESSED
[info]  Task BTN - BTN HOVER
```

Esto es debido a que la tarea del boton nunca suelta el recurso y por lo tanto se ejecuta indefinidamente.
