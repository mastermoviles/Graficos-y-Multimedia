# Reproducción en dispositivos externos

Existen diferentes sistemas que nos permiten reproducir los contenidos multimedia de nuestros dispositivos en dispositivos externos, tales como pantallas de TV u ordenadores. Encontramos diferentes alternativas, como _Chromecast_, _Miracast_ y _Airplay_.

_Airplay_ es un protocolo propietario de Apple que en principio se diseñó para transmitir contenidos multimedia desde iPhone o iPad al dispositivo Apple TV, aunque actualmente puede utilizarse para enviar contenidos entre diferentes tipos de dispositivos conectados a la misma red.

_Chromecast_ es un dispositivo fabricado por Google que se conecta al puerto HDMI del televisor y nos permite reproducir en él contenidos multimedia enviados desde distintos tipos de dispositivos móviles (Android e iOS).

Encontramos también otros protocolos para esta misma funcionalidad, como por ejemplo _Miracast_, que está presente en determinados televisores. También podemos encontrar dispositivos HDMI que soportan todos los protocolos anteriores.

En cualquiera de estos casos, estas tecnologías nos permitirán visualizar en otras pantallas las fotos o vídeos almacenados en el móvil, reproducir audio, o incluso mostrar lo mismo que se esté visualizando en la pantalla del móvil (_mirroring_).

Existen dos formas básicas de operar:

* **Reproducción remota**: El dispositivo externo funciona de forma independiente. Nosotros simplemente le indicamos lo que debe reproducir desde el móvil, y él se encarga de gestionar el acceso y la reproducción del medio. Así puede funcionar por ejemplo la aplicación _You Tube_ con pantallas externas configuradas. Envía al dispositivo externo la indicación del vídeo que queremos reproducir, y el dispositivo se encarga de conectarse y reproducir el vídeo.
* **Retransmisión de medios**: El dispositivo móvil se encarga de retransmitir vía _streaming_ los contenidos al dispositivo remoto secundario. Este dispositivo simplemente reproducirá lo que esté recibiendo desde el móvil, sin autonomía propia.

Vamos a ver a continuación cómo preparar nuestras aplicaciones para que soporten estas tecnologías. Hablaremos de _enrutado de medios_ para referirnos al camino de salida que tomarán para su reproducción (altavoz o pantalla del móvil, auriculares, pantalla externa, etc).

## Enrutado de medios en Android

En Android podremos definir la forma en la que realizar el _enrutado de medios_ mediante la clase `MediaRouter`. Las aplicaciones _enrutarán_ los medios a través de un objeto `MediaRouter`, y éste se conectará con un `MediaRouteProvider` (que puede representar un dispositivos _Chromecast_, _Miracast_, etc).

### Selección de ruta

Para utilizar esta API deberemos utilizar la librería de compatibilidad v7. Las aplicaciones que utilicen _enrutado de medios_ deberán incluir en el _Action Bar_ un botón para seleccionar la ruta de salida. Para ello incluiremos el siguiente _item_ en el menú, indicando que deberá aparecer siempre en la barra como _acción_:

```xml
<?xml version="1.0" encoding="utf-8"?>
<menu xmlns:android="http://schemas.android.com/apk/res/android"
      xmlns:app="http://schemas.android.com/apk/res-auto"
      >

    <item android:id="@+id/media_route_menu_item"
        android:title="@string/media_route_menu_title"
        app:actionProviderClass="android.support.v7.app.MediaRouteActionProvider"
        app:showAsAction="always"
    />
</menu>
```

> Este botón puede también ser añadido de _forma alternativa_ mediante un objeto `MediaRouteButton`.

Una vez definido el botón, deberemos hacer que al pulsar sobre él se muestre un cuadro selector de ruta. Para ello deberemos crear un objeto `MediaRouteSelector` en el que deberemos indicar el tipo de dispositivos con los que podemos conectar:

```java
mSelector = new MediaRouteSelector.Builder()
        .addControlCategory(MediaControlIntent.CATEGORY_LIVE_AUDIO)
        .addControlCategory(MediaControlIntent.CATEGORY_LIVE_VIDEO)
        .addControlCategory(MediaControlIntent.CATEGORY_REMOTE_PLAYBACK)
        .build();
```

Esto podemos hacerlo por ejemplo en el método `onCreate`.

Las posibles categorías de dispositivos externos son:

* `CATEGORY_LIVE_AUDIO`: Sistemas de sonido que reciben el audio vía _streaming_
* `CATEGORY_LIVE_VIDEO`: Sistemas de vídeo que reciben el contenido vía _streaming_
* `CATEGORY_REMOTE_PLAYBACK`: Dispositivos que pueden realizar la reproducción de forma autónoma.

Por último, deberemos añadir este selector al botón del menú:

```java
MediaRouteSelector mSelector;

@Override
public boolean onCreateOptionsMenu(Menu menu) {
    super.onCreateOptionsMenu(menu);

    getMenuInflater().inflate(R.menu.sample_media_router_menu, menu);

    MenuItem mediaRouteMenuItem = menu.findItem(R.id.media_route_menu_item);
    MediaRouteActionProvider mediaRouteActionProvider =
            (MediaRouteActionProvider)MenuItemCompat.getActionProvider(
            mediaRouteMenuItem);
    mediaRouteActionProvider.setRouteSelector(mSelector);

    return true;
}
```

> Para que el botón de selección de ruta se muestre deberemos tener un dispositivo compatible conectado en nuestra red.

### Conexión con la ruta seleccionada

Para conectar con una ruta en primer lugar debemos crear un objeto `MediaRouter` que es un objeto _singleton_ que debe ser retenido por nuestra actividad:

```java
private MediaRouter mMediaRouter;

@Override
protected void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    setContentView(R.layout.activity_main);

    mMediaRouter = MediaRouter.getInstance(this);
}
```

Necesitaremos proporcionar a este objeto un _callback_ de tipo `MediaRouter.Callback` para responder ante los eventos de _enrutamiento_. Al menos deberemos definir los siguientes métodos en el _callback_:

```java
private final MediaRouter.Callback mMediaRouterCallback =
        new MediaRouter.Callback() {

    @Override
    public void onRouteSelected(MediaRouter router, RouteInfo route) {
        // Se establece la conexión a un dispositivo de salida
    }

    @Override
    public void onRouteUnselected(MediaRouter router, RouteInfo route) {
        // Se ha desconectado de un dispositivo de salida
    }

    @Override
    public void onRoutePresentationDisplayChanged(
            MediaRouter router, RouteInfo route) {
        // Hay un cambio de display en la nueva salida
        // (por ejemplo distinto tamaño de pantalla)
    }
}
```


Podemos añadir el _callback_ en `onStart` y eliminarlo en `onStop`:

```java
@Override
public void onStart() {
    mMediaRouter.addCallback(mSelector, mMediaRouterCallback,
            MediaRouter.CALLBACK_FLAG_REQUEST_DISCOVERY);
    super.onStart();
}

@Override
public void onStop() {
    mMediaRouter.removeCallback(mMediaRouterCallback);
    super.onStop();
}
```


### Reproducción remota

Si queremos utilizar el enfoque de reproducción remota, en primer lugar deberemos comprobar que el dispositivo externo soporta este modo de funcionamiento con:

```java
route.supportsControlCategory(
            MediaControlIntent.CATEGORY_REMOTE_PLAYBACK)
```

Esto lo podemos hacer desde cualquiera de los métodos del _callback_. En caso de no soportarlo deberemos utilizar en modo de retransmisión desde el dispositivo.

Una vez comprobado que tenemos soporte, podemos enviar comandos al dispositivo externo para controlar la reproducción (pausar, reanudar, parar, cambiar el volumen, etc).

Podemos enviar el comando para que el dispositivo remoto inicie la reproducción de un vídeo en internet de la siguiente forma:

```java
RouteInfo mRoute;
RemotePlaybackClient mRemotePlaybackClient;

private void reproducirRemoto(RouteInfo route) {

    // Si había una ruta anterior, antes la liberamos
    if (mRoute != null && mRemotePlaybackClient != null) {
        mRemotePlaybackClient.release();
        mRemotePlaybackClient = null;
    }

    mRoute = route;
    mRemotePlaybackClient = new RemotePlaybackClient(this, mRoute);

    // Envía el comando de reproducción
    mRemotePlaybackClient.play(Uri.parse(
            "http://jtech.ua.es/dadm/video.mp4"),
            "video/mp4", null, 0, null, new ItemActionCallback() {

            @Override
            public void onResult(Bundle data, String sessionId,
                    MediaSessionStatus sessionStatus,
                    String itemId, MediaItemStatus itemStatus) {
            }

            @Override
            public void onError(String error, int code, Bundle data) {
            }
        });
    }
}
```

Podemos utilizar el objeto `RemotePlaybackClient` para enviar diferentes comandos al dispositivo externo:

* `play()`
* `pause()`
* `resume()`
* `seek()`
* `release()`


### Emisión a un dispositivo secundario

Si el modo de reproducción remota no está disponible, podemos emitir directamente el medio desde nuestro dispositivo móvil. De esta forma el móvil se encarga de la reproducción, y envía el contenido al dispositivo externo.

Para ello necesitamos un objeto de tipo `Presentation`, que define un segundo _display_ donde mostraremos todo lo que queramos enviar al dispositivo remoto. Este objeto tiene su propio contexto y podemos incluir en el un _layout_, como si de una actividad se tratase. Podemos por ejemplo añadir en él un `SurfaceView`:

```java
public class PantallaSecundaria extends Presentation {
    public PantallaSecundaria(Context outerContext, Display display) {
        super(outerContext, display);
    }

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        Resources resources = getContext().getResources();

        setContentView(R.layout.pantalla_secundaria);

        mSurfaceView = (GLSurfaceView)findViewById(R.id.surface_view);
        ...
    }
}
```

Podemos mostrar la presentación en la ruta establecida con:

```java
private void emitirContenido(RouteInfo route) {

    // Obtiene el display de la ruta seleccionada
    Display selectedDisplay = null;
    if (route != null) {
        selectedDisplay = route.getPresentationDisplay();
    }

    // Cierra la presentacion actual si existe y tiene un display distinto al seleccionado
    if (mPresentation != null && mPresentation.getDisplay() != selectedDisplay) {
        mPresentation.dismiss();
        mPresentation = null;
    }

    // Muestra la presentación en el display seleccionado, si no lo estabamos haciendo ya
    if (mPresentation == null && selectedDisplay != null) {
        mPresentation = new PantallaSecundaria(this, selectedDisplay);
        mPresentation.setOnDismissListener(
                new DialogInterface.OnDismissListener() {
                    @Override
                    public void onDismiss(DialogInterface dialog) {
                        if (dialog == mPresentation) {
                            mPresentation = null;
                        }
                    }
                });

        try {
            mPresentation.show();
        } catch (WindowManager.InvalidDisplayException ex) {
            mPresentation = null;
        }
    }
}
```

Podemos utilizar el _callback_ del `MediaRouter` para comprobar el modo de emisión soportado e iniciar la reproducción por el método adecuado:

```java
private final MediaRouter.Callback mMediaRouterCallback =
        new MediaRouter.Callback() {

    @Override
    public void onRouteSelected(MediaRouter router, RouteInfo route) {
        if (route.supportsControlCategory(
            MediaControlIntent.CATEGORY_REMOTE_PLAYBACK)){
            reproducirRemoto(route);
        } else {
            emitirContenido(route);
        }
    }

    @Override
    public void onRouteUnselected(MediaRouter router, RouteInfo route) {
    }

    @Override
    public void onRoutePresentationDisplayChanged(
            MediaRouter router, RouteInfo route) {
    }
}
```


## _AirPlay_ en iOS

En iOS el contenido reproducido por _AVFoundation_ tiene por defecto soporte para _AirPlay_, y se enruta automáticamente al dispositivo externo seleccionado actualmente.

Para hacer posible la reproducción por AirPlay deberemos tener activado el modo _Audio, AirPlay and Picture in Picture_ en _Background Modes_, desde la pestaña Capabilities de nuestro _target_, como vimos anteriormente. 

Esto será suficiente cuando lo que necesitemos sea emitir vídeo o audio al dispositivo externo, pero también podremos utilizar AirPlay para mostrar contenido propio en una segunda pantalla. Veremos a continuación con más detalle estas dos formas de trabajar con AirPlay.

### Selección de enrutamiento con AirPlay

Podemos enviar mediante AirPlay audio o vídeo almacenado localmente en el dispositivo, o al que accedemos de forma remota mediante _HTTP Live Streaming_ o descarga progresiva, siendo _HTTP Live Streaming_ el sistema preferido. En cualquier caso, siempre se debe acceder al contenido mediante HTTP. El audio y vídeo debe estar codificado en formatos compatibles con AirPlay: vídeo H.264 y audio AAC o MP3.

AirPlay está por defecto activado cuando reproducimos medios con MoviePlayer o con AVFoundation. Para poder seleccionar el dispositivo de salida del audio o el vídeo podemos introducir un botón predefinido que nos mostrará la lista de posibles dispositivos. Esto lo haremos creando un objeto de tipo `MPVolumeView` y añadiéndolo a la vista donde queremos mostrarlo:

```objectivec
MPVolumeView *volumeView = [ [MPVolumeView alloc] init];
[view addSubview: volumeView];
```

Este componente mostrará un control de volumen y además el botón para seleccionar la salida. Si sólo queremos mostrar el botón podemos ocultar la barra de volumen de la siguiente forma:

```objectivec
MPVolumeView *volumeView = [ [MPVolumeView alloc] init];
[volumeView setShowsVolumeSlider:NO];
[volumeView sizeToFit];
[view addSubview: volumeView];
```

También es posible añadir el botón anterior como botón de la barra de navegación o de herramientas, de la siguiente forma:

```objectivec
MPVolumeView *volumeView = [ [MPVolumeView alloc] init];
[volumeView setShowsVolumeSlider:NO];
[volumeView sizeToFit];

UIBarButtonItem *barButton = [[UIBarButtonItem alloc] initWithCustomView:volumeView];
self.navigationItem.rightBarButtonItem = barButton;
```

### AirPlay y reproducción de vídeo

La emisión de vídeo por AirPlay estará activa por defecto tanto cuando utilicemos `AVPlayer` como `MPMoviePlayerController`. Podemos desactivarlo o volverlo a activar con las siguientes propiedades:

```objectivec
AVPlayer *player;
MPMoviePlayerController moviePlayerController;

...

moviePlayerController.allowsAirPlay = NO;
player.allowsAirPlayVideo = NO;
```


### Pantalla secundaria

Vamos a ver ahora cómo manejar pantallas secundarias. Cuando tengamos nuestro dispositivo conectado a una pantalla secundaria por _AirPlay_, podremos o bien mostrar lo mismo que estamos mostrando en el dispositivo (_mirroring_), o contenido distinto. Por defecto tendremos el _mirroring_ activado cuando nos conectemos por _AirPlay_ a una pantalla secundaria. Vamos a estudiar cómo enviar un contenido diferente.

Para que el contenido en la pantalla secundaria sea distinto al de la pantalla del móvil deberemos crearnos un objeto de tipo `UIWindow` para ella:

```objectivec
@property (nonatomic, strong) UIWindow *ventanaSecundaria;
```

Comprobaremos si hay más de una pantalla, y en tal caso crearemos nuestra nueva ventana (`UIWindow`) a partir de ella:

```objectivec
if ([[UIScreen screens] count] > 1)
{
    UIScreen *pantallaSecundaria = [[UIScreen screens] objectAtIndex:1];

    self.ventanaSecundaria = [[UIWindow alloc] initWithFrame:pantallaSecundaria.bounds];
    self.ventanaSecundaria.screen = pantallaSecundaria;

    self.ventanaSecundaria.hidden = NO;
}
```

> Es importante asociar la nueva ventana a la pantalla secundaria antes de mostrarla

Con esto ya se mostrará el contenido que añadamos a dicha ventana en nuestro dispositivo externo. Podremos volver a activar el _mirroring_ simplemente eliminando la ventana secundaria de la pantalla secundaria.

### Eventos de la pantalla secundaria

Es importante estar al tanto de los eventos de conexión y desconexión de pantallas secundarias. Esto podemos hacerlo escuchando las siguientes notificaciones:


```objectivec
NSNotificationCenter *center = [NSNotificationCenter defaultCenter];
[center addObserver:self selector:@selector(pantallaConectada:)
            name:UIScreenDidConnectNotification object:nil];
[center addObserver:self selector:@selector(pantallaDesconectada:)
            name:UIScreenDidDisconnectNotification object:nil];
}
```

Podemos crear la ventana secundaria cuando se conecte una segunda pantalla, y eliminarla cuando se desconecte:


```objectivec
- (void)pantallaConectada:(NSNotification*)aNotification
{
    UIScreen *pantallaSecundaria = [aNotification object];

    if (!self.ventanaSecundaria)
    {
        self.ventanaSecundaria = [[UIWindow alloc] initWithFrame:pantallaSecundaria.bounds];
        self.ventanaSecundaria.screen = pantallaSecundaria;
    }
}

- (void)pantallaDesconectada:(NSNotification*)aNotification
{
    if (self.ventanaSecundaria)
    {
        self.ventanaSecundaria.hidden = YES;
        self.ventanaSecundaria = nil;
    }
}
```


## Aplicaciones para televisores

Además de poder emitir medios desde los dispositivos móviles a dispositivos externos, también podemos crear aplicaciones que se ejecuten de forma directa en la TV. Para ello contamos con dispositivos como Apple TV, con el sistema operativo tvOS, o los diferentes dispositivos que implementan Android TV. Estos sistemas añaden a las API que ya conocemos de iOS y Android respectivamente funcionalidades adicionales diseñadas de forma específica para pantallas de televisor. 

Vamos a ver a continuación una introducción a estos sistemas tvOS y Android TV.

### Apple TV y tvOS

El sistema operativo tvOS deriva de iOS, pero se trata de un operativo diferente. Veremos que el desarrollo será muy similar al que ya conocemos de iOS, podemos desarrollar una aplicación tvOS con Xcode, e incluso una misma aplicación puede destinarse a iOS y tvOS creando dos _targets_ distintos, y así reutilizar todas las clases que sean comunes. Una aplicación iOS se podrá normalmente ofrecer en tvOS sin tener que hacer cambios, simplemente añadiendo el _target_ correspondiente. La principal diferencia entre estos dos sistemas es que tvOS añade algunos nuevos _frameworks_ destinados a crear aplicaciones para TV.

Los principales _frameworks_ adicionales que encontramos en tvOS son:

* **TVMLKit**: Permite utilizar un lenguaje de marcado propio (TVML) y Javascript para crear la interfaz de las aplicaciones para TV. Este _framework_ nos proporciona el puente entre el código nativo de la aplicación y el código Javascript de la interfaz.
* **TVMLJS**: Describe las APIs Javascript utilizadas en la interfaz TVML.
* **TVServices**: Permite añadir una extensión a la _app_ para que cuando la seleccionemos en el menú de aplicaciones, nos aparezca en pantalla una previsualización de información destacada de la aplicación (por ejemplo contenidos recomendados). 

Algunas de las principales cuestiones a tener en cuenta cuando diseñemos una aplicación tvOS son:

* La interfaz ya no es táctil. Normalmente manejaremos el dispositivo con el mando a distancia (_Siri Remote_) o con un mando de videojuegos. 
* No es un dispositivo de uso personal. Los dispositivos móviles normalmente son personales, pero la TV es familiar. Deberemos tener esto en cuenta, por ejemplo creando diferentes perfiles como hacen por ejemplo aplicaciones como Netflix.
* El almacenamiento es limitado. Las aplicaciones pueden ocupar un máximo de 200Mb, y pueden almacenar un máximo de 500Kb de datos persistentes. Para poder tener aplicaciones más grandes, podemos utilizar _recursos bajo demanda_ en la _App Store_, es decir, paquetes de la aplicación que se descargarán cuando se necesiten (por ejemplo, niveles adicionales de un juego). Para el almacenamiento de información, se puede almacenar en el directorio de caché (se puede borrar de una ejecución a la siguiente), o en iCloud.

### Android TV

Además de aplicaciones para dispositivos vestibles, Android también nos permite crear aplicaciones para dispositivos de TV.

Las aplicaciones Android para televisores se crean utilizando la API 21 (Android 5.0). Se pueden utilizar además de forma opcional librerías de soporte para TV, como la librería _v17_ (_leanback_), que depende de las librerías _v4_ y _v7_, y nos proporciona elementos y estilos orientados a este tipo de pantallas.

Una actividad se marca como destinada a televisores mediante la categoría `LEANBACK_LAUNCHER`:

```xml
<activity
android:name="es.ua.eps.MainActivity"
android:label="@string/app_name" >

<intent-filter>
  <action android:name="android.intent.action.MAIN" />
  <category android:name="android.intent.category.LEANBACK_LAUNCHER" />
</intent-filter>
</activity>
```

Los televisores carecen de gran parte del _hardware_ con el que contamos en dispositivos móviles. Si queremos hacer una aplicación que sirva tanto para teléfonos móviles como para televisores, deberemos declarar que dichas características no son requeridas (pantalla táctil, cámara, teléfono, etc) para que de esta manera pueda instalarse en ambos tipos de dispositivos:

```xml
<uses-feature android:name="android.hardware.touchscreen"
          android:required="false" />
<uses-feature android:name="android.hardware.telephony"
        android:required="false"/>
<uses-feature android:name="android.hardware.camera"
        android:required="false"/>
<uses-feature android:name="android.hardware.nfc"
        android:required="false"/>
<uses-feature android:name="android.hardware.gps"
        android:required="false"/>
<uses-feature android:name="android.hardware.microphone"
        android:required="false"/>
```

Deberemos comprobar en código si dichas características están disponibles antes de intentar utilizarlas, para así conseguir aplicaciones compatibles con distintos tipos de dispositivos.

#### Navegación

La aplicación para TV deberá manejarse mediante un _pad_ direccional (un mando de videojuegos o un mando a distancia). Para ello es importante que en los botones proporcionemos diferentes aspectos según si está seleccionado, pulsado, o no pulsado, y que en el _layout_ especifiquemos de forma correcta el orden en el que recibirán el foco los distintos elementos de la interfaz..

#### Estilo

El aspecto estándar recomendado para las aplicaciones de TV es _leanback_, perteneciente a la librería v17:

```xml
<activity
  android:name="es.ua.eps.MiTVActividad"
  android:label="@string/app_name"
  android:theme="@style/Theme.Leanback">
```

En cualquier caso, será importante que el tema que utilicemos no tenga barra de título. Si queremos utilizar otro estilo como tema, deberemos siempre utilizar uno que no tenga barra de título ni _action bar_:

```xml
<activity
  android:name="es.ua.eps.MiTVActividad"
  android:label="@string/app_name"
  android:theme="@android:style/Theme.NoTitleBar">
```

#### Contenido

Los principales usos de las aplicaciones de TV son los videojuegos y la reproducción de medios. Dentro de la librería _v17_ encontramos diferentes componentes orientados a las aplicaciones de TV, como por ejemplo _browsers_ para el catálogo de medios y fichas para mostrar los metadatos de los medios en reproducción.

Para el _browser_ de medios se puede utilizar el tipo de fragmento `BrowseFragment`, que nos mostrará la lista de contenidos, y para ver los detalles de cada contenido tenemos `DetailsFragment`.

Podemos también mostrar una ficha con los datos de la reproducción actual mediente el objeto `MediaSession`. Con su método `setMetadata()` podremos introducir los metadatos del medio en reproducción y así aparecerán en la ficha de _reproducción actual_.




