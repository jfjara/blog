---
layout: single
title: "Inmutabilidad"
excerpt: "¿Nunca os ha pasado que, cuando en un entorno laboral tecnológico heredáis lo que las compañías suelen llamar código legacy (o comúnmente, código de mierda arcano -o no tan arcano-), intentáis realizar el seguimiento de la vida de un objeto de dominio, y este pasa por tantas partes y lo modifica tantos procesos que le perdéis la pista a su contenido, o se os escapa por qué leches hay un valor que se esta modificando y no tenéis ni idea de dónde? Pues a mi si me ha pasado. Muchas veces. En muchos proyectos de esos heredados. Y es un verdadero fastidio, aunque en estos casos suele ocurrir, aparte de no aplicar la inmutabilidad de los objetos, que existan muchos antipatrones que se deberían de evitar a toda costa."
date: 2023-01-25
classes: wide
header:
  teaser: /assets/images/htb-writeup-delivery/delivery_logo.png
  teaser_home_page: true
  icon: /assets/images/hackthebox.webp
categories:
  - pattern
  - practise
tags:  
  - pattern
  - java
  - POO
---

![](/assets/images/htb-writeup-delivery/delivery_logo.png)

¿Por qué siempre hago que mis objetos sean inmutables? ¿Qué es la inmutabilidad y por qué es tan importante? ¿Qué ceno hoy?

A excepción de la última pregunta, a las demás podemos darle respuesta en este artículo. Así que agarraos bien que comenzamos.

## Que no me modifiques, leñe!

¿Nunca os ha pasado que, cuando en un entorno laboral tecnológico heredáis lo que las compañías suelen llamar código legacy (o comúnmente, código de mierda arcano -o no tan arcano-), intentáis realizar el seguimiento de la vida de un objeto de dominio, y este pasa por tantas partes y lo modifica tantos procesos que le perdéis la pista a su contenido, o se os escapa por qué leches hay un valor que se esta modificando y no tenéis ni idea de dónde? Pues a mi si me ha pasado. Muchas veces. En muchos proyectos de esos heredados. Y es un verdadero fastidio, aunque en estos casos suele ocurrir, aparte de no aplicar la inmutabilidad de los objetos, que existan muchos antipatrones que se deberían de evitar a toda costa.

Pero, ¿qué es la inmutabilidad? Diremos que es una propiedad de los objetos que impide que sus propiedades se vean alteradas (intencionalmente o no). Muy bien, y ahora nos preguntamos, ¿qué beneficios puede llegar a tener esta forma de definir las clases?

Nos alejaremos de las tediosas listas indicando los beneficios (que no son pocos) de este método de implementación para explicar aquellos que, bajo mi humilde opinión, son los más importantes.

Naces, te vuelves inmutable e impasible a la vida y la diñas

Con este título te puedes hacer una idea del objetivo: controlar el estado del objeto durante todo su ciclo de vida. Así, no no llevamos sorpresas. Sabemos lo que nuestro objeto contiene desde que nace hasta que lo expulsamos del server (ya sea con un delete en C++ o que la JVM saque la escoba para barrer a los objetos desechados).

## Dámelo todo! Mejor no

Encapsulación de los atributos y solo exponer al exterior lo justo y necesario. Porque no vas por la calle con los calzoncillos por fuera, ¿verdad? Y ya de añadir modificadores de acceso ni hablemos. Esto nos da mucha seguridad y una robustez a nuestras clases, evitando en gran medida errores y ocultando parte del código de la clase.

La palabra final. Ideal para títulos de películas y para Java

La palabra reservada final en Java se utiliza para implementar objetos inmutables. Concretamente, indica que una referencia de memoria no pueda ser modificada (la referencia de memoria es eso que, cuando depuramos e inspeccionamos variables, indican la posición de memoria a la que apunta y se aloja el objeto en cuestión, suelen tener un formato tal que así: {MiClase@12345}). O sea, que esa referencia es una variable alojada en la memoria Stack (o pila) con el valor de la posición de memoria donde se encuentra el objeto de tipo «MiClase» en la memoria Heap. Traduciendo: si hacemos final una variable lo que estamos haciendo inmutable es la referencia, no el contenido del objeto en si (a menos que declares final los tipos primitivos, que esos si se almacenan en el Stack). Veamos:

```

final int myAge = 42;
//y el año que viene...
myAge++;  // Oops. Error de compilación

final Date now = new Date();
now = new Date();  //Oops. Error de compilación again.

```

Muy bien, vemos que es efectivo en los tipos básicos, pero, ¿y si hacemos lo siguiente con una clase más elaborada?

```

final Date now = new Date();
now.setTime(System.CurrentTimeInMillis());

```

What? no falla. ¿Pues no era inmutable la variable now gracias a final? Pues si, es inmutable la referencia, ababol. O sea, lo que se encuentra en el Stack, pero el contenido del objeto se encuentra en el Heap. Y le damos acceso a modificar el contenido con el setTime. Así que esa declaración como que no es muy inmutable.

Otro uso de la palabra final viene a la hora de establecerlo en la declaración de la clase. Y ¿para qué haces eso? Para indicar que nuestra clase no se puede extender por una clase hija y acceda a nuestras propiedades.

```

public class final MiClase {
   
   final private int miVariable;

   public MiClase(final int miVariable) {
      this.miVariable = miVariable;
   }

   int getMiVariable() {
      return miVariable;
   }

} 

public class MiOtraClase extends MiClase {
...
}   // Oops. Error de compilación. No se puede heredar de una clase final 

```

Buena pinta esto para la inmutabilidad eh? Pues no tanto en algunos casos. Veamos:

```

public final class MiClase {
   
   final private Date edad;

   public MiClase(final Date edad) {
      this.edad = edad;
   }

   public Date getEdad() {
      return edad;
   }

}

```

Esto es, en teoría, una clase inmutable. Pero tenemos el mismo problema que vimos anteriormente:

```

MiClase miClase = new MiClase(new Date());
miClase.getDate().setTime(System.currentTimeInMillis()); 

```

Lo mismo que en el caso anterior. Nuestro objeto parecía inmutable, pero se ve que no lo es. ¿Es problema de nuestro diseño? Pues en parte si, ya que la clase Date no es inmutable. Para resolver este problema podemos usar clases de java.time, ya que son inmutables, o con este sencillo truquito:

```

public final class MiClase {
   
   final private Date edad;

   public MiClase(final Date edad) {
      this.edad = edad;
   }

   public Date getEdad() {
      return new Date(this.edad.getTime());
   }

}

```

De este modo, devolvemos una copia del objeto edad, por mucho que modifiquemos el valor del atributo del objeto no se modifica.

Acabamos de mostrar una forma de crear clases inmutables. Veamos otras formas, más enfocadas a clases que tengan más atributos. Pongamos que tenemos que implementar una clase que tenga muchos atributos. Bueno, pongamos no muchos, 6 atributos. No son muchos para una clase, aunque si lo son para un constructor o un método. Citando a Robert C. Martin en su libro "Clean Code":

> El número ideal de argumentos para una función es cero. Después uno (monádico) y dos (diádico). Siempre que sea posible, evite la presencia de tres argumentos (triádico). Más de tres argumentos (poliádico) requiere una justificación especial y no es muy habitual. — Robert C. Martin, después de ver una función con 8 parámetros de entrada.

Por extensión, cambiemos la palabra método por constructor. Es poco legible un constructor de este tipo:

```

public Persona(String nombre, String primerApellido, String segundoApellido, String dni, int edad, Date fechaNacimiento, String direccion, ...)

```

A la hora de instanciar una clase de tipo Persona puede ser tan doloroso como un martillazo en el pie. Solucionemos este problema y apliquemos inmutabilidad con un bonito patrón Builder:

```

public final class MiClase {
   private int miVariable1;
   private int miVariable2;
   ...
   private int miVariableN;


   public MiClase(Builder builder) {
	this.miVariable1 = builder.getMiVariable1();
        this.miVariable2 = builder.getMiVariable2();
        ...
   }

   public int getMiVariable1() {
      return miVariable1;
   }
   
   ...

}

public class MiClaseBuilder {

   private int miVariable1;

   private int miVariable2;
        
   ...

   private int miVariableN;

   public Builder miVariable1(int miVariable1) {
      this.miVariable1 = miVariable1;
      return this;
   }

   public Builder miVariable2(int miVariable2) {
      this.miVariable2 = miVariable2;
      return this;
   }
		
   ...
	
   public int getMiVariable1() {
     return miVariable1;
   }

   public int getMiVariable2() {
     return miVariable2;
   }

   public MiClase build() {
      return new MiClase(this);
   }

}

```

Mucho más elegante. Veamos la instanciación:

```

MiClase miClase = new MiClaseBuilder()
                         .miVariable1(1)
                         .miVariable2(2)
                         ...
                         .miVariableN(N).build();

```

Hay más formas de crear dicho patrón, hemos elegido una de ellas para este ejemplo de implementación inmutable. Sobre todo, recordar que de nada nos sirve implementar la inmutabilidad si en los métodos de acceso a los atributos devolvemos elementos mutables.

En el caso de tener que modificar un valor de una propiedad del objeto inmutable, ¿cómo lo hago? Pues para garantizar las características de los objetos inmutables lo lógico siempre sería realizar una copia del objeto y añadir la modificación correspondiente. Y para terminar, veamos un ejemplo implementado con la librería

## Lombok

```

@Getter
@Builder(toBuilder = true)
public class Place {
    private String id;
    private CoordinateDto coordinate;
    private String name;
    private String address;
    private String city;
    private String country;
    private String postalCode;
    private String typeService;
    @Getter(AccessLevel.NONE)
    private List<Service> services;
   
    public List<ServiceDtoEnum> getServices() {
        return Collections.unmodifiableList(services);
    }
}

```

En este caso habilitamos la creación del patrón Builder con el tag @Builder anotado con toBuilder=true (devuelve el Builder relleno con los valores actuales de la instancia, ideal para hacer copias del objeto. El @Getter nos genera automáticamente todos los getters de acceso a los atributos de clase. En el caso de del getter de la lista le indico que no se autogenere (AccessLevel.NONE) ya que si devolvemos en el getter la lista tal como está definida podemos modificarla ya que la clase List no es inmutable. Para ello creo mi propio getter y devuelvo una lista inmutable a raíz de la del objeto mediante Collections.unmodifiableList.

Otra opción con Lombok seria marcar la clase con el tag @Value, el cual nos crea la clase inmutable (pero sin patrón Builder). Tal como indica la propia documentación de Lombok, @Value equivale a añadir @Getter @FieldDefaults(makeFinal=true, level=AccessLevel.PRIVATE) @AllArgsConstructor @ToString y @EqualsAndHashCode.

Y poco más que comentar, hemos repasado las bondades de usar inmutabilidad en nuestros objetos así como una pequeña introducción al patrón Builder. Solo me queda responder a lo de que cenaba hoy. Sandwich.

