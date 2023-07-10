---
layout: single
title: "Crear anotaciones customizadas con lectura y comparación de propiedades de configuración"
excerpt: "En el siguiente post explicaremos como crear anotaciones customizables que leen propiedades de configuración de la aplicación y se comparan con el valor definido en dicha anotación."
date: 2023-07-10
classes: wide
header:
  teaser: /assets/images/htb-writeup-delivery/delivery_logo.png
  teaser_home_page: true
  icon: /assets/images/hackthebox.webp
categories:
  - java
  - programming
  - spring boot
tags:  
  - springboot
  - java
  - annotations
  - configuration
---

![](/assets/images/htb-writeup-delivery/delivery_logo.png)

En el siguiente post explicaremos como crear anotaciones customizables que leen propiedades de configuración de la aplicación y se comparan con el valor definido en dicha anotación. 

En algunas ocasiones nos interesa poder cargar al application context de spring unos determinados beans u otros dependiendo de un valor de configuración. Para entender mejor el caso veamos un ejemplo:

*Tenemos una aplicación donde dependiendo de un valor de configuración (app.database) podamos cargar unos determinados beans dependiendo el tipo de base de datos que manejemos. En este ejemplo nuestra aplicación estará preparada para postgresql y mongodb.*

Según el enunciado, tenemos los siguientes beans en nuestra clase de configuración de base de datos:


```
@Configuration
public class DatabaseConfig {

    @Bean
    UserRepository userPostgreSQLRepository() {
        return new UserPostgreSQLRepository();
    }

    @Bean
    UserRepository userMongoDbRepository() {
        return new UserMongoDbRepository();
    }

}

```

Nos interesa solo declarar en el application context uno de esos beans dependiendo de la base de datos que tengamos definida en nuestro fichero de propiedades del proyecto. Es por ello que vamos a crear una anotación para marcar qué repositorio vamos a cargar dependiendo de dicha configuración. 

* **Crear nuestra condición que se encarga de comprobar el valor del properties y de la anotación y compararlas:**

```

public class DatabaseCondition implements Condition {

    private static final String APP_DATABASE_PROPERTY = "app.database";
    private static final String VALUE_ANNOTATION_METHOD = "databaseName";

    @Override
    public boolean matches(ConditionContext conditionContext, AnnotatedTypeMetadata annotatedTypeMetadata) {

        var databasePropertyValue = conditionContext.getEnvironment().getProperty(APP_DATABASE_PROPERTY);

        if (databasePropertyValue == null) {
            throw new DatabaseConfigNotFoundException();
        }

        var databaseOnConditionalAnnotation = annotatedTypeMetadata.getAllAnnotationAttributes(DatabaseOnConditional.class.getName());
        var databaseAnnotationValue = (String) databaseOnConditionalAnnotation.getFirst(VALUE_ANNOTATION_METHOD);

        return databaseAnnotationValue.equals(databasePropertyValue);
    }
    
}


```

Como podemos observar se implementa la interfaz *Condition* con su método *matches*. Con los parámetros de entrada de dicho método podemos acceder a contexto de la aplicación obteniendo la información del fichero de propiedades así como la información de la anotación y su valor establecido (esto lo vemos en el siguiente punto). Luego simplemente comparamos dichos valores y obtendremos si dicha comprobación se cumple. En caso de no encontrar la propiedad en el fichero de propiedades lanzamos una excepción propia que hereda de *RuntimeException*.

* **Creamos la anotación que indicará el tipo de base de datos del bean**

Para ello, creamos un anotación que nos permitirá realizar dicha carga.

```

@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.METHOD})
@Conditional(DatabaseCondition.class)
public @interface DatabaseOnConditional {

    String databaseName();
    
}

```

La anotación *Retention* nos indica cuando se chequea la anotación, en este caso en tiempo de ejecución. *Target* nos indica los lugares donde podemos aplicar nuestra anotación, nosotros la vamos a usar solo para carga de beans, por lo que anotamos que solo la usaremos en métodos. Le aplicamos nuestra condición previamente declarada e indicamos a nuestra anotación que tendrá un atributo *databaseName* el cuál es obligatorio (si no lo queremos obligatorio bastaría con anotar un valor por defecto con *default*). Y ya tenemos nuestra anotación lista para consumir. Ahora, volvamos al código de configuración de base de datos:

```
@Slf4j
@Configuration
public class DatabaseConfig {

    @Bean
    @DatabaseOnConditional(databaseName = "postgresql")
    UserRepository userPostgreSQLRepository() {
        log.info(":: Load PostgreSQL database bean ::");
        return new UserPostgreSQLRepository();
    }

    @Bean
    @DatabaseOnConditional(databaseName = "mongodb")
    UserRepository userMongoDbRepository() {
        log.info(":: Load MongoDB database bean ::");
        return new UserMongoDbRepository();
    }

}

```

Anotamos ambos beans con el valor correspondiente que puede llevar la propiedad app.database del properties de la aplicación. Probamos ahora a arrancar la aplicación con el valor *app.database: postgresql* y vamos que ocurre en la consola de la aplicación:


```

  .   ____          _            __ _ _
 /\\ / ___'_ __ _ _(_)_ __  __ _ \ \ \ \
( ( )\___ | '_ | '_| | '_ \/ _` | \ \ \ \
 \\/  ___)| |_)| | | | | || (_| |  ) ) ) )
  '  |____| .__|_| |_|_| |_\__, | / / / /
 =========|_|==============|___/=/_/_/_/
 :: Spring Boot ::                (v3.1.1)

2023-07-09 15:45:58.923  INFO 11112 --- [           main] com.jfjara.XXXXXXXXXX                    : Starting XXXXXXXXXX using Java 17.0.5 
2023-07-09 15:45:59.787  INFO 11112 --- [           main] o.s.cloud.context.scope.GenericScope     : BeanFactory id=acc59933-9708-3305-9e76-bbdddc0c3f9d
2023-07-09 15:45:59.787  INFO 11112 --- [           main] in.n2w.web.config.DatabaseConfig         : :: Load PostgreSQL database bean ::
2023-07-09 15:46:01.752  INFO 11112 --- [           main] o.s.b.a.e.web.EndpointLinksResolver      : Exposing 1 endpoint(s) beneath base path '/actuator'
2023-07-09 15:46:03.038  INFO 11112 --- [           main] o.s.b.web.embedded.netty.NettyWebServer  : Netty started on port 63080
2023-07-09 15:46:03.239  INFO 11112 --- [           main] com.jfjara.XXXXXXXXXX                    : Started XXXXXXXXXX in 4.798 seconds (JVM running for 5.182)

```

Como se puede observar, se pinta el log por consola de carga del bean correspondiente **:: Load PostgreSQL database bean ::**

Ahora, probamos a cambiar la configuración de propiedades a app.database: mongodb y observamos la consola:

```

  .   ____          _            __ _ _
 /\\ / ___'_ __ _ _(_)_ __  __ _ \ \ \ \
( ( )\___ | '_ | '_| | '_ \/ _` | \ \ \ \
 \\/  ___)| |_)| | | | | || (_| |  ) ) ) )
  '  |____| .__|_| |_|_| |_\__, | / / / /
 =========|_|==============|___/=/_/_/_/
 :: Spring Boot ::                (v3.1.1)

2023-07-09 15:48:49.993  INFO 10764 --- [           main] com.jfjara.XXXXXXXXXX                    : Starting XXXXXXXXXX using Java 17.0.5 
2023-07-09 15:48:50.850  INFO 10764 --- [           main] o.s.cloud.context.scope.GenericScope     : BeanFactory id=2f0ab26e-d8c9-319a-a9aa-1999058df7ff
2023-07-09 15:48:50.850  INFO 10764 --- [           main] in.n2w.web.config.DatabaseConfig         : :: Load MongoDB database bean ::
2023-07-09 15:48:52.679  INFO 10764 --- [           main] o.s.b.a.e.web.EndpointLinksResolver      : Exposing 1 endpoint(s) beneath base path '/actuator'
2023-07-09 15:48:53.988  INFO 10764 --- [           main] o.s.b.web.embedded.netty.NettyWebServer  : Netty started on port 63080
2023-07-09 15:48:54.164  INFO 10764 --- [           main] com.jfjara.XXXXXXXXXX                    : Started XXXXXXXXXX in 4.692 seconds (JVM running for 5.038)

```

Si ahora nos fijamos, se está cargando el bean correspondiente a MongoDB **:: Load MongoDB database bean ::**

Esta técnica de carga de beans a contexto es muy útil para evitar tener que declarar ambos repositorios en nuestros casos de uso evitando las tediosas sentencias condicionales como *if-else if* o los *switch*. Aprovechamos el poder de la abstracción y el polimorfismo para resolver dicho problema. Recuerda, clean code y SOLID son tus fieles amigos en el desarrollo.

Deseo que este pequeño artículo os ayude en vuestros desarrollos. Si conocen otras formas de resolver este problema, estaré encantado de conocerlas, para ello tienen mi mail disponible para consultas **juanfranciscojara@gmail.com**

