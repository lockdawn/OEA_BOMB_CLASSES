# Random Minefield Spawner for Arma 3
Este script en SQF permite crear minas antipersonales o antivehículo de manera aleatoria dentro del área de un trigger cuando unidades BLUFOR entran en él.
Cuando los jugadores salen del área, las minas se eliminan automáticamente para evitar sobrecarga en el servidor.

Con esto se simulan zonas minadas dinámicas, evitando tener que colocar manualmente decenas de minas fijas en el editor.

# ¿Para qué situaciones se recomienda?
- Misiones multijugador donde se desea añadir incertidumbre táctica al movimiento de tropas.
- Escenarios de contrainsurgencia con IEDs y minas ocultas.
- Situaciones donde el diseño busca forzar el uso de caminos seguros o rutas designadas.
- Escenarios dinámicos donde los jugadores no repitan siempre la misma experiencia, ya que las minas cambian de ubicación en cada activación.

# ¿Cómo se usa?
1. Crear un trigger en el editor.
2. Configurar el trigger para detectar **BLUFOR PRESENT**.
3. Configurar el trigger como **REPEATABLE**.
4. Configurar el trigger como **SERVER ONLY**.
5. En el campo On Activation del trigger, llamar la función:
    ```
    if (isServer) then {
        [thisTrigger,20] call OEA_fnc_spawnBombsInTrigger;
    } else {
        [thisTrigger,20] remoteExecCall ["OEA_fnc_spawnBombsInTrigger",2];
    };
    ```
    Donde:
        - `thisTrigger` → es el área del trigger.
        - 20 → número de minas a generar.
    4. En On Deactivation, limpiar las minas:
    ```
    if (isServer) then {
      [thisTrigger] call OEA_fnc_deleteBombsInTrigger;
    } else {
      [thisTrigger] remoteExecCall ["OEA_fnc_deleteBombsInTrigger", 2];
    };
    ```

# Factores de impacto en el rendimiento
1. Creación de objetos (minas)
    - Cada `createMine` genera un objeto con IA mínima (tiene sensores para detectar jugadores, armas, explosión).
    - 20 minas ≈ 20 entidades más que el servidor debe simular.
    - 100 minas distribuidas en el mapa ya empiezan a notarse, pero no tanto como 100 soldados IA.
    - Comparado con tropas enemigas, las minas son baratas en rendimiento.
2. Bucle de generación (`while`)
    - Está limitado: intenta varias posiciones hasta llegar al número solicitado.
    - Se ejecuta en un spawn, así que no congela el servidor.
    - Cada intento hace sleep corto → reparte carga, no genera lag pico.
3. Eliminación de minas
    - `deleteVehicle` es ligero, limpia bien la memoria y la red.
    - Como las minas se borran al salir del trigger, no se acumulan.
4. Sin sincronización pesada en red
    - Como las minas existen solo en el servidor (isServer check), los clientes solo reciben los objetos colocados, no el cálculo.
    - Eso mantiene el rendimiento estable en los jugadores.

# En números aproximados
- 10–30 minas activas: impacto casi nulo (<1% uso CPU adicional).
- 50–100 minas activas: ligero consumo de red y CPU, dependiendo de cuántos jugadores estén cerca para que las minas entren en simulación.
- 200+ minas activas: puede notarse, sobre todo si están todas en áreas de combate activas.

# Buenas prácticas para mantener rendimiento
- Spawn dinámico (como haces con triggers): perfecto, evita que todo esté cargado desde el inicio.
- Limitar cantidad máxima por trigger: 20–30 es seguro.
- No abuses de IEDs Remotos: son más pesados que minas simples porque pueden tener lógica de radio/detonador.
- Zonas grandes: mejor varios triggers medianos que uno gigante con 200 minas.

En conclusión: este código es más ligero y eficiente que colocar grandes campos minados estáticos.

# ¿Lo ejecuta el servidor o el cliente?
La ejecución se fuerza solo en el servidor `(remoteExec ["...", 2]`.

¿Por qué?
- En multiplayer, el servidor es el responsable de manejar objetos del mundo y sincronizarlos entre todos los clientes.
- Si se ejecutara desde cada cliente, podrían generarse múltiples duplicados de las mismas minas.
- Con la ejecución centralizada, todos los jugadores ven el mismo resultado y el rendimiento es más predecible.

La función de spawn de minas debe ejecutarse solo en el servidor, porque el servidor es quien crea los objetos y los sincroniza a todos los clientes.

- **Cliente que entra al trigger** → el código On Activation se ejecuta en **su máquina**.

```
if (isServer) then {
    [thisTrigger,20] call OEA_fnc_spawnBombsInTrigger;
} else {
    [thisTrigger,20] remoteExecCall ["OEA_fnc_spawnBombsInTrigger",2];
};
```

- isServer → si se ejecuta en el servidor, simplemente llama a la función.
- else → si se ejecuta en un cliente, le dice al servidor que lo haga por él (remoteExecCall al servidor, 2 significa “solo server”).

## Resumen práctico
- Lo que importa: las minas siempre las crea el servidor.
- Los clientes solo “informan” al servidor si son quienes activaron el trigger.
- Si tienes la certeza de que el trigger siempre se dispara en el servidor, puedes simplificar a:

# Pros
- Dinámico: cada activación genera minas en diferentes posiciones.
- Optimizado: solo existen mientras el trigger esté activo.
- Fácil integración con triggers en editor.
- Mayor realismo e incertidumbre para los jugadores.
- Compatible con misiones multijugador.

# Contras
- No detecta automáticamente si una mina destruye vehículos o IA (esto depende del motor de Arma 3).
- Puede sorprender demasiado a jugadores inexpertos si no hay pistas de campos minados.
- Requiere coordinación con otros scripts que manejen objetivos, ya que elimina todo al salir del trigger.

# Conclusión
El Random Minefield Spawner for Arma 3 es una herramienta ligera y flexible para diseñadores de misiones que buscan añadir tensión, realismo y variedad a sus escenarios de Arma 3.

Su diseño basado en triggers y ejecución en el servidor garantiza un bajo impacto en el rendimiento, incluso en servidores multijugador con muchos jugadores.

Ideal para escenarios dinámicos donde la incertidumbre y la optimización son clave.

# Notas finales y buenas prácticas
- Repeatable puede quedar en Yes si quieres que vuelva a activarse tras desactivación.
- Evita valores excesivos (20–40 minas por trigger es razonable).
- `createMine` sincroniza objetos a clientes; usa `isServer` para evitar duplicados.
- El flag `OEA_bombs_spawned_flag` impide que múltiples activaciones simultáneas generen sets extra.
- Si quieres previnir spawns si ya hay minas globales, puedo agregar una cuenta global máxima.
