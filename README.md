# Modelo y Simulación de un Electrolizador PEM

Este repositorio contiene un conjunto de scripts de Octave/MATLAB diseñados para modelar, validar y simular el comportamiento electroquímico y térmico de un electrolizador PEM.

El proyecto se divide en dos partes principales:
1.  **Validación del Modelo:** Se recrea y compara una curva de polarización (Voltaje vs. Corriente), curva Hidrógeno producido vs potencia eléctrica (H2 vs kW) y una curva de trabajo específico con datos del siguiente paper de referencia. Link: https://www-sciencedirect-com.pucdechile.idm.oclc.org/science/article/pii/S1876610217363506
2.  **Simulación Dinámica:** Se utiliza el modelo validado para simular el comportamiento térmico del electrolizador (aumento de temperatura) cuando es alimentado por una fuente de potencia variable, como un panel solar en un día nublado.

---

## Parte 1: Validación de Curva de Polarización (main_PROYECTO.m)

Este script se enfoca en la validación del modelo electroquímico en estado estacionario.

### Objetivo
El objetivo es recrear la curva de polarización (Voltaje vs. Densidad de Corriente) presentada en el paper (identificado en los gráficos como "Modelo Colbertaldo P.") y comparar los resultados del modelo propio bajo las mismas condiciones de operación (55°C y 70 bar). 

### Funcionamiento
1.  **Define Condiciones:** Establece los parámetros de operación fijos (temperatura `T_op_C` y presión `P_op_bar`). Temperatura y presión se puede cambiar. 
2.  **Cálculo del Modelo:** Llama a la función `calcularCurvaPolarizacion.m` con un vector de densidades de corriente (`i_input_vector`) el cual debe ser escrito como lista dentro del código. 
3.  **Cálculo de Sobrepotenciales:** Dentro de `calcularCurvaPolarizacion.m`, el voltaje total de la celda se calcula como la suma de los potenciales y sobrepotenciales:
    * **$V_{id}$ (Potencial Ideal/Nernst):** Calculado usando `calculo_DeltaG.m` y las presiones parciales (obtenidas con la librería `XSteam.m`).
    * **$V_{act}$ (Sobrepotencial de Activación):** Modelado con la simplificación de Tafel.
    * **$V_{ohm}$ (Sobrepotencial Óhmico):** Considera la resistividad de electrodos y la conductividad de la membrana (calculada con la correlación de Springer). Se toma el supuesto de una resistencia de contacto de 0.175 ohms.
    * **$V_{diff}$ (Sobrepotencial por Difusión):** Modelado en base a la corriente límite (`i_L`).
4.  **Comparación:** El script calcula el error entre el vector de voltajes calculado (`V_cell`) con el vector de voltajes del paper (`Vcell_paper`), este útlimo debe ser escrito como lista dentro del código. 
5.  **Generación de Gráficos:** Se generan tres figuras que comparan el "Modelo recreado" vs. el "Modelo Colbertaldo P.":
    * Curva de Polarización (V vs. A/cm²).
    * Producción de H₂ (Nm³/h) vs. Potencia Eléctrica (kW).
    * Trabajo Específico (kWh/kg H₂) vs. Producción de H₂ (g/h).

---

## Parte 2: Simulación Dinámica con Perfil Solar (main_SIMULACION.m)

Este script utiliza el modelo electroquímico validado para simular el comportamiento dinámico de la PEM, enfocándose en la respuesta térmica a una fuente de energía intermitente de un panel solar en un día nublado. link paper de referencia: https://research-ebsco-com.pucdechile.idm.oclc.org/c/r6zury/viewer/pdf/atk445bn25?route=details

### Objetivo
Simular la evolución de la **temperatura** y la **corriente** del stack PEM durante 24 horas, asumiendo que es alimentado por un panel solar en un día con alta variabilidad (nublado).

### Funcionamiento
1.  **Define Parámetros del Sistema:** Establece parámetros globales para la simulación dinámica, como la capacidad térmica total del stack (`C_tot`), el coeficiente de transferencia de calor con el ambiente (`UA`), y la temperatura ambiente (`T_amb_K`). El sistema trabaja a presión constante.
2.  **Perfil de Potencia Solar:** Define una función anónima `P_target_func(t)` que entrega la potencia eléctrica (en Watts) disponible del panel solar en cualquier segundo `t` del día. Esta función simula un día nublado con variaciones rápidas. 
3.  **Sistema de Ecuaciones (DAE):** El núcleo de la simulación es un sistema de Ecuaciones Diferenciales-Algebraicas (DAE) resuelto con `ode15s`. Este sistema está definido en `sistema_dae.m`:
    * **Ecuación Diferencial (Balance de Energía):** Es la Ecuación 1 (`res1`). Modela cómo cambia la temperatura de la PEM (`dT/dt`).
        $C_{tot} \cdot \frac{dT}{dt} = Q_{gen\_neto} - Q_{loss}$
        Donde `Q_gen_neto` es el calor generado (Potencia eléctrica menos el $\Delta H$ de la reacción) y `Q_loss` es el calor perdido al ambiente.
    * **Ecuación Algebraica (Balance de Potencia):** Es la Ecuación 2 (`res2`). Asegura que la potencia eléctrica consumida por la PEM es igual a la potencia suministrada por el panel solar en ese instante.
        $P_{panel}(t) = P_{stack}(i, T)$
4.  **Solver:** El solver `ode15s` itera en cada paso de tiempo para encontrar la temperatura (`T_K_act`) y la corriente (`i_cell_act`) que satisfacen ambas ecuaciones simultáneamente.
5.  **Cálculo de Voltaje Interno:** Para resolver la ecuación algebraica, el solver llama continuamente a `calcular_voltaje_ESCALAR.m`. Esta es una versión optimizada del modelo de la Parte 1, diseñada para calcular el voltaje para un solo valor (escalar) de corriente y temperatura.
6.  **Resultados:** El script genera un gráfico de 24 horas que muestra:
    * Arriba: El perfil de potencia de entrada del panel solar (kW).
    * Abajo: La evolución de la temperatura del electrolizador (°C) como respuesta a esa potencia.

---

## Estructura de Archivos y Dependencias

### Scripts Principales
* `main_PROYECTO.m`: Ejecuta la validación del modelo estacionario.
* `main_SIMULACION.m`: Ejecuta la simulación dinámica de 24 horas.

### Funciones del Modelo
* `calcularCurvaPolarizacion.m`: (Vectorizado) Calcula la curva V-I completa para la validación.
* `calcular_voltaje_ESCALAR.m`: (Escalar) Calcula un solo punto (V, i) para el solver DAE. Maneja el caso `i=0` para evitar errores matemáticos.
* `sistema_dae.m`: Define el sistema DAE (balance de energía y potencia) para `ode15s`.
* `calculo_DeltaG.m` y `calculo_DeltaG_MODIFICADO.m`: Calculan propiedades termodinámicas (ΔG, ΔH, ΔS) en función de la temperatura, usando integrales de polinomios de Cp.

### Bibliotecas Externas
* `XSteam.m`: **Dependencia externa necesaria.** Es la biblioteca para calcular propiedades termodinámicas del agua y vapor (ej. `psat_T` para la presión de saturación).

## Cómo Usar
1.  Asegúrese de tener Octave o MATLAB instalado.
2.  Coloque todos los archivos `.m` en el mismo directorio.
3.  IMPORTANTE: Asegúrese de que la biblioteca `XSteam.m` esté en el path de Octave/MATLAB.
4.  Para la validación del modelo estacionario:
    ```octave
    >> main_PROYECTO
    ```
5.  Para la simulación dinámica del sistema:
    ```octave
    >> main_SIMULACION
    ```
