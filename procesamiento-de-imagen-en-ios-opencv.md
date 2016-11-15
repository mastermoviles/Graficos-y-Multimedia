# Procesamiento de imágenes en iOS: OpenCV

En esta sección vamos a ver como llevar a cabo procesamiento de imágenes en iOS utilizando la librería de visión por computador **OpenCV**.

## OpenCV

[OpenCV](http://opencv.org/) es una librería de código abierto para visión por computador. El principal objetivo de esta librería es ofrecer algoritmos de visión artificial que funcionen en tiempo real. Originalmente OpenCV fue desarrollada por el departamento de investigación de Intel, pero más tarde recibió soporte de la empresa [Willow Garage](http://www.willowgarage.com/) y actualmente es mantenido por la empresa [Itseez](https://vk.com/itseez). Se trata de una librería multi-plataforma (Windows, Linux, MacOSX, iOS, Android, ... )y se puede utilizar libremente al ser desarrollada bajo una licencia de código abierto BSD. 

Áreas de aplicación librería OpenCV:
* Egomotion
* Sistema de detección/reconocimiento facial
* Reconocimiento de gestos
* Segmentación
* Calibración de cámaras
* Tracking
* Algoritmos estéreo
* Structure from Motion (SfM)
* Retoque de imágenes (Balance de blancos, HDR, constrate, etc)
* ...

Además, OpenCV cuenta con una librería de aprendizaje automático que contiene métodos tradicionales de clasificación y regresión:
* Arboles de decisión
* Redes neuronales artificiales
* k-vecinos más cercanos
* Random forest
* Support vector machine (SVM)
* ...

## OpenCV en iOS

OpenCV es una librería escrita en C/C++, por lo que utilizando Objective-C no hay mucho problema para su integración en iOS. Sin embargo, con Swift la cosa es ligeramente más complicada, y necesitaremos utilizar una cabecera puente en nuestro proyecto para acceder a funciones de OpenCV. Por ello tendremos que escribir un poco de Objective-C++ en nuestros proyectos.

Para utilizar OpenCV en iOS tendremos que descargarnos la versión correspondiente para esta plataforma de la web oficial. En concreto vamos a utilizar la versión 2.4.13 para iOS. [Enlace descarga](https://sourceforge.net/projects/opencvlibrary/files/opencv-ios/2.4.13/opencv2.framework.zip/download).

El siguiente paso será crear un nuevo proyecto en xcode y añadir las librerías necesarias para usar OpenCV en nuestro proyecto. Creamos un proyecto nuevo (Single View) y simplemente arrastramos al explorador del proyecto el fichero `opencv2.framework`, nos aparecerá un diálogo para importarlo en el proyecto. Marcamos la opción copiar si es necesario.

![](imagenes/opencv/ios_opencv_import_lib.png)

Con este paso ya tendremos OpenCV añadido a nuestro proyecto, pero para poder utilizarlo tendremos que crear una cabecera puente (Bridging header), que nos permita utilizar código Objetive-C++ desde código Swift. Para crear la cabecera puente, vamos a File > New > File (⌘N) y en iOS > Source seleccionamos `Cocoa Touch Class`. Lo llamaremos OpenCVWrapper, en este fichero escribiremos el código que servirá de envoltorio para utilizar funciones OpenCV en Objective-C++. Al crear este fichero Xcode nos sugerirá crear la cabecera puente: `nombreproyecto-Bridging-header.h`. Todas las librerías que incluyamos desde este fichero header, las clases y métodos definidas en las mismas serán accesibles desde nuestro código en Swift.

Con estos pasos habremos creados tres nuevos ficheros en nuestro proyecto: `OpenCVWrapper.h`, `OpenCVWrapper.m` y `nombre-proyecto-Bridging-Header.h`. Como hemos comentado, la cabecera puente será nuestra interfaz para usar funciones de OpenCV, por lo tanto tenemos que añadir el envoltorio que hemos creado: 

```
#import "OpenCVWrapper.h"
```

