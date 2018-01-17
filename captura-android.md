# Captura de medios en Android

En esta sesión nos centraremos en estudiar los métodos para capturar audio y vídeo desde un dispositivo Android. Las últimas versiones del SDK de Android permiten emular la captura de video y audio en nuestros dispositivos virtuales, lo cual facilitará las pruebas de este tipo de aplicaciones. En concreto, es posible realizar esta simulación por medio de una webcam, que se utilizará para captar lo que se supone que estaría captando la cámara del dispositivo real.

Existen dos formas básicas de capturar medios:

* Utilizar un _intent_ para llamar a la aplicación de captura y obtener el resultado que ésta nos proporcione.
* Crear nuestra propia aplicación de captura. Este método es más complejo pero nos permite personalizar la forma en la que se realiza la captura.

## Captura de medios mediante _intents_

La forma más sencilla de capturar medios es lanzando la aplicación nativa que realiza esta tarea mediante un _intent_. Al lanzar el _intent_ deberemos indicar mediante un parámetro \(_extra_\) el lugar donde queremos guardar el medio capturado. Deberemos seleccionar la ubicación adecuada según el uso que le queramos dar.

### Almacenamiento de medios

En cualquier caso los medios deberán ser almacenados en la tarjeta SD para no gastar el almacenamiento externo y permitir a los usuarios tener acceso a los medios capturados. Básicamente tenemos dos opciones:

* `Environment.getExternalStoragePublicDirectory(Environment.DIRECTORY_PICTURES)` indica el directorio externo donde almacenar los medios de forma pública e independiente de la aplicación. Se recomienda crear dentro de estas carpeta un subdirectorio por cada aplicación.

  > Este método sólo esta disponible a partir de Android 2.2 \(API 8\). Si queremos compatibilidad con APIs anteriores utilizaremos Environment.getExternalStorageDirectory\(\)

* `Context.getExternalFilesDir(Environment.DIRECTORY_PICTURES)` nos indica un directorio propio de la aplicación donde almacenar los medios. Si la aplicación se desinstala, todos los medios almacenados en este directorio se borrarán.

Para poder escribir en el soporte de almacenamiento externo deberemos solicitar el siguiente permiso:

```xml
<uses-permission android:name="android.permission.WRITE_EXTERNAL_STORAGE" />
```

En los dispositivos Android también contamos con un proveedor de contenido denominado _Media Store_. Este proveedor de contenidos consiste en una base de datos de medios almacenados en el dispositivo donde podremos registrar aquellos medios que capturemos desde nuestra aplicación, de forma que sean fácilmente accesibles desde cualquier otra aplicación del dispositivo. Más adelante veremos como registrar un fichero \(fotografía, audio o vídeo\) en esta base de datos.

### Toma de fotografías

En esta sección veremos cómo tomar fotografías desde nuestra aplicación y utilizar la imagen obtenida para realizar alguna tarea. Como veremos se tratará ni más ni menos que un ejemplo clarísimo de _intent_ implícito, en el que pediremos al sistema que se lance una actividad que pueda tomar fotografías. Por medio de este mecanismo de comunicación obtendremos la imagen capturada \(o una dirección a la localización de la misma en el dispositivo\) para trabajar con ellas.

> Hoy en día es posible simular la cámara del dispositivo virtual por medio de una webcam, así que no es necesario utilizar un dispositivo real para poder probar estos ejemplos.

La acción a solicitar mediante el `Intent` implícito será `MediaStore.ACTION_IMAGE_CAPTURE` \(más adelante hablaremos de la clase `MediaStore`\). Lanzaremos el `Intent` por medio del método `startActivityForResult`, con lo que en realidad estaremos haciendo uso de una subactividad. Recuerda que esto tenía como consecuencia que al terminar la subactividad se invoca el método `onActivityResult` de la actividad padre. En este caso el identificador que se le ha dado a la subactividad es `TAKE_PICTURE`, que se habrá definido como una constante en cualquier otro lugar de la clase:

```java
startActivityForResult(new Intent(MediaStore.ACTION_IMAGE_CAPTURE), TAKE_PICTURE);
```

Si no hemos hecho ningún cambio al respecto en nuestro sistema, esta llamada lanzará la actividad nativa para la toma de fotografías. No podemos evitar recordar una vez más la ventaja que esto supone para el desarrollador Android, ya que en lugar de tener que desarrollar una nueva actividad para la captura de imágenes desde cero, es posible hacer uso de los recursos del sistema.

Según los parámetros del `Intent` anterior, podemos hablar de dos modos de funcionamiento en cuanto a la toma de fotografías:

* **Modo thumbnail**: este es el modo de funcionamiento por defecto. El `Intent` devuelto como respuesta por la subactividad, al que podremos acceder
  desde `onActivityResult`, contendrá un parámetro extra de nombre `data`, que consistirá en un thumbnail de tipo `Bitmap`.
* **Modo de imagen completa**: la captura de imágenes se realizará de esta forma si se especifica una URI como valor del parámetro extra `MediaStore.EXTRA_OUTPUT` del `Intent` usado para lanzar la actividad de toma de fotografías. En este caso se guardará la imagen obtenida por la cámara, en su resolución completa, en el destino indicado en dicho parámetro extra.  En este caso el `Intent` de respuesta no se usará para devolver un thumbnail, y por lo tanto el parámetro extra `data` tendrá como valor `null`.

En el siguiente ejemplo tenemos el esqueleto de una aplicación en el que se utiliza un `Intent` para tomar una fotografía, ya sea en modo thumbnail o en modo de imagen completa. Según queramos una cosa o la otra deberemos llamar a los métodos `getThumbnailPicture` o `saveFullImage`, respectivamente. En `onActivityResult` se determina el modo empleado examinando el valor del campo extra `data` del `Intent` de respuesta. Por último, una vez tomada la fotografía, se puede almacenar en el _Media Store_ \(hablamos de esto un poco más adelante\) o procesarla dentro de nuestra aplicación antes de descartarla.

```java
private static int TAKE_PICTURE = 1;
private Uri ficheroSalidaUri;

private void getThumbailPicture() {
    Intent intent = new Intent(MediaStore.ACTION_IMAGE_CAPTURE);
    startActivityForResult(intent, TAKE_PICTURE);
}

private void saveFullImage() {
    Intent intent = new Intent(MediaStore.ACTION_IMAGE_CAPTURE);
    File file = new File(Environment.getExternalStorageDirectory(), "prueba.jpg");
    ficheroSalidaUri = Uri.fromFile(file);
    intent.putExtra(MediaStore.EXTRA_OUTPUT, ficheroSalidaUri);
    startActivityForResult(intent, TAKE_PICTURE);
}

@Override
protected void onActivityResult(int requestCode, int resultCode, Intent data) {
    if (requestCode == TAKE_PICTURE) {
        Uri imagenUri = null;
        // Comprobamos si el Intent ha devuelto un thumbnail
        if (data != null) {
            if (data.hasExtra("data")) {
                Bitmap thumbnail = data.getParcelableExtra("data");
                // HACER algo con el thumbnail
            }
        }
        else {
             // HACER algo con la imagen almacenada en ficheroSalidaUri
        }
    }
}
```

### Captura de vídeo

La manera más sencilla de comenzar a grabar vídeo es mediante la constante `ACTION_VIDEO_CAPTURE` definida en la clase `MediaStore`, que deberá utilizarse conjuntamente con un `Intent` que se pasará como parámetro a `startActivityForResult`:

```java
startActivityForResult(new Intent(MediaStore.ACTION_VIDEO_CAPTURE), GRABAR_VIDEO);
```

Esto lanzará la aplicación nativa de grabación de vídeos en Android, permitiendo al usuario comenzar o detener la grabación, revisar lo que se ha grabado, y volver a comenzar la grabación en el caso en el que se desee. La ventaja como desarrolladores será la misma de siempre: al utilizar el componente nativo nos ahorramos el tener que desarrollar una actividad para la captura de vídeo desde cero.

La acción de captura de vídeo que se pasa como parámetro al `Intent` acepta dos párametros extra opcionales, cuyos identificadores se definen como constantes en la clase `MediaStore`:

* `EXTRA_OUTPUT`: por defecto el vídeo grabado será guardado en el _Media Store_. Para almacenarlo en cualquier otro lugar indicaremos una URI como parámetro extra utilizando este identificador.
* `EXTRA_VIDEO_QUALITY`: mediante un entero podemos especificar la calidad del vídeo capturado. Sólo hay dos valores posibles: 0 para tomar vídeos en baja resolución
  y 1 para tomar vídeos en alta resolución \(este último valor es el que se toma por defecto\).

A continuación se puede ver un ejemplo en el que se combinan todos estos conceptos vistos hasta ahora:

```java
private static int GRABAR_VIDEO = 1;
private static int ALTA_CALIDAD = 1;
private static int BAJA_CALIDAD = 0;

private void guardarVideo(Uri uri) {
    Intent intent = new Intent(MediaStore.ACTION_VIDEO_CAPTURE);

    // Si se define una uri se especifica que se desea almacenar el
    // vídeo en esa localización. En caso contrario se hará uso
    // del Media Store
    if (uri != null)
        intent.putExtra(MediaStore.EXTRA_OUTPUT, output);

    // En la siguiente línea podríamos utilizar cualquiera de las
    // dos constantes definidas anteriormente: ALTA_CALIDAD o BAJA_CALIDAD
    intent.putExtra(MediaStore.EXTRA_VIDEO_QUALITY, ALTA_CALIDAD);

    startActivityForResult(intent, GRABAR_VIDEO);
}

@Override
protected void onActivityResult(int requestCode, int resultCode, Intent data) {
    if (requestCode == GRABAR_VIDEO) {
        Uri videoGrabado = data.getData();
        // Hacer algo con el vídeo
    }
}
```

## Captura de medios desde nuestra actividad

En el caso en el que se desee reemplazar la aplicación nativa de captura o se quiera tener más control sobre la grabación será posible utilizar la clase `MediaRecorder`.

Para poder capturar audio y video desde nuestra propia actividad, sin lanzar la aplicación nativa, será necesario incluir los siguientes permisos en el _manifest_:

```xml
<uses-permission android:name="android.permission.CAMERA"/>
<uses-permission android:name="android.permission.RECORD_AUDIO"/>
```

### La cámara

Tanto si queremos tomar fotografías como grabar vídeo necesitaremos tener acceso a la cámara del dispositivo. Para ello deberemos solicitar el permiso `CAMERA`:

```xml
<uses-permission android:name="android.permission.CAMERA" />
```

También deberemos indicar en el _manifest_ que la aplicación utiliza la característica de la cámara:

```xml
<uses-feature android:name="android.hardware.camera" />
```

Podemos especificar diferentes requerimientos de _hardware_ de la cámara para nuestra aplicación:

* En el caso de que la aplicación pueda utilizar la cámara, pero no sea obligatorio contar con una cámara para poder ejecutar nuestra aplicación, podremos indicar que dicha característica no es obligatoria con:

  ```xml
  <uses-feature android:name="android.hardware.camera" android:required="false" />
  ```

  Dentro del código de la aplicación, podremos detectar si la cámara está presente con:

  ```java
  boolean tieneCamara = context.getPackageManager().hasSystemFeature(PackageManager.FEATURE_CAMERA);
  ```

* Actualmente es habitual encontrar dispositivos que incorporan más de una cámara. Si necesitamos que el dispositivo cuente con cámara frontal se puede especificar el siguiente requerimiento:

  ```xml
  <uses-feature android:name="android.hardware.camera.front" />
  ```

  Podemos consultar el número de cámaras con las que cuenta el dispositivo con:

  ```java
  int numCamaras = Camera.getNumberOfCameras();
  ```

* También podemos solicitar otras características, como que cuente con _autofocus_ o _flash_.

Para acceder a la cámara utilizaremos el método `Camera.open()`. Este método abrirá la cámara principal del dispositivo, si queremos utilizar otra cámara podremos utilizar `Camara.open(int numCamara)`.

Podemos utilizar el objeto `Camera` para mostrar la previsualización de la cámara mientras hacemos captura de foto o vídeo. Para ello necesitaremos una vista de tipo `SurfaceView`:

```java
public class CameraPreview extends SurfaceView
                        implements SurfaceHolder.Callback {
    private SurfaceHolder mHolder;
    private Camera mCamera;

    public CameraPreview(Context context, Camera camera) {
        super(context);
        mCamera = camera;

        mHolder = getHolder();
        mHolder.addCallback(this);
    }

    public void surfaceCreated(SurfaceHolder holder) {
              // Pone en marcha el preview de la camara
        try {
            mCamera.setPreviewDisplay(holder);
            mCamera.startPreview();
        } catch (IOException e) { }
    }

    public void surfaceDestroyed(SurfaceHolder holder) {
    }

    public void surfaceChanged(SurfaceHolder holder, int format, int w, int h) {
       if (mHolder.getSurface() != null){

                    // Detiene el preview con el formato anterior
            try {
                mCamera.stopPreview();
            } catch (Exception e){ }

                    // Reanuda el preview con el nuevo formato
            try {
                mCamera.setPreviewDisplay(mHolder);
                mCamera.startPreview();
            } catch (Exception e){ }
             }
    }
}
```

Podemos utilizar la vista anterior en una actividad:

```java
public class CapturaActivity extends Activity {

    private Camera mCamera;
    private CameraPreview mPreview;

    @Override
    public void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
                try {
                    mCamera = Camera.open();
                    mPreview = new CameraPreview(this, mCamera);
                    setContentView(mPreview);
                } catch(Exception e) {
                    // No tiene acceso a la camara
                }
    }
}
```

Dentro de esta actividad podremos capturar fotos directamente a través del objeto `Camera` mientras se previsualiza, o capturar vídeo utilizando un objeto `MediaRecorder`.

Es importante liberar la cámara cuando no se vaya a utilizar más con `Camera.release()`. Un lugar adecuado para hacer esto puede ser el método `onDestroy()` de la actividad:

```java
        @Override
        public void onDestroy() {
                super.onDestroy();
                mCamera.release();
        }
```

### Captura de fotografías

Podemos utilizar el objeto `Camera` para tomar fotografías. Para ello llamaremos al método `takePicture` pasando como parámetro un _listener_ de tipo `PictureCallback`, al que se llamará cuando la imagen haya sido tomada:

```java
mCamera.takePicture(null, null, new PictureCallback() {

    @Override
    public void onPictureTaken(byte[] data, Camera camera) {
                // Grabar data en directorio de medios
                ...
    }
});
```

El primer parámetro de `takePicture` es opcional y nos permite definir un _callback_ para el obturador. El segundo y tercer parámetro sirve para especificar un _callback_ para guardar la imagen en formato RAW y JPEG respectivamente.

Por ejemplo, podríamos guardar la imagen JPEG en el almacenamiento externo con:

```java
try {
    File file = new File(Environment.getExternalStorageDirectory(),
                             "prueba.jpg");
    FileOutputStream fos = new FileOutputStream(file);
    fos.write(data);
    fos.close();
} catch (Exception e) { }
```

### API de Camara 2

A partir de Android 5.0 \(_Lollipop_\) aparece una nueva versión de API de cámara \(`android.hardware.camera2`\), que nos permite tener un mayor control sobre este dispositivo, pasando la antigua API a estar desaprobada. Sin embargo, la nueva API no es compatible con versiones anteriores de Android, por lo que si queremos mantener la compatibilidad con versiones de Android anteriores a la 5.0 \(API 21\) deberemos utilizar la antigua cámara, o bien código separado para cada versión:

```java
if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.LOLLIPOP) {
    // Camara 2
} else {
    // Camara 1
}
```

La nueva API de cámara deja desaprobada la clase `Camera`, y en su lugar utilizar `CameraDevice`. Para acceder a las cámaras disponibles se proporciona la clase `CameraManager`.

### Captura de vídeo con `MediaRecorder`

Podemos utilizar la clase `MediaRecorder` para capturar audio o video desde nuestras propias actividades. La creación de un objeto de esta clase es sencilla:

```java
MediaRecorder mediaRecorder = new MediaRecorder();
```

La clase `MediaRecorder` permite especificar el origen del audio o vídeo, el formato del fichero de salida y los _codecs_ a utilizar. Como en el caso de la clase `MediaPlayer`, la clase `MediaRecorder` maneja la grabación mediante una máquina de estados. Esto quiere decir que el orden en el cual se inicializa y se realizan operaciones con los objetos de este tipo es importante. En resumen, los pasos para utilizar un objeto `MediaRecorder` serían los siguientes:

* Crear un nuevo objeto `MediaRecorder`.
* Asignarle las fuentes a partir de las cuales grabar el contenido.
* Definir el formato de salida.
* Especificar las características del vídeo: _codec_, _framerate_ y resolución de salida.
* Seleccionar un fichero de salida.
* Prepararse para la grabación.
* Realizar la grabación.
* Terminar la grabación.

Una vez finalizamos la grabación hemos de hacer uso del método `release` del objeto `MediaRecorder` para liberar todos sus recursos asociados:

```java
mediaRecorder.release();
```

### Configurando y controlando la grabación de vídeo

Como se ha indicado anteriormente, antes de grabar se deben especificar la fuente de entrada, el formato de salida, el codec de audio o vídeo y el fichero de salida, en ese estricto orden.

Los métodos `setAudioSource` y `setVideoSource` permiten especificar la fuente de datos por medio de constantes estáticas definidas en  `MediaRecorder.AudioSource` y `MediaRecorder.VideoSource`, respectivamente. El siguiente paso consiste en especificar el formato de salida por medio del método `setOutputFormat` que recibirá como parámetro una constante entre las definidas en `MediaRecorder.OutputFormat`. A continuación usamos el método `setAudioEnconder` o `setVideoEncoder` para especificar el codec usado para la grabación, utilizando alguna de las constantes definidas en `MediaRecorder.AudioEncoder` o `MediaRecorder.VideoEncoder`, respectivamente. Es en este punto en el que podremos definir el framerate o la resolución de salida si se desea. Finalmente indicamos la localización del fichero donde se guardará el contenido grabado por medio del método `setOutputFile`. El último paso antes de la grabación será la invocación del método `prepare`.

El siguiente código muestra cómo configurar un objeto `MediaRecorder` para capturar audio y vídeo del micrófono y la cámara usando un codec estándar y grabando el resultado en la tarjeta SD:

```java
MediaRecorder mediaRecorder = new MediaRecorder();

mCamera.unlock();
mediaRecorder.setCamera(mCamera);

// Configuramos las fuentes de entrada
mediaRecorder.setAudioSource(MediaRecorder.AudioSource.MIC);
mediaRecorder.setVideoSource(MediaRecorder.VideoSource.CAMERA);

// Seleccionamos el formato de salida
mediaRecorder.setOutputFormat(MediaRecorder.OutputFormat.DEFAULT);

// Seleccionamos el codec de audio y vídeo
mediaRecorder.setAudioEncoder(MediaRecorder.AudioEncoder.DEFAULT);
mediaRecorder.setVideoEncoder(MediaRecorder.VideoEncoder.DEFAULT);

// Especificamos el fichero de salida
mediaRecorder.setOutputFile("/mnt/sdcard/mificherodesalida.mp4");

// Nos preparamos para grabar
mediaRecorder.prepare();
```

> Recuerda que los métodos que hemos visto en el ejemplo anterior deben invocarse en ese orden concreto, ya que de lo contrario se lanzará una excepción de tipo  
> _Illegal State Exception_.

Para comenzar la grabación, una vez inicializados todos los parámetros, utilizaremos el método `start`:

```java
mediaRecorder.start();
```

Cuando se desee finalizar la grabación se deberá hacer uso en primer lugar del método `stop`, y a continuación invocar el método `reset`. Una vez seguidos estos pasos es posible volver a utilizar el objeto invocando de nuevo a `setAudioSource` y `setVideoSource`. Llama a `release` para liberar  
los recursos asociados al objeto `MediaRecorder` \(el objeto no podrá volver a ser usado, se tendrá que crear de nuevo\):

```java
mediaRecorder.stop();
mediaRecorder.reset();
mediaRecorder.release();
```

### Previsualización

Durante la grabación de vídeo es recomendable mostrar una previsualización de lo que se está captando a través de la cámara en tiempo real. Para ello utilizaremos el método `setPreviewDisplay`, que nos permitirá asignar un objeto `Surface` sobre el cual mostrar dicha previsualización.

El comportamiento en este caso es muy parecido al de la clase `MediaPlayer` para la reproducción de vídeo. Debemos definir una actividad que incluya una vista de tipo `SurfaceView` en su interfaz y que implemente la interfaz `SurfaceHolder.Callback`. Una vez que el objeto `SurfaceHolder` ha sido creado podemos  
asignarlo al objeto `MediaRecorder` invocando al método `setPreviewDisplay`, tal como se puede ver en el siguiente código. El vídeo comenzará a previsualizarse tan pronto como se haga uso del método `prepare`.

```java
public class CapturaActivity extends Activity implements SurfaceHolder.Callback
{

    private MediaRecorder mMediaRecorder;

    @Override
    public void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.main);

        SurfaceView surface = (SurfaceView)findViewById(R.id.surface);
        SurfaceHolder holder = surface.getHolder();
        holder.addCallback(this);
        holder.setType(SurfaceHolder.SURFACE_TYPE_PUSH_BUFFERS);
        holder.setFixedSize(400, 300);
    }

    public void surfaceCreated(SurfaceHolder holder) {
        if (mMediaRecorder != null) {
            try {
                mMediaRecorder.setAudioSource(MediaRecorder.AudioSource.MIC);
                mMediaRecorder.setVideoSource(MediaRecorder.VideoSource.CAMERA);

                mMediaRecorder.setOutputFormat(MediaRecorder.OutputFormat.DEFAULT);

                mMediaRecorder.setAudioEncoder(MediaRecorder.AudioEncoder.DEFAULT);
                mMediaRecorder.setVideoEncoder(MediaRecorder.VideoEncoder.DEFAULT);

                mMediaRecorder.setOutputFile("/sdcard/myoutputfile.mp4");

                // Asociando la previsualización a la superficie
                mMediaRecorder.setPreviewDisplay(holder.getSurface());
                mMediaRecorder.prepare();
            } catch (IllegalArgumentException e) {
                Log.d("MEDIA_PLAYER", e.getMessage());
            } catch (IllegalStateException e) {
                Log.d("MEDIA_PLAYER", e.getMessage());
            } catch (IOException e) {
                Log.d("MEDIA_PLAYER", e.getMessage());
            }
        }
    }

    public void surfaceDestroyed(SurfaceHolder holder) {
        mMediaRecorder.release();
    }

    public void surfaceChanged(SurfaceHolder holder,
                    int format, int width, int height) { }
}
```

## Agregar ficheros multimedia en el Media Store

El comportamiento por defecto en Android con respecto al acceso de contenido multimedia es que los ficheros multimedia generados u obtenidos por una aplicación no podrán ser accedidos por el resto. En el caso de que deseemos que un nuevo fichero multimedia sí pueda ser accedido desde el exterior de nuestra aplicación deberemos almacenarlo en el _Media Store_, que mantiene una base de datos de la metainformación de todos los ficheros almacenados tanto en dispositivos externos como internos del terminal telefónico.

> El _Media Store_ es un proveedor de contenidos, y por lo tanto utilizaremos el mecanismo estándar para acceso a dichos proveedores para acceder a la información que contiene.

Existen varias formas de incluir un fichero multimedia en el _Media Store_. La más sencilla es hacer uso de la clase `MediaScannerConnection`, que permitirá determinar automáticamente de qué tipo de fichero se trata, de tal forma que se pueda añadir sin necesidad de proporcionar ninguna información adicional.

La clase `MediaScannerConnection` proporciona un método `scanFile` para realizar esta tarea. Sin embargo, antes de escanear un fichero se deberá llamar al método `connect` y esperar una conexión al _Media Store_. La llamada a `connect` es asíncrona, lo cual quiere decir que deberemos crear un objeto `MediaScannerConnectionClient` que nos notifique en el momento en el que se complete la conexión. Esta misma clase también puede ser utilizada para que se lleve a cabo una notificación en el momento en el que el escaneado se haya completado, de tal forma que ya podremos desconectarnos del _Media Store_.

En el siguiente ejemplo de código podemos ver un posible esqueleto para un objeto `MediaScannerConnectionClient`. En este código se hace uso de una instancia de la clase `MediaScannerConnection` para manejar la conexión y escanear el fichero. El método `onMediaScannerConected` será llamado cuando la conexión ya se haya establecido, con lo que ya será posible escanear el fichero. Una vez se complete el escaneado se llamará al método `onScanCompleted`, en el que lo más aconsejable es llevar a cabo la desconexión del _Media Store_.

```java
MediaScannerConnectionClient mediaScannerClient = new MediaScannerConnectionClient() {
    private MediaScannerConnection msc = null;
    {
        msc = new MediaScannerConnection(getApplicationContext(), this);
    msc.connect();
    }

    public void onMediaScannerConnected() {
        msc.scanFile("/mnt/sdcard/DCIM/prueba.mp4", null);
    }

    public void onScanCompleted(String path, Uri uri) {

        // Realizar otras acciones adicionales

        msc.disconnect();
    }
};
```

## Ejercicios

### Grabación de vídeo con MediaRecorder

En este ejercicio optativo utilizaremos la aplicación _Video_ que se te proporciona en las plantillas para crear una aplicación que permita guardar vídeo, mostrándolo en pantalla mientras éste se graba. La interfaz de la actividad principal tiene dos botones, _Grabar_ y _Parar_, y una vista `SurfaceView` sobre la  
que se previsualizará el vídeo siendo grabado.

Debes seguir los siguientes pasos:

* Añade los permisos necesarios en el _Manifest_ de la aplicación para poder grabar audio y vídeo y para poder guardar el resultado en la tarjeta SD \(recuerda
  que el siguiente código debe aparecer antes del elemento `application`\):

```xml
<uses-permission android:name="android.permission.CAMERA"/>
<uses-permission android:name="android.permission.RECORD_AUDIO"/>
<uses-permission android:name="android.permission.WRITE_EXTERNAL_STORAGE"/>
```

* Añade un atributo a la clase `VideoActivity:`

```java
MediaRecorder mediaRecorder;
```

* Inicializa el objeto `MediaRecorder` en el método `onCreate`:

```java
mediaRecorder = new MediaRecorder();
```

* Para poder previsualizar el vídeo en el `SurfaceView` hemos de obtener su _holder_. Como esta operación es asíncrona, debemos añadir los manejadores adecuados, de tal forma que sólo se pueda reproducir la previsualización cuando todo esté listo. El primer paso consiste en hacer que la clase `VideoActivity` implemente la interfaz `SurfaceHolder.Callback`. Para implementar esta interfaz deberás añadir los siguiente métodos a la clase:

```java
public void surfaceCreated(SurfaceHolder holder) {
    // TODO: asociar la superficie al MediaRecorder
}

public void surfaceDestroyed(SurfaceHolder holder) {
    // TODO: liberar los recursos
}
```

* Añadimos en `onCreate` el código necesario para obtener el _holder_ de la superficie y asociarle como manejador la propia clase `VideoActivity`:

```java
m_holder = superficie.getHolder();
m_holder.addCallback(this);
m_holder.setType(SurfaceHolder.SURFACE_TYPE_PUSH_BUFFERS);
```

* Gracias al método `surfaceCreated` podremos asociar el objeto `MediaRecorder` al _holder_ del `SurfaceView`. Dentro de esta misma función le daremos al atributo booleano `preparado` el valor true, lo cual nos permitirá saber que ya podemos iniciar la reproducción:

```java
mediaRecorder.setPreviewDisplay(holder.getSurface());
preparado = true;
```

* En el método `surfaceDestroyed` simplemente invocaremos el método `release` del objeto `MediaRecorder`, para liberar los recursos del objeto  
  al finalizar la actividad.

* Se ha añadido un método `configurar` a la clase `VideoActivity` que se utilizará para indicar la fuente de audio y vídeo, el nombre del fichero donde  
  guardaremos el vídeo grabado, y algunos parámetros más. En esa función debes añadir el siguiente código. Fíjate cómo se ha incluido una llamada a `prepare` al final:

```java
if (mediaRecorder != null) {
    try {

        // Inicializando el objeto MediaRecorder
        mediaRecorder.setAudioSource(MediaRecorder.AudioSource.MIC);
        mediaRecorder.setVideoSource(MediaRecorder.VideoSource.CAMERA);

        mediaRecorder.setOutputFormat(MediaRecorder.OutputFormat.THREE_GPP);

        mediaRecorder.setAudioEncoder(MediaRecorder.AudioEncoder.DEFAULT);
        mediaRecorder.setVideoEncoder(MediaRecorder.VideoEncoder.DEFAULT);

        File file = new File(Environment.getExternalStoragePublicDirectory(Environment.DIRECTORY_PICTURES),"video.mp4");
        mediaRecorder.setOutputFile(file.getPath());

        mediaRecorder.prepare();

    } catch (IllegalArgumentException e) {
        Log.d("MEDIA_PLAYER", e.getMessage());
    } catch (IllegalStateException e) {
        Log.d("MEDIA_PLAYER", e.getMessage());
    } catch (IOException e) {
        Log.d("MEDIA_PLAYER", e.getMessage());
    }
}
```

* Sólo queda introducir el código necesario para iniciar y detener la reproducción. En el manejador del botón _Grabar_ invocaremos al método `start` del objeto `MediaRecorder`, sin olvidar realizar una llamada previa al método `configurar`.

* En el manejador del botón _Parar_ invocamos en primer lugar el método `stop` y en segundo lugar el método `reset` del objeto `MediaRecorder`. Con esto podríamos volver a utilizar este objeto llamando a `configurar` y a `start`.

> A la hora de redactar estos ejercicios existía un bug que impedía volver a utilizar un objeto `MediaRecorder` tras haber usado `reset`. Puede que sea necesario que tras hacer un `reset` debas invocar el método `release` y crear una nueva instancia del objeto `MediaRecorder` con el operador `new`.



