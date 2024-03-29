# Generación de Código

![](../.gitbook/assets/codegen.png)

## 1. Introducción

En esta asignación, ustedes van a implementar un generador de código para COOL. Cuando hayan terminado satisfactoriamente esta parte, ustedes van a tener un compilador de COOL totalmente funcional.

{% hint style="info" %}
Esperamos que se den cuenta, que al finalizar esta fase, resolvieron un problema de ingeniería sustancialmente complejo. De ahora en adelante cualquier reto en cuestión de diseño e implementación debería de ser por lo menos un poco más fácil.
{% endhint %}

El generador de código hace uso del _Abstract Syntax Tree_ \(AST\) construido durante la fase 2 del proyecto \(Parser\) y la fase 3 del proyecto \(Semantic\). Su generador de código debería de producir código de ensamblador, que representa cualquier programa de COOL correcto para la arquitectura **RISC-V**.

En esta fase no hay recuperación de errores, en la generación de código ya podemos estar seguros de que cualquier programa erroneo de COOL ya ha sido detectado en las fases anteriores del compilador \(Lexer, Parser, Semantic\).

Así como en la fase de análisis semántico, esta última parte tiene decisiones de diseño que ustedes tienen que pensar bien para lograr completarla. Su programa va estar correcto si el código generado funciona correctamente, ¿cómo lo van a lograr? ustedes tienen que ver que camino seguir, que decisiones tomar, que técnicas utilizar, etc. Les recomendamos algunos convenios que nosotros creemos les van a hacer la vida más fácil, pero no están obligados a seguir nuestras recomendaciones. Esta última parte, comparte mucha de la infrastructura de la fase anterior. **Por favor, empiecen lo más pronto posible**.

Es bastante importante que para que ustedes puedan lograr generar código correctamente, ustedes tengan claro ya el compartamiento interno de COOL, como interactura con el runtime system \(_**runtime.s**_\) y el código generado. El compartamiento esperado de los programas de COOL están definidos en las operaciones semánticas de COOL que están definidas en las sección 13 del manual de referencia de COOL. Recuerden que esta solo es una especificación del significado del lenguaje, no como se implementa un generador de código. La interfaz entre el runtime system y el código generado está dado en _**runtime.s**_ y el manual _Cool Runtime System_. Revisen esos documentos y archivos para que tengan una clara visión de todos los requerimientos del runtime system en el código generado. Hay un montón de información en esta asignación y en los documentos mencionados, y ustedes tienen que saber de estos para poder generar código de manera correcta. Por favor lean _cuidadosamente_ para entender a detalle todo.

## 2. Ambiente

Como ustedes _**no**_ van a generar código ensamblador de MIPS como lo hace **coolc** originalmente, sino de **RISC-V**, necesitamos preparar el ambiente de trabajo instalando el simulador de RISC-V **Jupiter** e instalando el compilador de referencia **coolc-rv**, haciendo lo siguiente en un una terminal \(CTRL+T\):

```bash
sudo add-apt-repository ppa:andrescv/jupiter
sudo apt update
sudo apt install jupiter
```

Los comandos anteriores instalarán jupiter, para verificar si se instaló correctamente deberían de probar lo siguiente:

```bash
jupiter -h
       __          _ __
   __ / /_ _____  (_) /____ ____
  / // / // / _ \/ / __/ -_) __/
  \___/\_,_/ .__/_/\__/\__/_/
           /_/

RISC-V Assembler & Runtime Simulator
               v3.1

usage: jupiter [options] <files>

[General Options]
  -h, --help               show Jupiter help message and exit
  -v, --version            show Jupiter version
  -l, --license            show Jupiter license

[Simulator Options]
  -b, --bare               bare machine (no pseudo-instructions)
  -s, --self               enable self-modifying code
  -e, --extrict            assembler warnings are consider errors
  -g, --debug              start debugger
      --start <label>      set global start label (default: __start)
      --hist <size>        history size for debugging

[Cache Options]
  -c, --cache              enable cache simulation
      --assoc <assoc>      cache associativity as a power of 2 (default: 1)
      --block-size <size>  cache block size as a power of 2 (default: 16)
      --num-blocks <num>   number of cache blocks as a power of 2 (default: 4)
      --policy <policy>    cache block replace policy (LRU|FIFO|RAND) (default: LRU)

[Dump Options]
      --dump-code <file>   dump generated machine code to a file
      --dump-data <file>   dump static data to a file

Please report issues at https://github.com/andrescv/Jupiter/issues
```

Ahora tienen que instalar el compilador de referencia _**cooc-rv**_ y el runtime system, bajen el siguiente archivo:

{% file src="../.gitbook/assets/install.sh" %}

Luego que lo hayan descargado, ejecuten los siguientes comandos en una terminal:

```bash
chmod +x install.sh
./install.sh
```

Para comprobar que se haya instalado todo correctamente, compilen y corran un programa simple de COOL:

```python
--(main.cl)
class Main inherits IO {

  main(): Object {
    out_string("hello world")
  };

};
```

Entonces utilizando el compilador que acabamos de instalar, podemos generar código de RISC-V:

```bash
coolc-rv main.cl # genera un archivo main.s
```

Por último pueden probar el programa de la siguiente manera:

```bash
jupitercl main.s
hello world
COOL program successfully executed

Jupiter: exit(0)
```

{% hint style="info" %}
Noten que se está haciendo uso del comando _**jupitercl**_, este a diferencia del comando _**jupiter**_ carga automáticamente el runtime system de COOL.
{% endhint %}

## 3. Archivos y Directorios

Para empezar, en su carpeta que contiene todas las demás fases como se explicó en:

{% page-ref page="instalacion-material.md" %}

 clonen el siguiente repositorio de Git:

```bash
git clone https://github.com/CC-4/PA4.git
```

Casi todos estos archivos han sido explicados en las fases anteriores del proyecto. Esta es una lista de los archivos que ustedes tal vez quieran modificar. Ustedes ya tienen que estar familiarizados con la mayoría de estos archivos por las fases anteriores.

* **CgenClassTable.java** y **CgenNode.java**: Estos archivos proveen una implementación del grafo de herencia para el generador de código. Ustedes van a tener que completar `CgenClassTable` para poder construir su generador de código. Ustedes pueden utilizar el código que se les provee o pueden reemplazarlo con el suyo o con el que hicieron en la fase anterior.
* **StringSymbol.java**, **IntSymbol.java**, y **BoolConst.java**: Estos archivos proveen soporte para las constantes de COOL. Ustedes van a tener que completar el método para generar definiciones de estas constantes.
* **cool-tree.java**: Este archivo contiene la definición del los nodos del AST. Ustedes van a tener que añadir rutinas de generación de código `code(PrintStream)` para todas las expresiones en este archivo. **El generador de código es mandado a llamar, empezando por** `code(PrintStream)` **que está en la clase program**. Ustedes tal vez quieran añadir más métodos, pero no modifiquen los métodos ya existentes.
* **TreeConstants.java**: Como antes, este archivo define algunas constantes útiles, siéntanse libres de añadir las suyas y lo que quieran.
* **CgenSupport.java**: Este archivo contiene soporte general para generar código. Van a encontrar una serie de funciones que se vuelven bastantes útiles para emitir instrucciones de RISC-V, aquí está todo lo que necesitan. Pueden añadir también las que ustedes quieran, pero no modifiquen nada de lo que ya está definido.
* **example.cl**: Este es su archivo de prueba para ver si lo que genera su generador de código funciona correctamente. Hagan bastantes pruebas con este archivo como lo requieran.

## 4. Diseño

Antes de continuar, les sugerimos que de verdad lean el documento The Cool Runtime System para que estén cómodos con los requerimientos que necesita implementar su generador de código impuestos por el runtime system. En consideración a su diseño, así a grandes rasgos, su generador de código deberá hacer las siguientes tareas:

* Determinar y emitir código para las constantes globales, como los prototipos de objeto.
* Determinar y emitir código para las tablas globales, como la `class_nameTab`, la `class_objTab` y las dispatch tables.
* Determinar y emitir código para la inicialización de los métodos y atributos de cada clase.
* Determinar y emitir código para cada método de cada clase.

Hay varias maneras posibles de implementar un generador de código. Una estrategia razonable es hacer el generador de código en dos pasadas. La primera pasada decide el layout para cada clase, particularmente el offset en donde cada atributo es guardado en un objeto. Utilizando esta información, la segunda pasada recursivamente visita cada feature y genera código de un stack machine para cada expresión.

Hay un número de cosas que ustedes tienen que mantener en mente mientras diseñan su generador de código:

* Su generador de código tiene que funcionar correctamente con el Runtime de COOL \(_**runtime.s**_\), que es explicado a detalle en _Cool Runtime System Manual_.
* Ustedes tiene que tener claro como funciona la semántica operacional del runtime de COOL. La semántica es descrita informalmente en la primera parte del _Cool Reference Manual_ y una descripción precisa de cómo los programas de COOL deberían de comportarse está explicada en la sección 13 del manual de COOL.
* Deberían entender el set de instrucciones de RISC-V, y tener una vista general de las operaciones que se pueden efectuar con `Jupiter`.
* Ustedes tienen que decidir que invariantes su generador de código puede esperar u observar \(es decir, qué registros se van a salvar, cuales se pueden sobreescribir, convenios, etc\). Ustedes van a encontrar bastante útil, ver la información que se les enseño en clase.

### 4.1 Runtime Error Checking

Al final del manual de COOL existe una lista de errores que pueden terminar un programa antes de lo esperado. De esta lista su generador de código debería de ser capaz de atrapar las primeras tres: **dispatch on void**, **case on void** y **missing branch** e imprimir un mensaje de error antes de abortar. Atrapar errores de **substring out of range** y **heap overflow** es responsabilidad del runtime system que está en **runtime.s**. Ya que estamos utilizando RISC-V, es necesario que validen en el código generado la división entre 0, mandando a llamar a **\_div\_by\_zero** que está definida en **runtime.s**, vean como _**coolc-rv**_ maneja estos casos.

### 4.2 Garbage Collection

El equipo de CC-4 que tradujo el runtime de COOl de MIPS a RISC-V, no implementó un garbage collector complejo como el original que está en el runtime de MIPS \(un _garbage collector generacional_\), debido al tiempo y porque ustedes realmente no van a interactuar mucho con el garbage collector en este curso \(incluso si hicieran esta fase para la versión de MIPS\). El código que tienen que tener para que el GC funcione correctamente ya se las damos en el código base y no tienen que preocuparse por esto, pero si se tienen que preocupar porque su generador de código funcione bien y sea compatible con el runtime system de COOL.

## 5. Probando su CodeGen

Van a necesitar un analizador léxico, sintáctico y semántico para probar su generador de código. Pueden utilizar sus implementaciones a estas fases o utilizar las que nosotros les proveemos. Por defecto, las que nosotros les proveemos son utilizadas, para cambiar este comportamiento tienen que cambiar los archivos `lexer`, `parser` y `semant` con sus propias implementaciones. De todas maneras el autograder principal de este proyecto utiliza los analizadores del compilador de COOL **coolc**.

Ustedes van a tener que correr su generador de código utilizando `./mycoolc`, que es un shell script que pega el generador de código que ustedes implementaron con el resto de fases. Tomen en cuenta que `./mycoolc` puede tomar una bandera -c para depurar el generador de código, utilizando esta bandera, se cambia la bandera debug a verdadero, que está en **Flags.java**. Agregar código para producir información de depuración es tarea de ustedes si quieren utilizarlo.

### 5.1 Jupiter

El ejecutable de Jupiter `jupitercl` es el simulador para la arquitectura RISC-V en donde pueden probar su generador de código. Tiene muchas funcionalidades que les permite examinar el estado de la memoria, símbolos globales y locales, registros, etc del programa. Ustedes también pueden poner breakpoints y hacer step de su programa. Para poder utilizar el debugger de Jupiter ustedes pueden hacer lo siguiente:

```text
jupitercl -g <archivo.s>
>>> help
Available commands:

[General Commands]
 help/?                - show this help message
 exit/quit/q           - exit the simulator and debugger
 !                     - execute previous command

[Display]
 rvi                   - print all RVI registers
 rvi <reg>             - print RVI register reg
 rvf                   - print all RVF registers
 rvf <reg>             - print RVF register reg
 memory <addr>         - print 12 x 4 cells of memory starting at the given address
 memory <addr> <rows>  - print rows x 4 cells of memory starting at the given address
 globals               - print global symbol table
 locals                - print local symbol tables

[Execution Control]
 step/s                - continue until another instruction reached
 backstep/b            - back to the previous instruction
 continue/c            - continue running
 reset                 - reset

[Breakpoints]
 break <addr>          - set a breakpoint at the given address
 clear                 - delete all breakpoints
 delete <addr>         - delete breakpoint at the given address
 list                  - show defined breakpoints
```

La documentación de Jupiter la pueden encontrar [aquí](https://docs.riscvsim.com/).

## 6. Autograder

El script que califica su proyecto, ya viene por defecto en los archivos bases que clonaron anteriormente. Cada vez que ustedes van ejecutar el autograder ustedes deberían hacer lo siguiente:

```text
./pa4-grading.py
```

Si ustedes quieren probar su parser utilizando su fase 1, 2 y 3:

{% file src="../.gitbook/assets/pa4-grading-all.sh" caption="pa4-grading-all.sh" %}

y ejecutar los siguientes comando:

```bash
chmod +x pa4-grading-all.sh
./pa4-grading-all.sh
```

