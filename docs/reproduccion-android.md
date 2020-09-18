# Reproducción de medios en Android

La capacidad de reproducir contenido multimedia es una característica presente en la práctica totalidad de las terminales telefónicas existentes en el mercado hoy en día. Muchos usuarios prefieren utilizar las capacidades multimedia de su teléfono, en lugar de tener que depender de otro dispositivo adicional para ello.

En esta sesión vamos a aprender a añadir contenido multimedia en nuestras aplicaciones. En concreto, veremos cómo reproducir audio o video en una actividad.

La reproducción de contenido multimedia se lleva a cabo por medio de la clase `MediaPlayer`. Dicha clase nos permite la reproducción de archivos multimedia almacenados como recursos de la aplicación, en ficheros locales, en proveedores de contenido, o servidos por medio de _streaming_ a partir de una URL. En todos los casos, como desarrolladores, la clase `MediaPlayer` nos permitirá abstraernos del formato así como del origen del fichero a reproducir.

## Reproducción de audio

### Tipos de fuentes de audio

Podemos reproducir audio proveniente de diferentes fuentes:

* Un recurso de la aplicación
* Un fichero en el dispositivo local
* Una URL remota

Incluir un fichero de audio en los recursos de la aplicación para poder ser reproducido durante su ejecución es muy sencillo. Simplemente creamos una carpeta `raw` dentro de la carpeta `res`, y almacenamos en ella sin comprimir el fichero o ficheros que deseamos reproducir. A partir de ese momento el fichero se identificará dentro del código como `R.raw.nombre_fichero` (obsérvese que no es necesario especificar la extensión del fichero).

Para reproducir un fichero de audio almacenado en el móvil o en un servidor remoto simplemente deberemos especificar su URL (local o remota).

> **Importante**: En caso de necesitar reproducir un medio alojado en una URL remota, será necesario que la aplicación solicite el permiso `INTERNET` en el fichero `AndroidManifest.xml`:
```xml
<uses-permission android:name="android.permission.INTERNET" />
```

### Inicialización del reproductor de medios

Para reproducir un fichero de audio tendremos que seguir una secuencia de pasos. En primer lugar deberemos crear una instancia de la clase `MediaPlayer` e indicar qué fichero será el que se reproducirá. Tenemos dos opciones para hacer esto.

La **primera opción** para inicializar la reproducción multimedia es por medio del método `setDataSource`, el cual asigna una fuente multimedia a una instancia ya existente de la clase `MediaPlayer`.

```java
MediaPlayer mediaPlayer = new MediaPlayer();
mediaPlayer.setDataSource("/sdcard/test.mp3");
mediaPlayer.prepare();
```

Al instanciar la clase `MediaPlayer` se encontrará en estado _idle_. En este estado lo primero que debemos hacer es indicar el fichero a reproducir. Una vez hecho esto pasa a estado _inicializado_. En este estado ya sabe qué fichero ha de reproducir, pero todavía no se ha preparado para ello (inicializar _bufferes_, etc), por lo que no podrá comenzar la reproducción. Para prepararlo deberemos llamar al método `prepare()`, con lo que tendremos el reproductor listo para empezar a reproducir el audio.


La **segunda opción** consiste en crear una instancia de la clase `MediaPlayer` por medio del método `create`. En este caso se deberá pasar como parámetro, además del contexto de la aplicación, el identificador del recurso, como se puede ver en el siguiente ejemplo:

```java
Context appContext = getApplicationContext();

// Recurso de la aplicación
MediaPlayer resourcePlayer =
	MediaPlayer.create(appContext, R.raw.my_audio);
// Fichero local (en la tarjeta de memoria)
MediaPlayer filePlayer =
	MediaPlayer.create(appContext, Uri.parse("file:///sdcard/localfile.mp3"));
// URL
MediaPlayer urlPlayer =
	MediaPlayer.create(appContext, Uri.parse("http://site.com/audio/audio.mp3"));
// Proveedor de contenido
MediaPlayer contentPlayer =
	MediaPlayer.create(appContext, Settings.System.DEFAULT_RINGTONE_URI);
```

En este caso el método `create()` se encarga de asignar la fuente de audio y además pasar el reproductor a estado _preparado_. Por lo tanto, en este caso no será necesario llamar a `prepare()`, sino que podremos reproducir el medio directamente. Aunque es más sencillo que la primera opción, también resulta menos flexible.


### Inicialización asíncrona

La llamada a `prepare()` puede resultar bastante costosa y producir un retardo considerable, especialmente cuando accedemos a medios externos a la aplicación. Por este motivo debemos evitar llamarla desde el hilo de eventos, para evitar bloquear este hilo.

Podemos crear un hilo secundario y realizar la preparación del medio desde él, pero también contamos con una variante del método anterior que nos facilitará realizar la preparación de forma asíncrona, fuera del hilo de eventos. Esta variante es `prepareAsync()`.

Podemos utilizar un _listener_ de tipo `MediaPlayer.OnPreparedListener` para que se nos notifique con el método `onPrepared` cuándo está preparado el reproductor, y así poder comenzar la reproducción. Este listener se deberá registrar en el `MediaPlayer` con el método `setOnPreparedListener()`:

```java
public class MiActividad extends Activity implements MediaPlayer.OnPreparedListener {
    MediaPlayer mMediaPlayer = null;

    public void reproducir() {
        mMediaPlayer = new MediaPlayer();
        mMediaPlayer.setOnPreparedListener(this);
        mMediaPlayer.prepareAsync();
    }

    public void onPrepared(MediaPlayer player) {
        mMediaPlayer.start();
    }
}
```


### Métodos del reproductor de medios

Una vez que la instancia de la clase `MediaPlayer` ha sido inicializada, podemos comenzar la reproducción mediante el método `start`. También es posible utilizar los métodos `stop` y `pause` para detener y pausar la reproducción. Si se detuvo la reproducción de audio mediante el método `stop` será imprescindible invocar el método `prepare` antes de poder reproducirlo de nuevo mediante una llamada a `start`. Por otra parte, si se detuvo la reproducción por medio de `pause`, tan sólo será necesario hacer una llamada a `start` para continuar en el punto donde ésta se dejó.


Otros métodos de la clase `MediaPlayer` que podríamos considerar interesante utilizar son los siguientes:

* `setLooping` nos permite especificar si el clip de audio deberá volver a reproducirse cada vez que finalice.

```java
if (!mediaPlayer.isLooping())
	mediaPlayer.setLooping(true);
```

* `setScreenOnWhilePlaying` nos permitirá conseguir que la pantalla se encuentre activada siempre durante la reproducción. Tiene más sentido en el caso de la reproducción de video, que será tratada en la siguiente sección.

```java
mediaPlayer.setScreenOnWhilePlaying(true);
```

* `setVolume` modifica el volumen. Recibe dos parámetros que deberán ser dos números reales entre 0 y 1, indicando el volumen del canal izquierdo y del canal derecho,
respectivamente. El valor 0 indica silencio total mientras que el valor 1 indica máximo volumen.

```java
mediaPlayer.setVolume(1f, 0.5f);
```

* `seekTo` permite avanzar o retroceder a un determinado punto del archivo de audio. Podemos obtener la duración total del clip de audio con el método `getDuration`,
mientras que `getCurrentPosition` nos dará la posición actual. En el siguiente código se puede ver un ejemplo de uso de estos tres últimos métodos.

```java
mediaPlayer.start();

int pos = mediaPlayer.getCurrentPosition();
int duration = mediaPlayer.getDuration();

mediaPlayer.seekTo(pos + (duration-pos)/10);
```

### Liberación del reproductor de medios

Una acción muy importante que deberemos llevar a cabo una vez haya finalizado definitivamente la reproducción (porque se vaya a salir de la aplicación o porque se vaya a cerrar la actividad donde se reproduce el audio) es destruir la instancia de la clase `MediaPlayer` y liberar su memoria. Para ello deberemos hacer uso del método `release`.


```java
if(mediaPlayer!=null) {
    mediaPlayer.release();
    mediaPlayer = null;
}
```

Por ejemplo, si nuestra actividad crea un reproductor en `start()`, deberemos asegurarnos de destruirlo siempre en `stop()` para evitar que pueda haber más de uno simultáneamente.


### _Streams_ de audio y control de volumen

Android reproduce el audio en diferentes _streams_ según su naturaleza: música, alarmas, notificaciones, llamadas, etc. Cada uno de estos _streams_ tiene su propio nivel de volumen, de forma que el volumen del tono de llamada puede ser distinto al volumen del reproductor de música. Cuando el usuario manipule los botones de control de volumen del dispositivo, estará modificando el volumen del _stream_ que esté sonando actualmente.

A la hora de reproducir sonido con el `MediaPlayer` podemos especificar el _stream_ dentro del cual lo queremos reproducir con `setAudioStreamType`. Siempre llamaremos a este método en el estado _idle_, antes de haber especificado la fuente de audio:

```java
MediaPlayer mediaPlayer = new MediaPlayer();
mediaPlayer.setAudioStreamType(AudioManager.STREAM_MUSIC);
mediaPlayer.setDataSource(getApplicationContext(), "sdcard/test.mp3");
mediaPlayer.prepare();
mediaPlayer.start();
```

> Es importante destacar que no podremos especificar el _stream_ de audio si lo creamos con el atajo `create`.

Como hemos comentado, cuando usamos los botones de control de volumen del dispositivos afectamos al _stream_ que esté sonando actualmente. Si no estuviese sonando ningún audio, entonces por defecto afectaría al _stream_ del tono de llamada.

Si estamos dentro de una aplicación como por ejemplo un videojuego, aunque no esté sonando nada normalmente nos interesará que al manipular el control de volumen dentro del videojuego se modifique el volumen de música del juego, y no el del todo de llamada. Para indicar a qué _stream_ debe afectar el volumen utilizaremos el método `setVolumenControlStream()`:

```java
setVolumeControlStream(AudioManager.STREAM_MUSIC);
```

Cuando la actividad o fragmento en el que hayamos llamado al método anterior esté visible, el control de volumen afectará al _stream_ de música, y no al por defecto.

### Reproducción de audio en segundo plano

Puede intersarnos dejar nuestro reproductor funcionando en segundo plano aunque cerremos la actividad, por ejemplo para implementar un reproductor de música o de _podcasts_. Para hacer esto deberemos iniciar el reproductor desde un servicio, en lugar de una actividad:

```java
public class MiServicio extends Service implements MediaPlayer.OnPreparedListener {
    private static final String ACTION_PLAY = "es.ua.jtech.action.PLAY";
    private static final String EXTRA_URI = "es.ua.jtech.extras.URI";
    MediaPlayer mMediaPlayer = null;

    public int onStartCommand(Intent intent, int flags, int startId) {
        if (intent.getAction().equals(ACTION_PLAY)) {
            mMediaPlayer = new MediaPlayer();
            mMediaPlayer.setDataSource(intent.getExtra(EXTRA_URI));
            mMediaPlayer.setOnPreparedListener(this);
            mMediaPlayer.prepareAsync();
        }
    }

    public void onPrepared(MediaPlayer player) {
        player.start();
    }
}
```

Cuando el móvil no se esté utilizando normalmente la pantalla y la CPU se apagarán automáticamente para ahorrar batería. Si esto ocurre la música se detendrá. Para evitarlo necesitamos adquirir lo que se conoce como un _wake lock_. Esto nos permitirá mantener el móvil activo aunque el usuario no lo esté utilizando.

Para poder obtener un _wake lock_ lo primero que deberemos es solicitar el permiso correspondiente:

```xml
<uses-permission android:name="android.permission.WAKE_LOCK" />
```

Una vez contemos con el permiso, podemos indicar al reproductor de medios que necesitamos un _wake lock_ parcial (se puede apagar la pantalla pero no la CPU):

```java
mMediaPlayer.setWakeMode(getApplicationContext(), PowerManager.PARTIAL_WAKE_LOCK);
```

Si estamos accediendo a un fichero remoto deberemos indicar también que no se apague la WiFi, mediante un _wifi lock_:

```java
WifiLock wifiLock = ((WifiManager) getSystemService(Context.WIFI_SERVICE))
    .createWifiLock(WifiManager.WIFI_MODE_FULL, "wifilock");

wifiLock.acquire();
```

Al parar o pausar la reproducción podemos liberar este _lock_ para así ahorrar batería:

```java
wifiLock.release();
```

Aunque la reproducción se produzca en segundo plano, normalmente el usuario querrá estar al tanto de la reproducción y poder deternerla. Por ello es conveniente que el servicio funcione en modo _foreground_. Para ello deberemos crear una notificación de tipo _ongoing_ (evento en curso) que se mostrará en la barra de notificaciones mientras el servicio esté activo. Llamando a `startForeground()` desde nuestro servicio haremos que dicha notificación se muestre mientras el servicio esté activo:

```java
// Crea un PendingIntent para iniciar la actividad ReproductorActivity
PendingIntent pi = PendingIntent.getActivity(getApplicationContext(), 0,
                new Intent(getApplicationContext(), ReproductorActivity.class),
                PendingIntent.FLAG_UPDATE_CURRENT);

// Creamos una notificación que lance el PendingIntentAnterior
Notification notification = new Notification.Builder(mContext)
        .setContentTitle("Reproductor de musica")
        .setContentText("Reproduciendo " + nombreCancion)
        .setSmallIcon(R.drawable.icono_play)
        .setTicker("Reproduccion en curso")
        .setOngoing(true)
        .addAction(R.drawable.icono_play, "Reproductor de musica", pi)
        .build();

startForeground(NOTIFICATION_ID, notification);
```

En el código anterior podemos ver que hemos añadido a la notificación un `PendingIntent` para que al pulsar sobre ella se abra la actividad del reproductor, y así el usuario podría pausar, detener, o cambiar la música que esté sonando.

Cuando queramos que nuestro servicio deje de funcionar en modo _foreground_ llamaremos a:

```java
stopForeground(true);
```

También podríamos incluir en la propia notificación controles para poder parar o reanudar el audio. Para ello podríamos utilizar un objeto de tipo `RemoteViews` y cargar en él el _layout_ que queramos que tenga la notificación. Al construir la notificación podremos establecer la vista remota a aplicar con `setContent`. 

```java
RemoteViews remoteViews = new RemoteViews(getPackageName(), R.layout.notificacion_audio);

intent = new Intent(ACTION_PLAY);       
pendingIntent = PendingIntent.getService(getApplicationContext(),
        REQUEST_CODE_PLAY, intent,
        PendingIntent.FLAG_UPDATE_CURRENT);

remoteViews.setOnClickPendingIntent(R.id.boton_play,
        pendingIntent);

intent = new Intent(ACTION_PAUSE);       
pendingIntent = PendingIntent.getService(getApplicationContext(),
        REQUEST_CODE_PAUSE, intent,
        PendingIntent.FLAG_UPDATE_CURRENT);

remoteViews.setOnClickPendingIntent(R.id.boton_pause,
        pendingIntent);

Notification notification = new NotificationCompat.Builder(getApplicationContext())
        .setSmallIcon(R.drawable.ic_launcher).setOngoing(true)
        .setWhen(System.currentTimeMillis())                
        .setContent(remoteViews)
        .build();

startForeground(NOTIFICATION_ID, notification);     
```

Al pulsar los botones de la notificación se lanzará un _intent_ con una acción `ACTION_PLAY` o `ACTION_PAUSE`, definidas ambas por nuestra aplicación. Haremos que nuestro servicio se ejecute cuando se lancen dichas acciones, añadiéndolas a un`<intent-filter>`:

```xml
<service
    android:name="es.ua.jtech.MiServicio"
    android:exported="false" >
    <intent-filter>
        <action android:name="eps.ua.es.ACTION_PLAY" />
        <action android:name="eps.ua.es.ACTION_PAUSE" />
        <category android:name="android.intent.category.DEFAULT" />
    </intent-filter>
</service>
```

Deberemos actualizar el código del servicio para que soporte también la acción `ACTION_PAUSE`:

```java
public class MiServicio extends Service implements MediaPlayer.OnPreparedListener {
    private static final String ACTION_PLAY = "es.ua.jtech.action.PLAY";
    private static final String ACTION_PAUSE = "es.ua.jtech.action.PAUSE";
    private static final String EXTRA_URI = "es.ua.jtech.extras.URI";
    MediaPlayer mMediaPlayer = null;

    public int onStartCommand(Intent intent, int flags, int startId) {
        if (intent.getAction().equals(ACTION_PLAY)) {
            if(mMediaPlayer == null) {
                mMediaPlayer = new MediaPlayer();
                mMediaPlayer.setDataSource(intent.getExtra(EXTRA_URI));
                mMediaPlayer.setOnPreparedListener(this);
                mMediaPlayer.prepareAsync();
            } else {
                mMediaPlayer.play();
            }
        } else if (intent.getAction().equals(ACTION_PAUSE)) {
            if(mMediaPlayer != null) {
                mMediaPlayer.pause();
            }
        }
    }

    public void onPrepared(MediaPlayer player) {
        player.start();
    }
}
```

A partir de Android 4.0 también podemos utilizar la clase `RemoteControlClient` para implementar estos controles, y a partir de Android 5.0 este mecanismo es reemplazado por la clase `MediaSession`. Podemos obtener más información sobre el nuevo mecanismo en:

http://developer.android.com/guide/topics/ui/notifiers/notifications.html#controllingMedia

### Gestión del foco del audio


La posibilidad de reproducir audio en un servicio en segundo plano nos trae también la problemática de poder tener varios sonidos reproduciéndose simultáneamente. Por ejemplo, podemos estar reproduciendo músia al mismo tiempo que suena la alerta por la llegada de un mensaje, lo cual puede suponer que no oigamos el aviso de mensaje.

Para evitar este problema deberemos coordinar la reproducción de los diferentes audios. La forma que tiene Android de resolverlo es mediante un sistema en el que cada aplicación podrá solitar el foco para reproducir audio. Por ejemplo, si estamos reproduciendo música y otra aplicación solicita el foco, podemos deterner la música momentáneamente o bajar su volumen (técnica conocida como _ducking_) mientras la otra aplicación reproduce el sonido, hasta que nos devuelva el foco.

En primer lugar deberemos solicitar el foco para reproducir audio mediante el servicio `AudioManager`:

```java
AudioManager audioManager = (AudioManager) getSystemService(Context.AUDIO_SERVICE);
int result = audioManager.requestAudioFocus(this,
    AudioManager.STREAM_MUSIC,
    AudioManager.AUDIOFOCUS_GAIN);

if (result != AudioManager.AUDIOFOCUS_REQUEST_GRANTED) {
    // No se ha concedido el foco
}
```

En este caso anterior vemos que hemos solicitado el foco para el _stream_ de reproducción de música. Además, al solicitar el foco debemos proporcional un _listener_ de tipo `AudioManager.OnAudioFocusChangeListener` que tendrá  siguiente estructura:

```java
class MiServicio extends Service
                implements AudioManager.OnAudioFocusChangeListener {

    public void onAudioFocusChange(int focusChange) {
        switch (focusChange) {
            case AudioManager.AUDIOFOCUS_GAIN:
                if (mMediaPlayer == null) {
                    initMediaPlayer();
                } else if (!mMediaPlayer.isPlaying()) {
                    mMediaPlayer.start();
                }
                mMediaPlayer.setVolume(1.0f, 1.0f);
                break;

            case AudioManager.AUDIOFOCUS_LOSS:
                if (mMediaPlayer.isPlaying()) {
                    mMediaPlayer.stop();
                }
                mMediaPlayer.release();
                mMediaPlayer = null;
                break;

            case AudioManager.AUDIOFOCUS_LOSS_TRANSIENT:
                if (mMediaPlayer.isPlaying()) {
                    mMediaPlayer.pause();
                }
                break;

            case AudioManager.AUDIOFOCUS_LOSS_TRANSIENT_CAN_DUCK:
                if (mMediaPlayer.isPlaying()) {
                    mMediaPlayer.setVolume(0.1f, 0.1f);
                }
                break;
        }
    }
}
```

Vemos en este ejemplo que hay varias mas de perder el foco:

* `AUDIOFOCUS_LOSS`: Se pierde de forma definitiva. Detenemos y liberamos el reproductor.
* `AUDIOFOCUS_LOSS_TRANSIENT`: Se pierde de forma temporal. Pausamos el reproductor.
* `AUDIOFOCUS_LOSS_TRANSIENT_CAN_DUCK`: Se pierde de forma temporal y se permite que simplemente reduzcamos el volumen del audio actual (_ducking_).

Cuando ganemos el foco de nuevo (`AUDIOFOCUS_GAIN`) deberemos inicializar el reproductor si no estaba inicializado todavía, reanudar la reproducción si se había pausado, y restaurar el volumen original.

> La característica _audio focus_ sólo está disponible a partir de Android 2.2 (API 8). Si queremos tener compatibiliad con versiones anteriores deberemos comprobar la versión de Android en código y sólo utilizar _audio focus_ si es mayor o igual que la 8.

### Desconexión de auriculares

Estamos acostumbrados a ver que en las aplicaciones móviles para reproducir música, al desconectar los auriculares la reproducción se detiene automáticamente para evitar que el ruido repentino pudiera causar molestias.

Este comportamiento no es automático, sino que lo deberemos programar nosotros. Para ello deberemos capturar el _intent_ `AUDIO_BECOMING_NOISY` mediante un _broadcast receiver_:

```xml
<receiver android:name=".MusicIntentReceiver">
   <intent-filter>
      <action android:name="android.media.AUDIO_BECOMING_NOISY" />
   </intent-filter>
</receiver>
```

Dentro del _broadcast receiver_ podremos por ejemplo realizar la acción de detener la reproducción actual:

```java
public class MusicIntentReceiver extends android.content.BroadcastReceiver {
   @Override
   public void onReceive(Context ctx, Intent intent) {
      if (intent.getAction().equals(
                    android.media.AudioManager.ACTION_AUDIO_BECOMING_NOISY)) {
          stopMusic();
      }
   }
}
```

## Reproducción de vídeo

### Reproducir vídeo mediante VideoView

La reproducción de vídeo es muy similar a la reproducción de audio, salvo dos particularidades. En primer lugar, no es posible reproducir un clip de vídeo almacenado como parte de los recursos de la aplicación. En este caso no existe ningún método para reproducir un vídeo a partir de un `id` de recurso. Lo que si que podemos hacer es representar un recurso `raw` de tipo vídeo mediante una URL. Esta URL tendrá la siguiente forma (hay que especificar como _host_ el paquete de nuestra aplicación, y como ruta el _id_ del recurso):


```java
Uri.parse("android.resource://es.ua.jtech/" + R.raw.video)
```


En segundo lugar, el vídeo necesitará de una superficie para poder reproducirse. Esta superficie se corresponderá con una vista dentro del layout de la actividad.



Existen varias alternativas para la reproducción de vídeo, teniendo en cuenta lo que acabamos de comentar. La más sencilla es hacer uso de una vista de tipo `VideoView`, que encapsula tanto la creación de una superficie en la que reproducir el vídeo como el control del mismo mediante una instancia de la clase `MediaPlayer`. Este método será el que veamos en primer lugar.

El primer paso consistirá en añadir la vista `VideoView` a la interfaz gráfica de la aplicación. Para ello añadimos el elemento en el archivo de layout correspondiente:


```xml
<VideoView android:id="@+id/superficie"
	      android:layout_height="fill_parent"
	      android:layout_width="fill_parent">
</VideoView>
```

Dentro del código Java podremos acceder a dicho elemento de la manera habitual, es decir, mediante el método `findViewById`. Una vez hecho esto, asignaremos una fuente que se corresponderá con el contenido multimedia a reproducir. El `VideoView` se encargará de la inicialización del objeto `MediaPlayer`. Para asignar un video a reproducir podemos utilizar cualquiera de estos dos métodos:


```java
videoView1.setVideoUri("http://www.mysite.com/videos/myvideo.3gp");
videoView2.setVideoPath("/sdcard/test2.3gp");
```

Una vez inicializada la vista se puede controlar la reproducción con los métodos `start`, `stopPlayback`, `pause` y `seekTo`. La clase `VideoView` también incorpora el método `setKeepScreenOn(boolean)`con la que se podrá controlar el comportamiento de la iluminación de la pantalla durante la reproducción del clip de vídeo. Si se pasa como parámetro el valor `true` ésta permanecerá constantemente iluminada.

El siguiente código muestra un ejemplo de asignación de un vídeo a una vista `VideoView` y de su posterior reproducción. Dicho código puede ser utilizado a modo de esqueleto en nuestra propia aplicación. También podemos ver un ejemplo de uso de `seekTo`, en este caso para avanzar hasta la posición intermedia del video.


```java
VideoView videoView = (VideoView)findViewById(R.id.superficie);
videoView.setKeepScreenOn(true);
videoView.setVideoPath("/sdcard/ejemplo.3gp");

if (videoView.canSeekForward())
	videoView.seekTo(videoView.getDuration()/2);

videoView.start();

// Hacer algo durante la reproducción

videoView.stopPlayback();
```



### Reproducir vídeo con MediaPlayer


La segunda alternativa para la reproducción de vídeo consiste en la creación de una superficie en la que dicho vídeo se reproducirá y en el uso directo de la clase `MediaPlayer`. La superficie deberá ser asignada manualmente a la instancia de la clase `MediaPlayer`. En caso contrario el vídeo no se mostrará. Además, la clase `MediaPlayer` requiere que la superficie sea un objeto de tipo `SurfaceHolder`.

Un ejemplo de objeto `SurfaceHolder` podría ser la vista `SurfaceView`, que podremos añadir al XML del layout correspondiente:


```xml
<SurfaceView
	android:id="@+id/superficie"
	android:layout_width="200dip"
	android:layout_height="200dip"
	android:layout_gravity="center">
</SurfaceView>
```

El siguiente paso será la inicialización el objeto `SurfaceView` y la asignación del mismo a la instancia de la clase `MediaPlayer` encargada de reproducir el vídeo. El siguiente código muestra cómo hacer esto. Obsérvese que es necesario que la actividad implemente la interfaz `SurfaceHolder.Callback`. Esto es así porque los objetos de la clase `SurfaceHolder` se crean de manera asíncrona, por lo que debemos añadir un mecanismo que permita esperar a que dicho objeto haya sido creado antes de poder reproducir el vídeo.


```java
public class MiActividad extends Activity implements SurfaceHolder.Callback
{
	private MediaPlayer mediaPlayer;

	@Override
	public void onCreate(Bundle savedInstanceState) {
		super.onCreate(savedInstanceState);
		setContentView(R.layout.main);
		mediaPlayer = new MediaPlayer();
		SurfaceView superficie = (SurfaceView)findViewById(R.id.superficie);
		// Obteniendo el objeto SurfaceHolder a partir del SurfaceView
		SurfaceHolder holder = superficie.getHolder();
		holder.addCallback(this);
		holder.setType(SurfaceHolder.SURFACE_TYPE_PUSH_BUFFERS);
	}

	// Este manejador se invoca tras crearse la superficie, momento
	// en el que podremos trabajar con ella
	public void surfaceCreated(SurfaceHolder holder) {
		try {
			mediaPlayer.setDisplay(holder);
		} catch (IllegalArgumentException e) {
			Log.d("MEDIA_PLAYER", e.getMessage());
		} catch (IllegalStateException e) {
			Log.d("MEDIA_PLAYER", e.getMessage());
		}
	}

	// Y este manejador se invoca cuando se destruye la superficie,
	// momento que podemos aprovechar para liberar los recursos asociados
	// al objeto MediaPlayer
	public void surfaceDestroyed(SurfaceHolder holder) {
		mediaPlayer.release();
	}

	public void surfaceChanged(SurfaceHolder holder, int format, int width, int height) { }
	}
}
```

Una vez que hemos asociado la superficie al objeto de la clase `MediaPlayer` debemos asignar a dicho objeto el clip de vídeo a reproducir. Ya que habremos creado el objeto `MediaPlayer` previamente, la única posibilidad que tendremos será utilizar el método `setDataSource`, como se muestra en el siguiente ejemplo. Recuerda que cuando se utiliza dicho método es necesario llamar también al método `prepare`.


```java
public void surfaceCreated(SurfaceHolder holder) {
	try {
		mediaPlayer.setDisplay(holder);
		mediaPlayer.setDataSource("/mnt/sdcard/DCIM/video.mp4");
		mediaPlayer.prepare();
		mediaPlayer.start();
	} catch (IllegalArgumentException e) {
		Log.d("MEDIA_PLAYER", e.getMessage());
	} catch (IllegalStateException e) {
		Log.d("MEDIA_PLAYER", e.getMessage());
	} catch (IOException e) {
		Log.d("MEDIA_PLAYER", e.getMessage());
	}
}
```



## Ejercicios



### Reproducir de un clip de audio

Se te proporciona en las plantillas de la sesión la aplicación _Audio_. Dicha aplicación contiene un clip de audio almacenado en los recursos, cuyo nombre
es _zelda_nes.mp3_. La aplicación contiene una única actividad con una serie de botones, a los que añadiremos funcionalidad para poder controlar la reproducción del clip de audio.

Se te pide hacer lo siguiente:

* Añade el código necesario en el constructor para crear una instancia de la clase `MediaPlayer` (donde el objeto `mp` habrá sido declarado como un
atributo de la clase):
```java
mp = MediaPlayer.create(this, R.raw.zelda_nes);
```

* Modifica el manejador del botón _Reproducir_ para que cada vez que se pulse éste se deshabilite, se habiliten los botones _Pausa_ y _Detener_, y empiece a reproducirse el clip de audio mediante la invocación del método `start` del objeto `MediaPlayer`.
> Para habilitar o deshabilitar botones usaremos el método `setEnabled`, que recibe como parámetro un booleano.


* Modifica el manejador del botón _Detener_ para que cada vez que se pulse se deshabilite dicho botón y el botón _Pausa_, se habilite el botón _Reproducir_, y se
detenga la reproducción del audio mediante
el método `stop`. Invoca también el método `prepare` del objeto `MediaPlayer` para dejarlo todo preparado por si se desea reproducir de nuevo el audio.

* Modifica el manejador del botón _Pausa_. Si el audio estaba en reproducción el texto del botón pasará a ser _Reanudar_ y se pausará la reproducción por medio
del método _pause_. Si ya estaba en pausa el texto del botón volverá a ser _Pausa_ y se reanudará la reproducción del audio. No olvides cambiar la etiqueta del botón a _Pausa_ si se pulsa el botón _Detener_.

* Observa que cuando detienes la reproducción con `stop` y la reanudas con `start` el archivo de audio continúa reproduciéndose en el punto donde se detuvo. ¿Qué tendrías que hacer para que al pulsar el botón `Reproducir` la reproducción comenzara siempre por el principio?

* Libera los recursos asociados al objeto `MediaPlayer` en el método `onDestroy`. No olvides invocar al método `onDestroy` de la superclase.




### Evento de finalización de la reproducción

El objeto `MediaPlayer` no pasa automáticamente al estado de detenido una vez que se reproduce completamente el clip de audio, sino que es necesario detener la reproducción a mano. Por lo tanto, en este ejercicio vamos a añadir un manejador para el evento que se dispara cuando el clip de audio finaliza. Añade el siguiente código en el método `onCreate`, tras la inicialización del objeto `MediaPlayer`:

```java
mp.setOnCompletionListener(new OnCompletionListener() {
			public void onCompletion(MediaPlayer mp) {
				// TODO Rellena con tu código

			}
        });
```

Lo que tiene que hacer este manejador es exactamente lo mismo que se debería hacer en el caso de haber pulsado el botón de `Detener`, es decir, habilitar el botón _Reproducir_, deshabilitar el resto, e invocar al método `stop`.



### Reproducir un clip de vídeo usando VideoView

Para este ejercicio se te proporciona en las plantillas el proyecto _Video_, que contiene una única actividad para controlar la reproducción de un clip de video, que se incluye en los recursos del proyecto (fichero `/res/raw/tetris.3gp`). En este caso tendremos un botón etiquetado como _Reproducir_, un botón etiquetado como _Detener_ y una vista de tipo _TextView_ que usaremos para mostrar la duración del vídeo, el cual se mostrará en una vista de tipo `VideoView`.

Los pasos que debes seguir son los siguientes:


* Modifica el manejador del botón _Reproducir_ para que cuando éste se pulse se deshabilite y se habilite el botón _Detener_.
* Modifica el manejador del botón _Detener_ para que cuando éste se pulse se deshabilite y se habilite el botón _Reproducir_.
* Prepara el vídeo a reproducir al pulsar el botón _Reproducir_ con la siguiente línea de código (pero no utilices el método `start` para reproducirlo todavía):
```java
superficie.setVideoUri(Uri.parse("android.resource://es.ua.jtech.android.video/" + R.raw.tetris));
```


* Detén la ejecución del vídeo cuando se pulse el botón _Detener_ por medio del método `stopPlayback`.
* Para poder reproducir el vídeo y poder obtener su duración y mosrtrarla en el `TextView`, hemos de esperar a que la superficie donde se va a reproducir esté preparada. Para ello hemos de implementar el manejador para el evento `OnPreparedListener` de la vista `VideoView`. Añade el manejador:
```java
superficie.setOnPreparedListener(new OnPreparedListener() {
	public void onPrepared(MediaPlayer mp) {
		// Tu código aquí
	}
});
```


* Añade código en el manejador anterior para comenzar la reproducción por medio del método `start`. Obtén la duración en minutos y segundos por medio del método
`getDuration` del `VideoView` y muéstrala en el `TextView` con el formato _mm:ss_.
> El método `getDuration` devuelve la duración del vídeo en milisegundos.
