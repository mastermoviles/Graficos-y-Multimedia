## Proyecto

### Plataforma de televisión _online_

Como proyecto de la asignatura vamos a implementar una plataforma de televisión _online_ que constará de las siguientes aplicaciones:

* Aplicación iOS para el acceso a radio, videoclub, y emisiones en directo.
* Aplicación Android para retransmitir vídeo en directo a los canales

### Aplicación iOS

En la aplicación iOS veremos 4 secciones diferentes:

* **Radio**: Mostrará una lista de contenidos de audio (música, programas de radio, etc) que podremos reproducir en el dispositivo. Seleccionando uno de ellos se reproducirá el audio en el dispositivo. Estos contenidos estarán almacenados de forma local en el dispositivo.
* **Videoclub**: Mostrará una lista de contenidos de vídeo (series, películas, programas, etc) bajo demanda. Seleccionando un elemento se reproducirá el vídeo en el dispositivo. A estos vídeos se accederá a través de un servidor que nos proporcione vídeo bajo demando vía _streaming_.
* **Directo**: Mostrará una lista de canales de televisión. Al seleccionar uno de ellos veremos sus emisiones en directo. Estas emisiones se realizarán desde un dispositivo Android.
* **Acerca de**: Debe mostrar una pantalla con el nombre de la aplicación y el nombre del autor, y una gráfica de _tarta_ generada de forma dinámica en la que se muestre la proporción de vídeo en directo, video bajo demanda, y audio que ofrece nuestra plataforma.

¿Qué tipo de navegación consideras más adecuada en iOS para definir las secciones anteriores?

### Aplicación Android

Tendremos también una aplicación Android que nos permitirá hacer emisiones en directo. Tendremos una serie de canales de televisión predefinidos, y desde la pantalla principal podremos:

* Mostrar la pantalla "Acerca de" con el nombre de la aplicación y del autor.
* Configurar la emisión. En la pantalla de configuración deberemos poder modificar la IP del servidor de _streaming_, el canal en el que vamos a emitir, y la calidad del vídeo a emitir.
* Comenzar la emisión. Lanzará una pantalla en la que se enviará vídeo capturado desde la cámara al servidor de _streaming_, mostrando una previsualización del vídeo emitido.


### Video bajo demanda en Wowza

Vamos a publicar vídeo bajo demanda en el servidor Wowza, para así poderlo reproducir en dispositivos móviles vía _streaming_.

* Copia los vídeos que quieras ofrecer como contenido bajo demanda en la carpeta de contenido VOD de Wowza.
* Comprueba en el _Test Player_ de Wowza que los vídeos se ven correctamente.
* Copia las direcciones para reproducir los vídeos en iOS y en Android. Podemos probarlo introduciendo esta dirección en el cuadro de dirección del navegador del móvil. Según si vamos a reproducir en emuladores o dispositivos reales el procedimiento cambiará.

**Prueba en emuladores**

La prueba en emuladores/simuladores dependerá de si utilizamos el simulador iOS o el emulador Android, ya que el primero es una aplicación más que se ejecuta en nuestra máquina local, mientras que el segundo se ejecuta dentro de una máquina virtual:

* En el **simulador de iOS** podemos utilizar la dirección que nos proporciona Wowza directamente.
* En el **emulador de Android** deberemos utilizar la IP del _host_ en el que se ejecuta la máquina virtual que emula el dispositivo. Esta IP es `10.0.2.2` (equivale a la dirección de _loopback_ en la máquina _host_). Sustituiremos la IP de la dirección que nos ha proporcionado Wowza por dicha IP.

**Prueba en dispositivos reales**

Para probar en dispositivos reales debemos hacer que nuestros dispositivos estén en la misma red que el servidor Wowza. Para ello deberemos activar en el Mac la opción de compartir Internet, siguiendo los siguientes pasos:

* Entramos en _Manzana > Preferencia del Sistema > Compartir_
* Con la casilla _Compartir Internet_ desmarcada seleccionamos la opción _Wi-Fi_.
* Entramos en _Opciones de Wi-Fi …_.
* Ponemos como SSID nuestro _login_ de Campus Virtual (para evitar conflictos entre redes), y elegimos un password propio.
* Conectamos el móvil a la WiFi del Mac con la que se comparte Internet. Ahora los dispositivos estarán conectados a la misma red.

![Configuración de hotspot](imagenes/streaming-01-hotspot.png)


Ahora podremos probar el enlace en el móvil directamente, ya que al estar en la misma red debe poder ver la dirección donde está escuchando Wowza.