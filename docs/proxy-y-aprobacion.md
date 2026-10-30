# Proxy de tráfico y aprobación humana

ARTEX es un sistema de pruebas de penetración autónomas. Esta nota solo describe los controles que ya expone el programa. Úsalo en un laboratorio aislado, con objetivos que te pertenezcan o para los que tengas autorización escrita.

## Puertos

`./start.sh` arranca la interfaz y la API en `:8787`. El proxy de grabación de tráfico escucha en `:8788` si se pasa `-proxy :8788`.

```bash
./start.sh -addr :8787 -proxy :8788
```

`-addr` es la UI y la API. `-proxy` es el proxy que registra el tráfico del laboratorio. Los parámetros se reenvían al binario `artex`.

## Qué queda registrado

El proxy es de tipo registro. La pantalla **Tráfico** muestra lo que pasó por ese proxy. No sustituye una autorización ni amplía el alcance de una tarea.

## Aprobación humana

Antes de que una herramienta siga, el sistema puede dejarla en espera. La revisión está en **Intercept** / **拦截审批**, en el registro global de aprobaciones y en la tarjeta de la conversación.

Mientras una llamada está pendiente:

- no se ejecuta hasta que una persona la aprueba o la rechaza;
- la tarea sigue visible en la UI;
- rechazarla detiene esa llamada, no borra el historial ya registrado.

La configuración de agentes y el número de workers por tarea están en la configuración del sistema. El valor por defecto documentado es 3 workers.

## Qué no hace esta guía

No indica cómo enrutar tráfico hacia sistemas ajenos ni cómo saltarse la aprobación. Si una herramienta queda bloqueada, la acción correcta es revisarla en la UI o detener la tarea.
