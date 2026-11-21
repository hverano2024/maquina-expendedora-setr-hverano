# Blog Maquina Expendedora

## Indice

---

## Detalles implementacion

- Hago otro hilo temperatura_humedad_thread y su funcion onrun temperatura_humedad, y hago lo siguiente dentro.Para no crear 2 funciones que hagan lo mismo cambiando solo 1 if (1 para admin otro para antes del cafe) creo una variable bool para saber si esta siendo llamado desde el admin o no que es temperatura_humedad_admin, esta iniciada a false y en servicio se pone a false tambien, y hago 2 funciones una para simplemente hacer el print con las variables que le llegan tempertautra y humedad print_temp_humeda (este no esta onrun en ningun sitio es una funcion normal que se llama desde la del thread que si esta on run) y en la otra que aplico la lógica de las decisiones del cuando hago el print. Aqui tengo variables bool iniciar_temp_humedad que es para hacer el inicio de la funcion sacando millis inicial para las dos variables que uso en la funcion, estas funciones son unsigned long tiempo_temp_y_humedad que es para medir si han pasado los 5 segundos que deberian pasar mostrando temperatura y humedad (si admin es false) y luego tiempo_temp_y_humedad_frecuencia que es para que se haga el print en el lcd cada 0,25 segundos porque si se corre de manera constante va demasiado rapido y se ve mal la pantalla, cuando pasan los 5 segundos pongo las vatiables necesitadas a false, la del enable y pongo a true el enable del thread del menu y pongo estado == menu. Despues debajo en el else de si es llamado por el admin
- 

