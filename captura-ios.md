# Captura y procesamiento de medios en iOS

Vamos a estudiar las formas en las que podemos capturar medios desde dispositivos iOS (fotografías y audio/vídeo), y posteriormente procesarlos.

## Fotografías y galería multimedia

La forma más sencilla de realizar una captura con la cámara del dispositivo es tomar una fotografía.
Para ello contamos con un controlador predefinido que simplificará esta tarea, ya que sólo deberemos ocuparnos
de instanciarlo, mostrarlo y obtener el resultado:

**Swift**
```swift
var picker = UIImagePickerController()
picker.sourceType = .camera
self.present(picker, animated: true, completion: nil)
```
**Objective-C**
```objectivec
UIImagePickerController *picker = [[UIImagePickerController alloc] init];
picker.sourceType = UIImagePickerControllerSourceTypeCamera;
[self presentModalViewController:picker animated:YES];
```

Podemos observar que podemos cambiar la fuente de la que obtener la fotografía. En el ejemplo anterior
hemos especificado la cámara del dispositivo, sin embargo, también podremos hacer que seleccione la imagen de
la colección de fotos del usuario (`UIImagePickerControllerSourceTypePhotoLibrary`), o del carrete
de la cámara (`UIImagePickerControllerSourceTypeSavedPhotosAlbum`):

**Swift**
```swift
picker.sourceType = .savedPhotosAlbum
```
**Objective-C**
```objectivec
picker.sourceType = UIImagePickerControllerSourceTypeSavedPhotosAlbum;
```

Si lo que queremos es almacenar una fotografía en el carrete de fotos del dispositivo podemos utilizar
la función `UIImageWriteToSavedPhotosAlbum`:

**Swift**
```swift
var image : UIImage = ...;
UIImageWriteToSavedPhotosAlbum(image, self, #selector(self.guardada), nil)
```
**Objective-C**
```objectivec
UIImage *image = ...;

UIImageWriteToSavedPhotosAlbum(image, self, @selector(guardada:), nil);
```

Como primer parámetro debemos proporcionar la imagen a guardar. Después podemos proporcionar de forma opcional un _callback_
mediante _target_ y _selector_ para que se nos notifique cuando la imagen haya sido guardada. Por último, podemos especificar
también de forma opcional cualquier información sobre el contexto que queramos que se le pase al _callback_
anterior, en caso de haberlo especificado.

Por ejemplo, si queremos tomar una fotografía y guardarla en el carrete del dispositivo, podemos crear un delegado
de `UIImagePickerController`, de forma que cuando la fotografía haya sido tomada llame a la función anterior para
almacenarla. Para ello debemos crear un objeto que adopte el delegado `UIImagePickerControllerDelegate` y
establecer dicho objeto en el propiedad `delegate` del controlador. Deberemos definir el siguiente método
del delegado:

**Swift**
```swift
func imagePickerController(picker: UIImagePickerController,   didFinishPickingMediaWithInfo info: [String : AnyObject]) {
 let pickedImage = info[UIImagePickerControllerOriginalImage]
 UIImageWriteToSavedPhotosAlbum(pickedImage as! UIImage, self, #selector(self.guardada), nil)
}
```
**Objective-C**
```objectivec
- (void)imagePickerController:(UIImagePickerController *)picker
didFinishPickingMediaWithInfo:(NSDictionary *)info
{
    UIImage *imagen =
        [info valueForKey: UIImagePickerControllerOriginalImage];
    UIImageWriteToSavedPhotosAlbum(imagen, self, @selector(guardada:),
                                   nil);
}
```

Este controlador nos permitirá capturar tanto imágenes como vídeo. Por defecto el controlador se mostrará con la interfaz de captura
de cámara nativa del dispositivo. Sin embargo, podemos personalizar esta interfaz con los métodos `showsCameraControls`,
`cameraOverlayView`, y `cameraViewTransform`. Si estamos utilizando una vista personalizada, podremos
controlar la toma de fotografías y la captura de vídeo con los métodos `takePicture`, `startVideoCapture` y
`stopVideoCapture`.

Si queremos tener un mayor control sobre la forma en la que se almacenan los diferentes tipos de recursos multimedia en el dispositivo
deberemos utilizar el _framework_ Assets. Con esta librería podemos por ejemplo guardar metadatos con las fotografías, como
puede ser la localización donde fue tomada.




## Captura avanzada de vídeo

A partir de iOS 4.0 en el _framework_ AVFoundation se incorpora la posibilidad de acceder
a la fuente de captura de vídeo a bajo nivel. Para ello tenemos un objeto `AVCaptureSession`
que representa la sesión de captura, y se encarga de coordinar la entrada y la salida de audio y vídeo,
y los objetos `AVCaptureInput` y  `AVCaptureOutput` que nos permiten establecer
la fuente y el destino de estos medios. De esta
forma podemos hacer por ejemplo que la fuente de vídeo sea un dispositivo de captura (por ejemplo la cámara), y que la salida se nos proporcione como datos
crudos de cada fotograma obtenido, para así poder procesarlo y mostrarlo nosotros como creamos
conveniente.

### Entrada de la sesión

Especificaremos la entrada mediante un objeto de tipo `AVCaptureInput` (o subclases suyas). Si queremos que la fuente de vídeo se obtenga de un dispositivo de captura, utilizaremos como entrada un objeto de la subclase `AVCaptureDeviceInput`, que inicializaremos proporcionando un objeto `AVCaptureDevice` que definirá el dispositivo del cual queremos capturar:

**Swift**
```swift
var captureDevice = AVCaptureDevice.defaultDevice(withMediaType: AVMediaTypeVideo)!
var captureInput = try! AVCaptureDeviceInput(device: captureDevice)
```
**Objective-C**
```objectivec
AVCaptureDevice *captureDevice = [AVCaptureDevice defaultDeviceWithMediaType:AVMediaTypeVideo];
AVCaptureDeviceInput *captureInput = [AVCaptureDeviceInput deviceInputWithDevice:captureDevice error: nil];
```

En este ejemplo estamos obteniendo el dispositivo de captura por defecto que nos proporcione vídeo (la cámara), pero podríamos solicitar otros tipos de dispositivos de entrada.

Podríamos también recorrer la lista de todos los dispositivos disponibles y comprobar sus características (por ejemplo si tiene _flash_ o _autofocus_, o si es la cámara frontal o trasera). 

### Salida de la sesión

La salida se especificará mediante subclases de `AVCaptureOutput`. Según el destino de la captura tenemos:  
* `AVCaptureMovieFileOutput`: Nos permite grabar el vídeo capturado en un fichero.
* `AVCaptureVideoDataOutput`: Nos permite procesar los fotogramas de vídeo capturados en tiempo real (nos da acceso al _framebuffer_).
* `AVCaptureAudioDataOutput`: Nos permite procesar audio capturado en tiempo real.
* `AVCaptureStillImageOutput`: Nos permite tomar fotografías a partir de la fuente de entrada.

#### Captura de fotogramas

Por ejemplo, para establecer la salida de tipo `AVCaptureStillImageOutput` podemos hacer lo siguiente:

**Swift**
```swift
var captureOutput = AVCaptureStillImageOutput()
```
**Objective-C**
```objectivec
captureOutput = [[AVCaptureStillImageOutput alloc] init];
```

Con este tipo de salida de captura, en todo momento podremos tomar un fotograma a partir de la entrada con:

**Swift**
```swift
...
```
**Objective-C**
```objectivec
AVCaptureConnection *connection = [[self.captureOutput connections] objectAtIndex:0];

[self.captureOutput captureStillImageAsynchronouslyFromConnection:connection completionHandler:^(CMSampleBufferRef imageDataSampleBuffer, NSError *error) {
     
     NSData *data = [AVCaptureStillImageOutput jpegStillImageNSDataRepresentation:imageDataSampleBuffer];
     UIImage *image = [UIImage imageWithData:data];
     [self.ivPreview performSelectorOnMainThread: @selector(setImage:)
                                      withObject: image
                                   waitUntilDone: YES];
     
}];
```

#### Procesamiento en tiempo real

Para capturar en memoria y poder procesar fotogramas podemos utiliza el tipo de salida `AVCaptureVideoDataOutput`:

```objectivec
    AVCaptureVideoDataOutput *captureOutput = [[AVCaptureVideoDataOutput alloc] init];
    captureOutput.alwaysDiscardsLateVideoFrames = YES;
    //captureOutput.minFrameDuration = CMTimeMakeWithSeconds(1, 1);
    
    dispatch_queue_t queue = dispatch_queue_create("cameraQueue", NULL);
    [captureOutput setSampleBufferDelegate: self queue: queue];
    
    NSDictionary *videoSettings = @{ (NSString*)kCVPixelBufferPixelFormatTypeKey : @(kCVPixelFormatType_32BGRA) };    
    [captureOutput setVideoSettings: videoSettings];
```

En este caso tenemos que proporcionar un delegado de tipo `AVCaptureVideoDataOutputSampleBufferDelegate`, que tendrá que definir un método como el siguiente que será invocado cada vez que se capture un fotograma:

```objectivec
- (void)captureOutput:(AVCaptureOutput *)captureOutput
didOutputSampleBuffer:(CMSampleBufferRef)sampleBuffer
       fromConnection:(AVCaptureConnection *)connection {
    
    // Procesar datos de sampleBuffer
    
}
```

Podemos utilizar este método para procesar el vídeo.

### Sesión de captura

La sessión de captura coordina la entrada y la salida. Podemos establecer diferentes _presets_ para la sesión, según la calidad con la que queramos capturar el medio. En el siguiente ejemplo utilizamos un _preset_ para capturar vídeo en 720p, y tras ello añadimos la entrada y la salida de la sesión:

```objectivec
AVCaptureSession *captureSession = [[AVCaptureSession alloc] init];
    
// Establece preset
if([captureSession canSetSessionPreset:AVCaptureSessionPreset1280x720]) {
    captureSession.sessionPreset = AVCaptureSessionPreset1280x720;
} else {
    NSLog(@"Preset no compatible");
}

// Establece entrada
if([captureSession canAddInput:captureInput]) {
    [captureSession addInput: captureInput];
} else {
    NSLog(@"No se puede añadir entrada");
}

// Establece salida
if([captureSession canAddOutput:captureOutput]) {
    [captureSession addOutput: captureOutput];
} else {
    NSLog(@"No se puede añadir salida");
}
```

Después de configurar la sesión, deberemos iniciar la captura con `startRunning`:

```objectivec
[captureSession startRunning];
```

### Ejemplo de captura y procesamiento de fotogramas

En el siguiente ejemplo creamos una sesión de captura que tiene como entrada el dispositivo de captura
de vídeo, y como salida los fotogramas del vídeo sin compresión como datos crudos.
Tras configurar la entrada, la salida, y la sesión de captura, ponemos dicha sesión en funcionamiento
con `startRunning`:

```objectivec
   // Entrada del dispositivo de captura de video
    AVCaptureDevice *captureDevice = [AVCaptureDevice defaultDeviceWithMediaType:AVMediaTypeVideo];
    AVCaptureDeviceInput *captureInput = [AVCaptureDeviceInput deviceInputWithDevice:captureDevice 
                                                                               error: nil];
    
    // Salida como fotogramas "crudos" (sin comprimir)
    AVCaptureVideoDataOutput *captureOutput = [[AVCaptureVideoDataOutput alloc] init];
    captureOutput.alwaysDiscardsLateVideoFrames = YES;

    dispatch_queue_t queue = dispatch_queue_create("cameraQueue", NULL);
    [captureOutput setSampleBufferDelegate: self queue: queue];
    
    NSDictionary *videoSettings = @{ (NSString*)kCVPixelBufferPixelFormatTypeKey : 
                                                        @(kCVPixelFormatType_32BGRA) };    
    [captureOutput setVideoSettings: videoSettings];
    
    // Creación de la sesión de captura
    self.captureSession = [[AVCaptureSession alloc] init];
    self.captureSession.sessionPreset = AVCaptureSessionPreset1280x720;
    [self.captureSession addInput: captureInput];
    [self.captureSession addOutput: captureOutput];
    
    [self.captureSession startRunning];
}
```

Una vez haya comenzado la sesión de captura, se comenzarán a producir fotogramas del vídeo
capturado. Para consumir estos fotogramas deberemos implementar el método delegado
`captureOutput:didOutputSampleBuffer:fromConnection:`

```objectivec
- (void)captureOutput:(AVCaptureOutput *)captureOutput
didOutputSampleBuffer:(CMSampleBufferRef)sampleBuffer
       fromConnection:(AVCaptureConnection *)connection {
    
    UIImage *image = [self imageFromSampleBuffer:sampleBuffer];
    
    [self.ivPreview performSelectorOnMainThread: @selector(setImage:)
                                     withObject: image
                                  waitUntilDone: YES];
}
```

Vemos que el primer paso consiste en transformar el _buffer_ del fotograma actual en un objeto `UIImage` que podamos mostrar. Para ello podemos definir un método como el siguiente:

```objectivec
- (UIImage *) imageFromSampleBuffer:(CMSampleBufferRef) sampleBuffer
{
    // Obtiene y bloquea el framebuffer del actual fotograma
    CVImageBufferRef imageBuffer = CMSampleBufferGetImageBuffer(sampleBuffer);
    CVPixelBufferLockBaseAddress(imageBuffer, 0);
    
    // Obtiene el puntero al inicio del framebuffer
    void *baseAddress = CVPixelBufferGetBaseAddress(imageBuffer);
    
    // Obtiene las dimensiones de la imagen y bytes por fila del framebuffer
    size_t bytesPerRow = CVPixelBufferGetBytesPerRow(imageBuffer);
    size_t width = CVPixelBufferGetWidth(imageBuffer);
    size_t height = CVPixelBufferGetHeight(imageBuffer);
    
    // Procesa la imagen
    void *output = [self procesaImagen:(uint8_t *)baseAddress];
    
    // Crea espacio de color RGB dependiente del dispositivo
    CGColorSpaceRef colorSpace = CGColorSpaceCreateDeviceRGB();
    
    // Crea un contexto gráfico para dibujar en un bitmap y vuelca en él el contenido del fotograma
    CGContextRef context = CGBitmapContextCreate(output, width, height, 8,
                                                 bytesPerRow, colorSpace, kCGBitmapByteOrder32Little | kCGImageAlphaPremultipliedFirst);
    // Crea una imagen a partir del contenido del contexto gráfico
    CGImageRef quartzImage = CGBitmapContextCreateImage(context);
    // Desbloquea el framebuffer
    CVPixelBufferUnlockBaseAddress(imageBuffer,0);
    
    // Libera el contexto gráfico
    CGContextRelease(context);
    CGColorSpaceRelease(colorSpace);
    
    // Crea una imagen UIImage a partir de la CGImage
    UIImage *image = [UIImage imageWithCGImage:quartzImage];
    
    // Libera la CGImage
    CGImageRelease(quartzImage);
    
    return (image);
}
```

En el método anterior observamos que podemos procesar y modificar el _buffer_ del fotograma antes de obtener una `UIImage` a partir de él. Por ejemplo, podríamos convertir la imagen a escala de grises con:

```objectivec
- (uint8_t *)procesaImagen:(uint8_t *)input {
    static long size = WIDTH * HEIGHT;
    static uint8_t result[WIDTH * HEIGHT*4];
    
    uint8_t *output = result;
    
    for(int i=0;i<size;i++) {
        uint8_t b = *input++;
        uint8_t g = *input++;
        uint8_t r = *input++;
        uint8_t a = *input++;
        
        uint8_t gray = (r+g+b)/3;
        
        *output++ = gray;
        *output++ = gray;
        *output++ = gray;
        *output++ = a;
    }
    
    return result;
}
```



## Procesamiento de imágenes en iOS

El procesamiento de imágenes es una operación altamente costosa, por lo que supone un auténtico reto llevarla
a un dispositivo móvil de forma eficiente, especialmente si queremos ser capaces de procesar vídeo en tiempo real.
Una de las aplicaciones del procesamiento de imágenes es el tratamiento de fotografías mediante una serie de
filtros. También podemos encontrar numerosas aplicaciones relacionadas con el campo de la visión por computador,
como la detección de movimiento, el seguimiento de objetos, o el reconocimiento de caras.

Estas operaciones suponen una gran carga de procesamiento, por lo que si queremos realizarlas de forma eficiente
deberemos realizar un fuerte trabajo de optimización. Implementar directamente los algoritmos de procesamiento
de imágenes sobre la CPU supone una excesiva carga para la aplicación y resulta poco eficiente. Sin embargo,
podemos llevar este procesamiento a unidades más adecuadas para esta tarea, y así descargar la carga de trabajo
de la CPU. Encontramos dos opciones:


* Utilizar la unidad NEON de los procesadores con juego de instrucciones ARMv7. Se trata de una unidad SIMD
(_Single Instruction Multiple Data_), con la cual podemos vectorizar las operaciones de procesamiento de imagen y ejecutarlas de una
forma mucho más eficiente, ya que en cada operación del procesador en lugar de operar sobre un único dato,
lo haremos sobre un vector de ellos. El mayor inconveniente de esta opción es el trabajo que llevará
vectorizar los algoritmos de procesamiento a aplicar. Como ventaja tenemos que el juego de instrucciones que
podemos utilizar funcionará en cualquier dispositivo ARMv7, y la práctica totalidad de dispositivos que hay
actualmente en el mercado disponen de este juego de instrucciones. De esta forma, el código que escribamos
será compatible con cualquier dispositivo, independientemente del sistema operativo que incorporen.

http://www.arm.com/products/processors/technologies/neon.php


* Utilizar la GPU (_Graphics Processing Unit_). Podemos programar _shaders_, es decir, programas
que se ejecutan sobre la unidad de procesamiento gráfica, que esta especializada en operaciones de manipulación
de gráficos con altos niveles de paralelismo. El lenguaje en el que se programan los shaders dentro de OpenGL
es GLSL. Con esta tecnología podemos desarrollar filtros que se ejecuten de forma optimizada por la GPU
descargando así totalmente a la CPU del procesamiento. Para utilizar esta opción deberemos estar familiarizados
con los gráficos por computador y con el lenguaje GLSL.


Con cualquiera de las opciones anteriores tendremos que invertir un gran esfuerzo en la implementación
óptima de las funciones de procesado. Sin embargo, a partir de iOS 5 se incorpora un nuevo _framework_
conocido como Core Image que nos permite realizar este procesamiento de forma óptima sin tener que entrar
a programar a bajo nivel. Este _framework_ ya existía anteriormente en MacOS, pero con la versión 5
de iOS ha sido trasladado a la plataforma móvil. Por el momento, la versión de iOS de Core Image es una versión
reducida, en la que encontramos una menor cantidad de filtros disponibles y además, al
contrario de lo que ocurre en MacOS, no podemos crear de momento nuestros propios filtros. Aun así, contamos
con un buen número de filtros (alrededor de 50) que podemos configurar y combinar para así aplicar distintos
efectos a las imágenes, y que nos permiten realizar tareas complejas de visión artificial como el reconocimiento
de caras. Vamos a continuación a ver cómo trabajar con esta librería.


### Representación de imágenes en Core Image

En el _framework_ Core Image las imágenes se representan mediante la clase `CIImage`. Este
tipo de imágenes difiere de las representaciones que hemos visto anteriormente (`UIImage` y `CGImageRef`)
en que `CIImage` no contiene una representación final de la imagen, sino que lo que contiene es una
imagen inicial y una serie de filtros que se deben aplicar para obtener la imagen final a representar. La
imagen final se calculará en el momento en el que la imagen `CIImage` final sea renderizada.

Podemos crear una imagen de este tipo a partir de imágenes de Core Graphics:

**Swift**
```swift
var cgImage = UIImage(named: "imagen.png")!.cgImage!
var ciImage = CIImage(cgImage: cgImage)
```

**Objective-C**
```objectivec
CGImageRef cgImage = [UIImage imageNamed: @"imagen.png"].CGImage;
CIImage *ciImage = [CIImage imageWithCGImage: cgImage];
```

También podemos encontrar inicializadores de `CIImage` que crean la imagen a partir de los contenidos
de una URL o directamente a partir de los datos crudos (`NSData`) correspondientes a los distintos formatos
de imagen soportados (JPEG, GIF, PNG, etc).

Podemos también hacer la transformación inversa, y crear un objeto `UIImage` a partir de una imagen
de tipo `CIImage`. Esto lo haremos con el inicializador `initWithCIImage:`, y podremos obtener
la representación de la imagen como `CIImage` mediante la propiedad `CIImage` de `UIImage`.
Dicha imagen podrá ser dibujada en el contexto gráfico como se ha visto en sesiones anteriores:

**Swift**
```swift
var uiImage = UIImage(ciImage: ciImage)
var ciImage = uiImage.ciImage!
...
// En drawRect: (o con algún contexto gráfico activo)
uiImage.draw(at: CGPoint.zero)
```

**Objective-C**
```objectivec
UIImage *uiImage = [UIImage imageWithCIImage: ciImage];
...

CIImage *ciImage = uiImage.CIImage;
...

// En drawRect: (o con algún contexto gráfico activo)
[uiImage drawAtPoint: CGPointZero];
```

> Cuando queramos crear una imagen para mostrar en la interfaz (`UIImage`) a partir de una imagen de Core
Image (`CIImage`), deberemos llevar cuidado porque la imagen puede no mostrarse correctamente en
determinados ámbitos. Por ejemplo, no se verá correctamente si la mostramos en un `UIImageView`, pero
si que funcionará si la dibujamos directamente en el contexto gráfico con sus métodos `drawAtPoint:`
o `drawInRect:`. La razón de este comportamiento se debe a que la representación interna de la imagen variará según la forma en la que se
cree. Si una imagen `UIImage` se crea a partir de una imagen de tipo `CGImageRef`, su propiedad
`CGImage` apuntará a la imagen a partir de la cual se creó, pero su propiedad `CIImage` será
`nil`. Sin embargo, si creamos una imagen a partir de una `CIImage` ocurrirá al contrario, su
propiedad `CGImage` será `NULL` mientras que su propiedad `CIImage` apuntará
a la imagen inicial. Esto causará que aquellos componentes cuyo funcionamiento se base en utilizar la propiedad
`CGImage` dejen de funcionar.

La clase `CIImage` tiene además una propiedad `extent` que nos proporciona las dimensiones de la
imagen como un dato de tipo `CGRect`. Más adelante veremos que resulta de utilidad para renderizar la imagen.




### Filtros de Core Image

Los filtros que podemos aplicar sobre la imagen se representan con la clase `CIFilter`. Podemos
crear diferentes filtros a partir de su nombre:

**Swift**
```swift
var filter = CIFilter(name: "CISepiaTone")
```

**Objective-C**
```objectivec
CIFilter *filter = [CIFilter filterWithName: @"CISepiaTone"];
```

Otros filtros que podemos encontrar son:


* `CIAffineTransform`
* `CIColorControls`
* `CIColorMatrix`
* `CIConstantColorGenerator`
* `CICrop`
* `CIExposureAdjust`
* `CIGammaAdjust`
* `CIHighlightShadowAdjust`
* `CIHueAdjust`
* `CISourceOverCompositing`
* `CIStraightenFilter`
* `CITemperatureAndTint`
* `CIToneCurve`
* `CIVibrance`
* `CIWhitePointAdjust`


Todos los filtros pueden recibir una serie de parámetros de entrada, que variarán según el filtro. Un parámetro
común que podemos encontrar en casi todos ellos es la imagen de entrada a la que se aplicará el filtro. Además,
podremos tener otros parámetros que nos permitan ajustar el funcionamiento del filtro. Por ejemplo, en el caso
del filtro para convertir la imagen a tono sepia tendremos un parámetro que nos permitirá controlar la intensidad
de la imagen sepia:

**Swift**
```swift
let filter = CIFilter(name: "CISepiaTone")
filter?.setValue(ciImage, forKey: kCIInputImageKey)
filter?.setValue(0.5, forKey: kCIInputIntensityKey)
```
**Objective-C**
```objectivec
CIFilter *filter =
    [CIFilter filterWithName:@"CISepiaTone"
               keysAndValues:
                   kCIInputImageKey, ciImage,
                   @"inputIntensity", [NSNumber numberWithFloat:0.8],
                   nil];
```

Podemos ver que para la propiedad correspondiente a la imagen de entrada tenemos la constante
`kCIInputImageKey`, aunque también podríamos especificarla como la cadena `@"inputImage"`.
Las propiedades de los filtros también pueden establecerse independientemente utilizando KVC:

**Swift**
```swift
filter?.setValue(ciImage, forKey: kCIInputImageKey)
filter?.setValue(0.8, forKey: kCIInputIntensityKey)
```

**Objective-C**
```objectivec
[filter setValue: ciImage forKey: @"inputImage"];
[filter setValue: [NSNumber numberWithFloat:0.8]
          forKey: @"inputIntensity"];
```

En la documentación de Apple no aparece la lista de filtros disponibles para iOS (si que tenemos la lista completa
para MacOS, pero varios de esos filtros no están disponibles en iOS). Podemos obtener la lista de los filtros disponibles
en nuestra plataforma desde la aplicación con los métodos `filterNamesInCategories:` y
`filterNamesInCategory:`. Por ejemplo, podemos obtener la lista de todos los filtros con:

**Swift**
```swift
var filters = CIFilter.filterNames(inCategories: nil)
```

**Objective-C**
```objectivec
NSArray *filters = [CIFilter filterNamesInCategories: nil];
```

Cada objeto de la lista será de tipo `CIFilter`, y podremos obtener de él sus atributos y las
características de cada uno de ellos mediante la propiedad `attributes`. Esta propiedad nos
devolverá un diccionario con todos los parámetros de entrada y salida del filtro, y las características de cada
uno de ellos. Por ejemplo, de cada parámetro nos dirá el tipo de dato que se debe indicar, y sus limitaciones
(por ejemplo, si es numérico sus valores mínimo y máximo). Como alternativa, también podemos obtener el
nombre del filtro con su propiedad `name` y las listas de sus parámetros de entrada y salida con
`inputKeys` y `outputKeys` respectivamente.

La propiedad más importante de los filtros es `outputImage`. Esta propiedad nos da la imagen
producida por el filtro en forma de objeto `CIImage`:

**Swift**
```swift
let outputImage = UIImage(ciImage: (filter?.outputImage)!)
```

**Objective-C**
```objectivec
CIImage *filteredImage = filter.outputImage;
```

Al obtener la imagen resultante el filtro no realiza el procesamiento. Simplemente anota en la imagen las
operaciones que se deben hacer en ella. Es decir, la imagen que obtenemos como imagen resultante, realmente
contiene la imagen original y un conjunto de filtros a aplicar. Podemos encadenar varios filtros en una
imagen:

**Swift**
```swift
for filter: CIFilter in filters {
   filter[kCIInputImageKey] = filteredImage
   filteredImage = filter.outputImage
}
```

**Objective-C**
```objectivec
for(CIFilter *filter in filters) {
    [filter setValue: filteredImage forKey: kCIInputImageKey];
    filteredImage = filter.outputImage;
}
```

Con el código anterior vamos encadenando una serie de filtros en la imagen `CIImage` resultante,
pero el procesamiento todavía no se habrá realizado. Los filtros realmente se aplicarán cuando rendericemos la
imagen, bien en pantalla, o bien en forma de imagen `CGImageRef`.

Por ejemplo, podemos renderizar la imagen directamente en el contexto gráfico actual. Ese será el momento
en el que se aplicarán realmente los filtros a la imagen, para mostrar la imagen resultante en pantalla:

**Swift**
```swift
override func draw(_ rect: CGRect)
{
    UIImage(ciImage: filteredImage).draw(at: CGPoint.zero)
}
```
**Objective-C**
```objectivec
- (void)drawRect:(CGRect)rect
{
    [[UIImage imageWithCIImage: filteredImage] drawAtPoint:CGPointZero];
}
```

A continuación veremos cómo controlar la forma en la que se realiza el renderizado de la imagen mediante
el contexto de Core Image.




### Contexto de Core Image

El componente central del _framework_ Core Image es la clase `CIContext` que representa
el contexto de procesamiento de imágenes, que será el motor que se encargará de aplicar diferentes filtros
a las imágenes. Este contexto puede se de dos tipos:


* **CPU**: El procesamiento se realiza utilizando la CPU. La imagen resultante se obtiene como
imagen de tipo Core Graphics (`CGImageRef`).
* **GPU**: El procesamiento se realiza utilizando la GPU, y la imagen se renderiza utilizando
OpenGL ES 2.0.


El contexto basado en CPU es más sencillo de utilizar, pero su rendimiento es mucho peor. Con el contexto
basado en GPU se descarga totalmente a la CPU del procesamiento de la imagen, por lo que será mucho más
eficiente. Sin embargo, para utilizar la GPU nuestra aplicación siempre debe estar en primer plano. Si queremos
procesar imágenes en segundo plano deberemos utilizar el contexto basado en CPU.

#### Procesamiento en contexto de CPU

Para crear un contexto basado en CPU utilizaremos el método `contextWithOption:`

**Swift**
```swift
var context = CIContext(options: nil)
```
**Objective-C**
```objectivec
CIContext *context = [CIContext contextWithOptions:nil];
```

Con este tipo de contexto la imagen se renderizará como `CGImageRef` mediante el método
`createCGImage:fromRect:`. Hay que especificar la región de la imagen que queremos renderizar.
Si queremos renderizar la imagen entera podemos utilizar el atributo `extent` de `CIImage`,
que nos devuelve sus dimensiones:

**Swift**
```swift
var cgiimg = context.createCGImage((filter?.outputImage)!, from: (filter?.outputImage?.extent)!)
```

**Objective-C**
```objectivec
CGImageRef cgImage = [context createCGImage:filteredImage
                                   fromRect:filteredImage.extent];
```

#### Procesamiento en contexto de GPU

En el caso del contexto basado en GPU, en primer lugar deberemos crear el contexto OpenGL en nuestra aplicación.
Esto se hará de forma automática en el caso en el que utilicemos la plantilla de Xcode de aplicación basada en
OpenGL, aunque podemos también crearlo de forma sencilla en cualquier aplicación con el siguiente código:

**Swift**
```swift 
let api: EAGLRenderingAPI = EAGLRenderingAPI.openGLES2 
var glContext = EAGLContext(api: api)
```

**Objective-C**
```objectivec
EAGLContext *glContext =
    [[EAGLContext alloc] initWithAPI:kEAGLRenderingAPIOpenGLES2];
```

Una vez contamos con el contexto de OpenGL, podemos crear el contexto de Core Image basado en GPU con el método `contextWithEAGLContext:`

**Swift**
```swift
var context = CIContext(glContext, options: nil)
```

**Objective-C**
```objectivec
CIContext *context = [CIContext contextWithEAGLContext: glContext];
```

Para realizar el procesamiento en tiempo real, si no necesitamos una alta fidelidad de color, se recomienda desactivar el uso del _color space_:

**Swift**
```swift
var options = [kCIContextWorkingColorSpace: NSNull()]
var context = CIContext(glContext, options: options)
```

**Objective-C**
```objectivec
NSDictionary *options = @{ kCIContextWorkingColorSpace : [NSNull null] };
CIContext *context = [CIContext contextWithEAGLContext:glContext options:options];
```

En este caso, para renderizar la imagen deberemos utilizar el método `drawImage:atPoint:fromRect:`
o `drawImage:inRect:fromRect:` del objeto `CIContext`. Con estos métodos la imagen se renderizará en una capa de OpenGL.
Para hacer esto podemos utilizar una vista de tipo `GLKView`. Podemos crear esta vista de la siguiente
forma:

**Swift**
```swift
var rect : CGRect = CGRect(x: 0, y: 0, width: 320, height: 480)
var glkView = GLKView(frame: rect, context: glContext!)
glkView.delegate = self
```

**Objective-C**
```objectivec
GLKView *glkView = [[GLKView alloc] initWithFrame: CGRect(0,0,320,480)
                                          context: glContext];
glkView.delegate = self;
```

El delegado de la vista OpenGL deberá definir un método `glkView:drawInRect:` en el que deberemos
definir la forma de renderizar la vista OpenGL. Aquí podemos hacer que se renderice la imagen filtrada:

**Swift**
```swift
func glkView(_ view: GLKView!, drawIn rect: CGRect) {
    ...
    context.draw(filteredImage, atPoint: CGPoint.zero,  fromRect: filteredImage.extent())}
```

**Objective-C**
```objectivec
- (void)glkView:(GLKView *)view drawInRect:(CGRect)rect {
   ...

   [context drawImage: filteredImage
           atPoint: CGPointZero
          fromRect: filteredImage.extent];
}
```

Para hacer que la vista OpenGL actualice su contenido deberemos llamar a su método `display`:

**Swift**
```swift
glkView.display()
```

**Objective-C**
```objectivec
[glkView display];
```

Esto se hará normalmente cuando hayamos definido nuevos filtros para la imagen, y queramos que se actualice el resultado en pantalla.

> La inicialización del contexto es una operación costosa que se debe hacer una única vez.
Una vez inicializado, notaremos que el procesamiento de las imágenes es mucho más fluido.


### Procesamiento asíncrono

El procesamiento de la imagen puede ser una operación lenta, a pesar de estar optimizada. Por lo tanto, al realizar
esta operación desde algún evento (por ejemplo al pulsar un botón, o al modificar en la interfaz algún factor de ajuste
del filtro a aplicar) deberíamos realizar la operación en segundo plano. Podemos utilizar para ello la clase
`NSThread`, o bien las facilidades para ejecutar código en segundo plano que se incluyeron a partir de
iOS 4.0, basadas en bloques.

```objectivec
dispatch_async(
    dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_BACKGROUND, 0),
    ^(void) {
         // Codigo en segundo plano (renderizar la imagen
         CGImageRef cgImage = [context createCGImage:filteredImage
                                       fromRect:filteredImage.extent];
         ...
    }
);
```

Con esto podemos ejecutar un bloque de código en segundo plano. El problema que encontramos es que dicho bloque de código
no se encuentra en el hilo de la interfaz, por lo que no podrá acceder a ella. Para solucionar este problema deberemos
mostrar la imagen obtenida en la interfaz dentro de un bloque que se ejecute en el hilo de la UI:

```objectivec
dispatch_async(
    dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_BACKGROUND, 0),
    ^(void) {
        ...
        dispatch_async(dispatch_get_main_queue(), ^(void) {
            self.imageView.image = [UIImage imageWithCGImage: cgImage];
        });
    }
);
```

Con esto podemos ejecutar un bloque de código de forma asíncrona dentro del hilo principal de la UI, y de esta forma podremos
mostrar la imagen obtenida en segundo plano en la interfaz.




### Detección de caras

A parte de los filtros vistos anteriormente, Core Image también incluye detectores de características en imágenes,  como por ejemplo detectores de caras, de texto, o de códigos QR. La API está diseñada para poder ser ampliada en el futuro.

Los detectores los crearemos mediante la clase `CIDetector`. Deberemos proporcionar el tipo de detector a utilizar, por ejemplo `CIDetectorTypeFace` para el detector de caras. Podemos además especificar una serie de parámetros, como
el nivel de precisión que queremos obtener:

**Swift**
```swift
let options = [CIDetectorAccuracy: CIDetectorAccuracyHigh]
let faceDetector = CIDetector(ofType: CIDetectorTypeFace, context: nil, options: options)!
```

**Objective-C**
```objectivec
CIDetector* detector = [CIDetector detectorOfType:CIDetectorTypeFace
     context:nil
     options:[NSDictionary dictionaryWithObject:CIDetectorAccuracyHigh
                                         forKey:CIDetectorAccuracy]];
```

Una vez creado el detector, podemos ejecutarlo para que procese la imagen (de tipo `CIImage`) en busca de las características deseadas
(en este caso estas características son las caras):

**Swift**
```swift
let features = faceDetector.features(in: ciImage)
```

**Objetive-C**
```objectivec
NSArray* features = [detector featuresInImage:ciImage];
```

Las características obtenidas se encapsulan en objetos de tipo `CIFeature`. Una propiedad básica de las
características es la región que ocupan en la imagen. Esto se representa mediante su propiedad `bounds`, de
tipo `CGRect`, que nos indicará el área de la imagen en la que se encuentra la cara. Pero además, en el
caso concreto del reconocimiento de caras, las características obtenidas son un subtipo específico de `CIFeature`
(`CIFaceFeature`), que además de la región ocupada por la cara nos proporcionará la región ocupada por
componentes de la cara (boca y ojos).

Es decir, este detector nos devolverá un _array_ con tantos objetos `CIFaceFeature` como caras
encontradas en la imagen, y de cada cara sabremos el área que ocupa y la posición de los ojos y la boca, en caso de que los haya encontrado.


## Ejercicios

### Procesamiento de imagen

En este ejercicio procesaremos una imagen con CoreImage tanto utilizando la CPU como la GPU. En el proyecto `ProcesamientoImagen` tenemos toda la infraestructura necesaria ya creada. En `viewDidLoad`
se inicializa la imagen `CIImage` original, y los contextos CPU y GPU. Tenemos dos _sliders_
que nos permitirán aplicar filtros con diferentes niveles de intensidad. En la parte superior de la pantalla tenemos una imagen (`UIImageView`) con un _slider_ para aplicar el filtro utilizando la CPU,
y en la mitad inferior tenemos una vista OpenGL (`GLKView`) y un _slider_ para aplicar el
filtro en ella utilizando la GPU. Se pide:

* Implementar el filtrado utilizando CPU, en el método `sliderCpuCambia:` que se ejecutará cada vez que el _slider_ superior cambie de valor. Utilizaremos el filtro de color sepia (`CISepiaTone`), al que proporcionaremos como intensidad el valor del _slider_.

* Implementar el filtrado utilizando GPU, en el método `sliderCpuCambia:` que se ejecutará cada vez que el _slider_ inferior cambie de valor. Utilizaremos el mismo filtro que en el caso anterior, pero en este caso guardaremos la imagen resultante en la propiedad `imagenFiltrada` y haremos que se redibuje la vista OpenGL para que muestre dicha imagen. Mueve los dos _sliders_. ¿Cuál de ellos se mueve con mayor fluidez?

* Vamos a encadenar un segundo filtro, tanto para el contexto CPU como GPU. El filtro será
`CIHueAdjust`, que se aplicará justo después del filtro sepia. Consulta la documentación de
filtros de Apple para saber qué parámetros son necesarios. Se utilizará el mismo _slider_ que
ya tenemos para darle valor a este parámetro, es decir, el mismo _slider_ dará valor simultáneamente
a los parámetros de los dos filtros.

* Por último, vamos a permitir guardar la foto procesada mediante CPU en el álbum de fotos
del dispositivo. Para ello deberemos introducir en el método `agregarFoto:` el código que se encargue de realizar esta tarea, tomando la foto de `self.imageView.image`. Este método se ejecutará al pulsar sobre el botón que hay junto a la imagen superior.

