---
layout: single
title: "Problema en Circuit Breaker reactivo cuando se lanzan excepciones fuera del ámbito reactivo con Spring Boot y Webflux"
excerpt: "En el siguiente post comentamos por qué no se ejecuta el fallback de un circuit breaker reactivo de Resilience4j cuando se lanza una excepción desde un servicio o repositorio externo."
date: 2023-07-09
classes: wide
header:
  teaser: /assets/images/htb-writeup-delivery/delivery_logo.png
  teaser_home_page: true
  icon: /assets/images/hackthebox.webp
categories:
  - reactive
  - java
  - POO
  - programming
  - circuit breaker
  - resilience4j
tags:  
  - springboot
  - java
  - reactive
  - webflux
  - resilience4j
  - circuit breaker
---

![](/assets/images/htb-writeup-delivery/delivery_logo.png)

En el siguiente post comentamos por qué no se ejecuta el fallback de un circuit breaker reactivo de Resilience4j cuando se lanza una excepción desde un servicio o repositorio externo no envuelto por una entidad Mono o Flux.

*Esta entrada al tratarse esctrictamente de una práctica concreta no entraremos en detalles sobre lo qué es la programación reactiva ni sus características. Ya existen muchas webs donde se explica muy bien, no entraremos en esa competencia. Nos dedicaremos exclusivamente a resolver un problema práctico donde necesitemos el uso de dicha paradigma de programación. Tampoco comentaremos el funcionamiento de un Circuit Breaker.*

En ocasiones al tener que utilizar apis o servicios de terceras partes debemos de tener en cuenta como se gestionan las excepciones en dichos módulos. Pongamos el ejemplo siguiente: un cliente de una api que autogeneramos mediante las tools de OpenApi y este nos crea un cliente reactivo el cual consume una api externa. Pongamos que dicho cliente, al ser reactivo, nos devolverá las posibles excepciones envueltas en un elemento reactivo Mono mediante *Mono.error*. Pero se puede dar el caso que el cliente valide las entradas de parametros que le pasamos, y este nos devuelva excepciones de una manera más clásica, esto es lanzando excepciones mediante la instrucción *throw*. Ahora, tenemos un *Circuit Breaker* en nuestro servicio que utiliza dicho cliente con un fallback configurado. Observamos que cuando el cliente nos devuelve los *Mono.error* el *Circuit Breaker* se activa y el *fallback* hace su trabajo. Pero, ¿qué ocurre cuando el validador del cliente comprueba los parámetros y uno de ellos no cumple las reglas del cliente? Este nos lanza un *throw new ACustomException* (por poner un ejemplo) y vemos que el *Circuit Breaker* no se activa y el fallback no entra en juego, dejando esa excepción sin controlarse y sin actuar el *Circuit Breaker*.

En este ejemplo quizás no interesa abrir circuito, ya que por una validación errónea no debería de ser motivo para impedir nuevas peticiones al servicio, pero pongamos en este ejemplo que es un requisito, si la validación no se cumple también debemos abrir circuito para impedir consumir dicho servicio.

Primero veamos la explicación de por qué un circuito reactivo no parece hacer caso a las excepciones que se lanzan de manera tradicional. Como bien hemos comentado, estamos trabajando con paradigma reactivo, por lo que las excepciones deberiamos gestionarlas mediante la instrucción error de las clase *Mono* o *Flux*, pero también se puede dar el caso de encontrar lanzamiento de excepciones sin estar envueltas en dicho método. Véase el método del cliente:

```

Mono<String> getText(final Long id) {
    if (id == null) {
        return new ValidationException("The id is null");
    }
    return client.invoke(id);
}

```

Y este es nuestro repositorio donde se invoca a dicho servicio y donde tenemos el *Circuit Breaker* definido:

```

@CircuitBreaker(name = "circuitBreakerService", fallbackMethod = "myFallback")
public Mono<MockServiceResponse> getTextFromApi(final Long id) {
    return client.getText(id);
}

```

Si en algún momento el parámetro 'id' llega al servicio con valor null, la excepción se lanzará, y veremos que el *circuit breaker* no se activa. Pero, ¿por qué?

La respuesta se basa en la *reactividad* de nuestro servicio y el comportamiento del *circuit breaker*. El método del cliente de la api autogenerada si no cumple la validación lanza una excepción en lugar de devolver un *Mono.error* por lo que no da tiempo a montar dicha estructura y en nuestro *circuit breaker*, el cual espera un *Mono.error* para poder activarse no lo recibe e ignora la excepción lanzada.

Ahora bien, **¿cómo podemos resolver este problema?** Veamos tres opciones para resolverlo:

* **La primera es envolver nuestro método del repositorio con un try-catch:**

```

@CircuitBreaker(name = "circuitBreakerService", fallbackMethod = "myFallback")
public Mono<String> getTextFromApi(final Long id) {
    try {
        return client..getText(id);
    } catch (ValidationException e) {
        return Mono.error(e);
    }
}

```

Esta solución, en mi opinión, **es la menos correcta de todas**. El *circuit breaker* en si es un un bloque *try-catch* (sin haber entrado en las entrañas de **Resilience4j** entiendo que funciona mediante aspectos, y ahi es donde tendrá el *try-catch* y la llamada a *fallback* en caso de error que se hará mediante reflexión, si no es así se le acercará) y como tal no tiene sentido anidar estos bloques de instrucciones. Aunque se hace "por debajo", el resultado queda una mezcla de paradigma reactivo con programación estructurada que no atrae demasiado. Veamos más opciones.

* **La siguiente es envolver la respuesta en una entidad Mono, de ese modo podremos manejar las excepciones de manera reactiva. Para ello podemos utilizar los métodos Mono.defer o Mono.create:**

```

@CircuitBreaker(name = "circuitBreakerService", fallbackMethod = "myFallback")
public Mono<String> getTextFromApi(final Long id) {
    return Mono
          .defer(() -> client.getText(id))
          .doOnError(ex -> {
              throw new MyRuntimeException(ex);
          });
}

```

```

@CircuitBreaker(name = "circuitBreakerService", fallbackMethod = "myFallback")
public Mono<String> getTextFromApi(final Long id) {
  return Mono
          .create(sink -> {
              client.getText(id)
              .doOnError(ex -> {
                  sink.error(new MyRuntimeException(ex));
              })
              .subscribe(v -> sink.success(v));
  });
}

```

Vemos que la opción *defer* es más limpia que *create*, aunque con *create* tenemos más posibilidades ya que controlamos la señal a emitir a los consumidores. Según la documentación de **Webflux**, podemos usar *Mono.just*, *Mono.defer* y *Mono.create* para publicar datos a los consumidores que se suscriben al *Mono*, pero se recomienda siempre ese orden, uso de instrucción más sencilla a más compleja. Pero, ¿por qué no podemos solucionar este problema con *Mono.just*? La respuesta es la *'evaluación en diferido'*. Mientras *defer* y *create* tienen este comportamiento *just* no. *Mono.just* se usa para publicar un dato una vez ya calculado (este se establece en el momento de la composición), mientras los otros montan la *'canalización reactiva'* trabajando con una expresión en lugar de un valor concreto como hace *just*, resolviéndose en el momento que nos suscribimos.

* **Otra opción es realizar una doble validación, concretamente realizar la misma validación de la api previamente a la llamada y asi controlar nosotros mismos la gestión de esa excepción: nos aseguramos que si realizamos la llamada nunca esas excepciones se van a lanzar.**

Deseo que este pequeño artículo os ayude. Si conocen otro método para resolver este problema, estaré encantado de conocerlo, pueden escribir a **juanfranciscojara@gmail.com**

