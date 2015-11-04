# Síntesis y reconocimiento del habla

En esta sesión continuamos examinando las capacidades multimedia de Android presentando el sintetizador de voz _Text to Speech_, el cual permitirá que una actividad reproduzca por los altavoces la lectura de un determinado texto. Se trata de un componente relativamente sencillo de utilizar que puede mejorar la accesibilidad de nuestras aplicaciones en gran medida.


## Sintetizador de voz de Android

Android incorpora desde la versión 1.6 un motor de síntesis de voz conocido como _Text to Speech_. Mediante su API podremos hacer que nuestros programas "lean" un texto al usuario. Es necesario tener en cuenta que por motivos de espacio en disco los paquetes de lenguaje pueden no estar instalados en el dispositivo. Por lo tanto, antes de que nuestra aplicación utilice _Text to Speech_ se podría considerar una buena práctica de programación el comprobar si dichos paquetes están instalados. Para ello podemos hacer uso de un `Intent` como el que se muestra a continuación:


```java
Intent intent = new Intent(TextToSpeech.Engine.ACTION_CHECK_TTS_DATA);
startActivityForResult(intent, TTS_DATA_CHECK);
```


El método `onActivityResult()` recibirá un `CHECK_VOICE_DATA_PASS` si todo está correctamente instalado. En caso contrario deberemos iniciar una nueva actividad por medio de un nuevo `Intent` implícito que haga uso de la acción `ACTION_INSTALL_TTS_DATA` del motor _Text to Speech_.



Una vez comprobemos que todo está instalado deberemos crear e inicializar una instancia de la clase `TextToSpeech`. Como no podemos utilizar dicha instancia hasta que esté inicializada (la inicialización se hace de forma asíncrona), la mejor opción es pasar como parámetro al constructor un manejador `onInitListener` de tal forma que en dicho  método se especifiquen las tareas a llevar a cabo por el sintetizador de voz una vez esté inicializado.


```java
boolean ttsIsInit = false;
TextToSpeech tts = null;

tts = new TextToSpeech(this, new OnInitListener() {
	public void onInit(int status) {
		if (status == TextToSpeech.SUCCESS) {
			ttsIsInit = true;
			// Hablar
		}
	}
});
```


Una vez que la instancia esté inicializada se puede utilizar el método `speak` para sintetizar voz por medio del dispositivo de salida por defecto. El primer parámetro será el texto a sintetizar y el segundo podrá ser o bien `QUEUE_ADD`, que añade una nueva salida de voz a la cola, o bien `QUEUE_FLUSH`, que elimina todo lo que hubiera en la cola y lo sustituye por el nuevo texto.


```java
tts.speak("Hello, Android", TextToSpeech.QUEUE_ADD, null);
```

Otros métodos de interés de la clase `TextToSpeech` son:



* `setPitch` y `setSpeechRate` permiten modificar el tono de voz y la velocidad. Ambos métodos aceptan un parámetro real.
* `setLanguage` permite modificar la pronunciación. Se le debe pasar como parámetro una instancia de la clase `Locale` para indicar el país y la lengua a
utilizar.
* El método `stop` se debe utilizar al terminar de hablar; este método detiene la síntesis de voz.
* El método `shutdown` permite liberar los recursos reservados por el motor de _Text to Speech_.



El siguiente código muestra un ejemplo en el que se comprueba si todo está correctamente instalado, se inicializa una nueva instancia de la clase `TextToSpeech`, y se utiliza dicha clase para decir una frase en español. Al llamar al método `initTextToSpeech` se desencadenará todo el proceso.


```java
private static int TTS_DATA_CHECK = 1;
private TextToSpeech tts = null;
private boolean ttsIsInit = false;

private void initTextToSpeech() {
    Intent intent = new Intent(Engine.ACTION_CHECK_TTS_DATA);
    startActivityForResult(intent, TTS_DATA_CHECK);
}

protected void onActivityResult(int requestCode, int resultCode,
                                Intent data) {
    if (requestCode == TTS_DATA_CHECK) {
        if (resultCode == Engine.CHECK_VOICE_DATA_PASS) {
            tts = new TextToSpeech(this, new OnInitListener() {
                public void onInit(int status) {
                    if (status == TextToSpeech.SUCCESS) {
                        ttsIsInit = true;
                        Locale loc = new Locale("es","","");
                        if (tts.isLanguageAvailable(loc) >=
                            TextToSpeech.LANG_AVAILABLE)
                                tts.setLanguage(loc);
                        tts.setPitch(0.8f);
                        tts.setSpeechRate(1.1f);
                        speak();
                    }
                }
            });
        } else {
            Intent installVoice =
                new Intent(Engine.ACTION_INSTALL_TTS_DATA);
            startActivity(installIntent);
        }
    }
}

private void speak() {
    if (tts != null && ttsIsInit) {
        tts.speak("Hola Android", TextToSpeech.QUEUE_ADD, null);
    }
}

@Override
public void onDestroy() {
    super.onDestroy();
    if (tts != null) {
        tts.stop();
        tts.shutdown();
    }

    super.onDestroy();
}
```

## Reconocimiento del habla en Android


Otro sensor que podemos utilizar para introducir información en
nuestras aplicaciones es el micrófono que incorpora el dispositivo.
Tanto el micrófono como la cámara se pueden utilizar para capturar
audio y video. Una característica altamente interesante
de los dispositivos Android es que nos permiten realizar reconocimiento
del habla de forma sencilla para introducir texto en nuestras
aplicaciones.

Para realizar este reconocimiento deberemos utilizar _intents_.
Concretamente, crearemos un `Intent` mediante las
constantes definidas en la clase `RecognizerIntent`,
que es la clase principal que deberemos utilizar para utilizar
esta característica.

Lo primer que deberemos hacer es crear un `Intent`
para inicial el reconocimiento:

```java
Intent intent = new Intent(
    RecognizerIntent.ACTION_RECOGNIZE_SPEECH);
```

Una vez creado, podemos añadir una serie de parámetros para
especificar la forma en la que se realizará el reconocimiento. Estos
parámetros se introducen llamando a:

```java
intent.putExtra(parametro, valor);
```

Los parámetros se definen como constantes de la clase
`RecognizerIntent`, todas ellas tienen el prefijo
`EXTRA_`. Algunos de estos parámetros son:

|Parámetro              |Valor           |
|:----------------------|:---------------|
|`EXTRA_LANGUAGE_MODEL` |**Obligatorio**. Debemos especificar el tipo de lenguaje utilizado. Puede ser lenguaje orientado a realizar una búsqueda web (`LANGUAGE_MODEL_WEB_SEARCH`), o lenguaje de tipo general (`LANGUAGE_MODEL_FREE_FORM`).
|`EXTRA_LANGUAGE`       |**Opcional**. Se especifica para hacer el reconocimiento en un idioma diferente al idioma por defecto del dispositivo. Indicaremos el idioma mediante la etiqueta IETF correspondiente, como por ejemplo `"es-ES"` o `"en-US"`|
|`EXTRA_PROMPT`         |**Opcional**. Nos permite indicar el texto a mostrar en la pantalla mientras se realiza el reconocimiento. Se especifica mediante una cadena de texto.|
`EXTRA_MAX_RESULTS`     |**Opcional**. Nos permite especificar el número máximo de posibles resultados que queremos que nos devuelva. Se especifica mediante un número entero.|

Una vez creado el _intent_ y especificados los parámetros,
podemos lanzar el reconocimiento llamando, desde nuestra actividad, a:

```java
startActivityForResult(intent, codigo);
```

Como código deberemos especifica un entero que nos permita identificar
la petición que estamos realizado. En la actividad deberemos definir el _callback_ `onActivityResult`,
que será llamado cuando el reconocimiento haya finalizado. Aquí deberemos
comprobar en primer lugar que el código de petición al que corresponde
el _callback_ es el que pusimos al lanzar la actividad. Una
vez comprobado esto, obtendremos una lista con los resultados obtenidos
de la siguiente forma:

```java
@Override
protected void onActivityResult(int requestCode,
                int resultCode, Intent data) {
    if (requestCode == codigo && resultCode == RESULT_OK) {

        ArrayList<String> resultados =
            data.getStringArrayListExtra(
                RecognizerIntent.EXTRA_RESULTS);

        // Utilizar los resultados obtenidos
        ...
    }
    super.onActivityResult(requestCode, resultCode, data);
}
```


## Síntesis de voz en iOS

La síntesis de voz aparece a partir de iOS 7. Hasta entonces la única forma de implementar síntesis de voz era utilizar la característica de accesibilidad _Voice Over_, lo cual forzaba a tener que tener activada esta característica.

A partir e iOS 7 aparece la clase `AVSpeechSynthesizer` que nos permite sintetizar voz en cualquier aplicación.

En primer lugar deberemos establecer el texto a vocalizar mediante un objeto de tipo `AVSpeechUtterance`:

```objectivec
AVSpeechUtterance *utterance = [AVSpeechUtterance speechUtteranceWithString: @"Hola mundo"];
```

Este objeto nos permite también especificar otras propiedades de la locución, como el _pitch_ o la voz a utilizar.

Una vez definido y configurado el _utterance_, podremos reproducirlo con:

```objectivec
AVSpeechSynthesizer *synthesizer = [[AVSpeechSynthesizer alloc] init];

[synthesizer speakUtterance: utterance];
```

A través de este objeto podremos pausar o reanudar la locución, o conocer si actualmente está reproduciéndose.


## Reconocimiento del habla en iOS

Los dispositivos iOS cuentan con el asistente Siri que utiliza reconocimiento de voz para realizar diferentes operaciones. Estas operaciones se realizan a nivel del Sistema Operativo, y nos permiten utilizar diferentes servicios que proporciona la plataforma, como por ejemplo hacer una llamada, leer los mensajes, o consultar el tiempo que hace.

Lamentablemente, no existe en el momento de la escritura de este texto ninguna API que nos permita integrar la capacidad de reconocimiento del habla de Siri en nuestra aplicaciones. Sin embargo, si que podemos encontrar APIs de terceros que nos proporcionan dicha funcionalidad, como por ejemplo las siguientes:

* *SpeechKit*

http://developer.nuance.com/

* *MindMeld*

https://expectlabs.com/docs/sdks/ios/gettingStarted

* *OpenEars*

http://www.politepix.com/openears/


## Ejercicios



### Síntesis de voz con Text to Speech

En este primer ejercicio vamos a utilizar el motor _Text to Speech_ para crear una aplicación que lea el texto contenido en un `EditText` de la actividad principal. Para ello el primer paso será descargar de las plantillas la aplicación _SintesisVoz_. La aplicación contiene una única actividad. La idea es que al pulsar el botón _Leer_ se lea el texto en el cuadro de edición. Existen dos botones de radio para escoger la pronunciación (inglés o español).

Deberemos seguir los siguientes pasos:


* Inserta el código necesario en el método `initTextToSpeech` para que se lance un `Intent` implícito para comprobar si el motor _Text to Speech_ está instalado en el sistema:


```java
Intent intent = new Intent(Engine.ACTION_CHECK_TTS_DATA);
startActivityForResult(intent, TTS_DATA_CHECK);
```


* En el manejador `onActivityResult` incorporamos el código necesario para inicializar el motor _Text to Speech_ en el caso en el que esté instalado, o para
instalarlo en el caso en el que no lo estuviera.


```java
if (requestCode == TTS_DATA_CHECK) {
	if (resultCode == Engine.CHECK_VOICE_DATA_PASS) {
		tts = new TextToSpeech(this, new OnInitListener() {
    		public void onInit(int status) {
    			if (status == TextToSpeech.SUCCESS) {
    				ttsIsInit = true;
    				Locale loc = new Locale("es","","");
    				if (tts.isLanguageAvailable(loc)
    						    >= TextToSpeech.LANG_AVAILABLE)
    					tts.setLanguage(loc);
    				tts.setPitch(0.8f);
    				tts.setSpeechRate(1.1f);
    			}
    		}
    	});
    } else {
    	Intent installVoice = new Intent(Engine.ACTION_INSTALL_TTS_DATA);
    	startActivity(installVoice);
    }
}
```

> En el código anterior `tts` es un objeto de la clase `TextToSpeech` que ya está definido en la plantilla. La variable booleana `ttsIsInit` tendrá valor `true` en el caso en el que el motor de síntesis de voz se haya inicializado correctamente. La utilizaremos más adelante para comprobar si se puede leer o no un texto. Mediante el objeto `loc` inicializamos el idioma a español, ya que es el botón de radio seleccionado por defecto al iniciar la actividad.


* Añade el código necesario en el método `onDestroy` para liberar los recursos asociados a la instancia de _Text to Speech_ cuando la actividad vaya a ser
destruida:


```java
if (tts != null) {
	tts.stop();
    tts.shutdown();
}
```


* El manejador del click del botón _Leer_ simplemente llama al método `speak`, que será el encargado de utilizar el objeto `TextToSpeech` para leer
el texto en la vista `EditText`. Introduce el código necesario para hacer esto; no olvides de comprobar si
el motor _Text to Speech_ está inicializado por medio de la variable booleana `ttsIsInit`.
* Por último añade el código necesario a los manejadores del click de los botones de radio para que se cambie el idioma a español o inglés según corresponda. Observa cómo se usa
la clase `Locale` en `onActivityResult` para hacer exactamente lo mismo.






