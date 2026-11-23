# Práctica 3: Máquina Expendedora Cafe

## Indice
1. [Introducción y Arquitectura del Sistema](#1-introducción-y-arquitectura-del-sistema)
2. [Montaje y Conexiones](#2-montaje-y-conexiones)
3. [Implementación Hilos y Funciones](#3-implementación-hilos-y-funciones)
4. [Gestión de Interrupciones Hardware](#4-gestión-de-interrupciones-hardware)
5. [Seguridad: Temporizador Watchdog](#5-seguridad-temporizador-watchdog)
6. [Problemas encontrados y soluciones](#6-problemas-encontrados-y-soluciones)
7. [Demostración](#7-demostración)

---

## 1. Introducción y Arquitectura del Sistema
Divido en sistema en muchos estados, cada uno lo asigno a un thead, sensor o logica. 
Los tres estados principales y sus sub estados son los siguientes:
- **APAGADO** : Simplemente cuando el sistema esta sin iniciar
- **ARRANQUE** : Mostrar Cargando... y enceder led, solo se llega cuando el sistema viene de apagado
- **SERVICIO** : Hace el Esperando cliente y gestiona llamadas a sus sub estados:
  - **TEMPERATURA_Y_HUMEDAD**
  - **MENU**
  - **PREPARAR**
- **ADMIN** : Muestra el menu y gestiona llamadas a sub estados:
  - **A_TEMPERATURA**
  - **A_DISTANCIA**
  - **A_CONTADOR**
  - **A_PRECIOS**
  - **CAMBIANDO_PRECIOS**

---

## 2. Montaje y Conexiones

<img width="2776" height="2493" alt="Cafe_bb" src="https://github.com/user-attachments/assets/e28fc1b6-c0de-4f5d-85ef-be6c6c8e6d2d" />

<img width="1080" height="1920" alt="WhatsApp Image 2026-11-23 at 13 14 02" src="https://github.com/user-attachments/assets/2d01f3c4-bcdf-4356-902d-8ce8b06233e6" />

Las conexiones que incluyo en el fritzing son las mismas que en mi placa la unica diferencia es que uno en la real la columna de pos y neg a la segunda mitad de la placa, y conecto los leddirectamente a negativo sin cables porque no me quedaban.  

---

## 3. Implementación Hilos y funciones
Usando arduino_threads para cada estado, y un controller para correr todos, los declaro con el enabled = false para que inicien apagados y les asigno un onRun para que cada hilo tenga su logica principal en una funcion. Para cada cambio de estado lo deshabilito igualando a false el enabled y poniendo a true al del estado que voy a pasar.

### Hilos Arranque
- **arranque_thread** : Muestra CARGANDO . . . y enciende el led 3 veces y pasa a Servicio

### Hilos Servicio
- **servicio_thread** : Se lanzan mediciones del ultrasonido saco la distancia duracion_ultrasonido * 0.0001715 para medirla en metros y muestra ESPERANDO CLIENTE mientras sea < 1, y sino salta ya a hacer temperatura y humedad.He tenido que implementar   servicio_thread.setInterval(100); porque me daba resultados raros a ratos ya que ntre itreacion e iteracion al tener 10 ms de mini delay para el sensor se sobreponian anteirores mediciones, asique al incluirlo se me soluciono el error.
- **temperatura_humedad_thread** : para no duplicar código entre el Servicio y el  Admin, el razonamiento esta en decisionTemperaturaHumedad(bool adminBool) y el print en printTempHumedad(), con adminBool = false muestra los datos durante 5 segundos cada 250 ms para evitar parpadeos y pasa automáticamente al menu
- **menu_thread** : saco el movimiento del joystick con la funcion que hice leerMovimiento() donde si le pasas true tambien mira el eje x y si no solo el eje y, te devuelve  +1 si es hacia arriba, -1 abajo, -2 izquierda y +2 derecha. Aparte uso tambien botonPulsado() que devuelve si se pulsa el boton con un antirrebote hecho con millis dentro. Desde la funcion onRun de este hilo, menu llamo a las funciones printMenuServicio donde tengo los strings construidos para cada linea incluido el array actualizado del precio, y se decide recibiendo como parametro contador_joy que en el menu se le suma 1 al ir hacia abajo y -1 hacia arriba, y para imprimir se usa la funcion printLCD. Cuando botonPulsado es true se pasa a preparando cafe.
- **preparar_thread** : saco el tiempo randome entre 4 y 8 segundos haciendo antes del randomSeed en A2 para total psudoaleatoriedad, imprimo por pantalla el preparando  y abajo mientras el tiempo transcurrido es menor que el random hago float progreso = (float)(millis() - tiempo_inicio_preparar) / duracion_preparar; que es un parseo que dimos el año pasado en sensores, para evitar problemas con unsigned long,y con esa operacion sacamos el numero entre 0 y 1 proporcional al tiempo pasado segun lo que tiene que durar, saco el brilo multibpliando 255*progreso y lo muestro. Despues cuando el tiempo a ha pasado imprimo Retirar bebida por 3 segundos y paso a SERVICIO de vuelta.

### Hilos Admin
Al entrar en la funcion se ponen los dos leds en HIGH y se muestra el menu interactivo usando las funciones que implementamos para servicio pero modificadas para admin con sus strings y condiciones, incluyendo una nueva que es si esta dentro de una de las sub funciones o estados y mueves el joystick a izquierrda osea -2 vuelves a la anterior funcion
- **a_temperatura_thread** Reutiliza decisionTemperaturaHumedad(true) sin límite de 5 segundos porque se pone adminBool a true
- **a_distancia_thread** Calcula la distancia como hicimos en servicio y la muestra por pantalla cada 500 ms para que no se solapen distancias
- **a_contador_thread** millis - contador incicado en millis al principio del programa / 1000
- **a_precios_thread` y `cambiando_precios_thread** volvemos a ver la interfaz del menu de productos podiendo navegar, y al hacer click pasamos a cambiar precios donde la primera linea se deja fijada que es done esta el nombre del producto (si contdor == 0 no se hace el cambio porque es donde pone producto y precio), y se va calculando el nuevo precio sumandole + 0.05 o - 0.05 con la subida o bajada del joysitick, y hago una doble implementacion donde si pulsas el boton se actualiza el precio y vuelves al menu de cambiar precios por si quieres volver a elegir que producto cambiar pero con el valor del precio y ya actualizada, pero si mientras cambias el precio mueves el joystick a la izquierda se vuelve al menu de cambiar precios sin actualizar el precio y con el anterior que tenia.

---

## 4. Gestión de Interrupciones Hardware

Como los pines 2 y 3 estaban ocupados por el lcd, conecte el boton al pin 18,que si que se puede usar apra interrupciones en la Arduino Mega.Hago un attachInterrupt con CHANGE, y llamo a la funcion donde hago un antirrebote de 50 ms , y cuando el boton esta pulsado y la var boton pulsado == false para evitar rebotes tambien, cojo el timepo inicial del pulsado, y pongo la variable a true, y luego cuando es HIGH y el boton pulsado true cojo el tiempo final pongo de vuelta a false esa variable y pongo a true una para leer en el loop donde trato las situaciones y mido la diferencia de tiempo para o reiniciar servicio o entrar a admin. Para entre 2 o 3 segundos y si se cumple que el estado sea < Admin y != Arranque y apagado, lo cual significa que es un estado de Servicio, entonces se reinicia el estado de servicio volviendo a ESPERAR CLIENTE, esto tambien ocurre cuando el tiempo pulsado es > 5s y estadoActual > Admin para volver de admin a servicio. Y cuando se pulsa mas de 5 s y estadoActual < Admin != Arranque y apagado entonces pasamos a modo Admin. Al hacer cambios de estados bruscos con el boton para no dejar hilos corriendo o bool que necesite en true, tengo una funcion llamad reinicio() donde pongo a false todos los hilos.enabled y reincio las variables necesarias, y luego ya hailitoo los estados a los qu eme metere tras pulsar el boton

---

## 5. Seguridad: Temporizador Watchdog
Para garantizar la recuperación automática ante cualquier bloqueo imprevistointegré el Watchdog con <avr/wdt.h>:
- Al iniciar setup() se llama a wdt_disable() para evitar bucles de reinicio
- Al finalizar setup() se activa con wdt_enable(WDTO_4S) (margen de 4 segundos, suficiente para que el timeout de pulseIn() no provoque falsos reinicio
- Al principio de cada iteración de loop() se ejecuta wdt_reset(). Si el microcontrolador se bloquea más de 4 segundos, se reinicia automáticamente volviendo al modo ARRANQUE

---

## 6. Problemas encontrados y soluciones
- **Lecturas del ultrasonidos** al principio el daba distancias falsas por dispararse demasiado rapido en cada vuelta del controlador. Se solucionado con ervicio_thread.setInterval(100)
- **Rebotes al soltar el boton de interrupcion** al soltar el botón tras 2 3 segundos, un rebote rápido a LOW sobrescribia el tiempo inicial y hacía fallar el reinicio. Se resolvio combinando un filtro de 50 ms con el bool boton_presionado.
* **Parpadeo del LCD** la pantalla. Se resolvió guardando posicion_menu_anterior en los menus y mostrando por pantalla cada mas tiempo

---

## 7. Demostración
[![Vídeo Circuito](https://img.youtube.com/vi/0cBU_-UbIYs/0.jpg)](https://www.youtube.com/watch?v=0cBU_-UbIYs)
