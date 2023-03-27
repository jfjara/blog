---
layout: single
title: "Spring Cloud Streams con Kafka. Un ejemplo básico para principiantes"
excerpt: "Spring Cloud Streams es un framework que tiene como propósito de abstraer el desarrollo de la comunicación entre microservicios, basada en eventos (EDA) y para el procesado de datos en tiempo real. La abstracción se basa en desacoplar los productores y consumidores de modo que se puede utilizar varios tipos de infraestructuras de mensajería (Apache Kafka, RabbitMQ, AWS SQS…) e intercambiarlas de manera transparente a la lógica de negocio, solo cambiando la implementación del binder de mensajería por otro. Como podrán observar, resulta muy fácil de configurar y de manera muy rápida, así que vamos a ello. En este ejemplo implementaremos Spring Cloud Streams junto con Apache Kafka y nos apoyaremos de otro módulo del ecosistema de Spring llamada Spring Cloud Functions, para dar soporte a la programación funcional que usaremos como entrada y salida de los mensajes."
date: 2023-02-19
classes: wide
header:
  teaser: /assets/images/htb-writeup-delivery/delivery_logo.png
  teaser_home_page: true
  icon: /assets/images/hackthebox.webp
categories:
  - springboot
  - practise
  - kafka
tags:  
  - springboot
  - cloud
  - kafka
  - java
  - POO
---

![](/assets/images/htb-writeup-delivery/delivery_logo.png)

Spring Cloud Streams es un framework que tiene como propósito de abstraer el desarrollo de la comunicación entre microservicios, basada en eventos (EDA) y para el procesado de datos en tiempo real. La abstracción se basa en desacoplar los productores y consumidores de modo que se puede utilizar varios tipos de infraestructuras de mensajería (Apache Kafka, RabbitMQ, AWS SQS…) e intercambiarlas de manera transparente a la lógica de negocio, solo cambiando la implementación del binder de mensajería por otro. Como podrán observar, resulta muy fácil de configurar y de manera muy rápida, así que vamos a ello. En este ejemplo implementaremos Spring Cloud Streams junto con Apache Kafka y nos apoyaremos de otro módulo del ecosistema de Spring llamada Spring Cloud Functions, para dar soporte a la programación funcional que usaremos como entrada y salida de los mensajes. Comencemos.

## Productor

Crearemos un microservicio encargado de producir mensajes que se enviarán a un tópico de Kafka. Para ello, definiremos una tarea que se ejecute mediante una expresión Cron y lance un mensaje cada cierto tiempo. Pero vamos primero con la configuración del productor:

```

server:
  port: 8080
spring:
  cloud:
    stream:
      bindings:
        invoice-out-0:
          destination: invoice.topic
          producer:
            useNativeEncoding: true
      kafka:
        bindings:
          invoice-out-0:
            producer:
              configuration:
                value:
                  serializer: com.jfjara.kafkaproducer.invoice.infrastructure.kafka.serializer.InvoiceSerializer
        binder:
          brokers: localhost:9092

```

La configuración del productor es bien sencilla: en el apartado bindings de Spring Cloud Streams declaramos, por así decirlo, los canales correspondientes de los productores y consumidores. Fijémonos en el definido en nuestra configuración (invoice-out-0). Tiene un nombre peculiar y es debido a una razón de peso: Spring Cloud Functions obliga a crear los bindings con un nombre que debe de cumplir el siguiente patron: \[nombre\_del\_function\]\-\[in/out\]\-\[índice\]. Traduciendo:

- Nombre del function: debe de coincidir con el nombre de function declarado. En nuestro caso, al usar para el envío de mensajes bajo demanda StreamBridge no es necesario que definamos ninguno para el productor. Si hubiésemos optado por utilizar programación funcional para el productor seria obligatorio que nuestro function para tal caso se llamase spring.cloud.function.definition: invoice (ya que nuestro binding se llama invoice-out-0). Lo veremos más claramente en el caso del consumidor.
- in/out: Este punto es obvio. En caso de productores, out. Para consumidores, in.
- Índice: Un simple índice para enumerar la definición de bindings.

Continuando con la configuración, el siguiente parámetro es destination, el cual indicamos el topic de Kafka al cual vamos a mandar los mensajes. El siguiente conjunto de valores se engloba en producer, donde se indican configuraciones del productor. En nuestro caso solo incluimos useNativeEncoding habilitado para poder establecer un serializador de mensajes propio.

El siguiente conjunto de configuraciones hacen referencia al broker de mensajería con el que vamos a trabajar. Se indica la configuración del binding (indicando su nombre), se le dice que es un productor y de igual modo que la configuración anterior se establecen sus configuraciones. Sólo indicamos un serializador propio para el envío de eventos. En el apartado binder también se incluyen configuraciones de la abstracción del broker, en este caso indicamos dónde esta Kafka (host + puerto). Hay muchísimos parámetros de configuración, aquí para el ejemplo solo usaremos lo básico para una comunicación. Os dejo enlace a la documentación del módulo.

Vayamos a la parte de código. Realizar el envío de mensaje es extremadamente sencillo. Desarrollamos un repositorio encargado de ello (siguiendo estrictamente los principios de Arquitectura Hexagonal) donde llamamos al comunicador StreamBridge para realizar el envío de mensajes bajo demanda:

```

@Service
public class SendInvoiceKafkaRepository implements SendInvoiceRepository {

    @Autowired
    private StreamBridge streamBridge;

    @Autowired
    private InvoiceMapper mapper;

    @Override
    public void send(final Invoice invoice) {
        final InvoiceEvent event = mapper.toEvent(invoice);
        streamBridge.send("invoice-out-0", event);
    }
    
}

```

Demasiado intuitivo. A la función send de StreamBridge se le indica el nombre del binding que usaremos (recuerden que debe de ser un productor) y el mensaje a enviar. Y listo. Bueno, o casi, nos falta el serializador:

```

public class InvoiceSerializer implements Serializer<InvoiceEvent> {
    
    private final Logger logger = LoggerFactory.getLogger(InvoiceSerializer.class);

    private final ObjectMapper objectMapper = new ObjectMapper();
    @Override
    public byte[] serialize(String topic, InvoiceEvent data) {
        try {
            return objectMapper.writeValueAsBytes(data);
        } catch (JsonProcessingException exception) {
            logger.info("Error serializing invoice event: {}", exception.getMessage());
            throw new SerializationException(exception);
        }
    }
    
}

```

Ahora si, tenemos todas las piezas para realizar envío de mensajes a un broker mediante la abstracción que nos brinda Spring Cloud Stream. Vayamos ahora con el consumidor.

## Consumidor

Para el consumidor aprovechamos las bondades que nos aporta Spring Cloud Functions (como previamente se ha comentado) y lo emplearemos como punto de entrada a nuestra aplicación:

```

server:
  port: 8081

spring:
  cloud:
    function:
      definition: invoice;
    stream:
      bindings:
        invoice-in-0:
          destination: invoice.topic
          consumer:
            use-native-decoding: true
      kafka:
        bindings:
          invoice-in-0:
            consumer:
              configuration:
                value:
                  deserializer: com.jfjara.kafkaproducer.invoice.infrastructure.kafka.serializer.InvoiceDeserializer
        binder:
          brokers: localhost:9092

```

La configuración en nuestro ejemplo es muy similar a la del productor, solo cambian los nombres de los bindings (recuerden ese patrón previamente descrito), indicar un deserializador de los eventos y el use-native-decoding para habilitar dicho deserializador. Lo que si se aprecia es un nuevo actor, y es que como hemos comentado usaremos para el consumer Spring Cloud Functions, por lo que debemos de definir el correspondiente como punto de entrada de nuestros eventos: *spring.cloud.function.definition: invoice.*

Una vez configurado, solo nos queda definir el componente que se encarga de recibir los mensajes mediante un bean que hace referencia a la función de consumidor. No olviden que debe de llamarse igual que la función definida en la configuración (invoice en nuestro ejemplo):

```

@Component
public class InvoiceReceiver {

    @Autowired
    private ProcessInvoice processInvoice;

    @Autowired
    private InvoiceMapper mapper;

    @Bean
    public Consumer<InvoiceEvent> invoice() {
        return event -> {
            final Invoice invoice = mapper.toModel(event);
            processInvoice.execute(invoice);
        };
    }

}

```

¡No me puedo creer que sea tan sencillo! Pues así es joven Padawan de los bits. Aprovechamos la abstracción que nos brinda Spring Cloud Streams y Spring Cloud Functions para estar una capa por encima de la libreria del broker, en este caso Apache Kafka. Ahora tocaría incluir políticas y configuraciones que creamos oportunas para la comunicación (como el manejo de offsets, política de confirmación de mensajes como exactly_once, etc). Recuerden echar un vistazo a la documentación del módulo.

Este es nuestro deserializador de los eventos:

```

public class InvoiceDeserializer implements Deserializer<InvoiceEvent> {

    private final Logger logger = LoggerFactory.getLogger(InvoiceDeserializer.class);
    
    private final ObjectMapper objectMapper = new ObjectMapper();

    @Override
    public InvoiceEvent deserialize(String topic, byte[] data) {
        try {
            return objectMapper.readValue(new String(data), InvoiceEvent.class);
        } catch (IOException e) {
            logger.error("Error deserializing invoice event");
            throw new SerializationException(e);
        }
    }

}

```

Y ahora, hagamos la magia. Arrancamos el proyecto del consumidor el del productor. Vemos los logs del productor:

```

INFO 6848 --- [   scheduling-1] c.j.k.i.a.usecases.SendInvoiceUseCase    : Execute send invoice usecase. Send invoice Invoice(id=5346144739450824144, amount=1.574836350385667, reference=fb94ca97-c1ac-45b0-879b-53bf2a009ec2, customerId=988390996874898053)
INFO 6848 --- [   scheduling-1] c.j.k.i.i.spring.job.InvoiceJob          : End invoice job. Invoice has been sent
INFO 6848 --- [   scheduling-1] c.j.k.i.a.usecases.SendInvoiceUseCase    : Execute send invoice usecase. Send invoice Invoice(id=5346144739450824144, amount=1.574836350385667, reference=6deee9e9-fffe-4e63-8cee-71d0de834756, customerId=988390996874898053)
INFO 6848 --- [   scheduling-1] c.j.k.i.i.spring.job.InvoiceJob          : End invoice job. Invoice has been sent
INFO 6848 --- [   scheduling-1] c.j.k.i.a.usecases.SendInvoiceUseCase    : Execute send invoice usecase. Send invoice Invoice(id=5346144739450824144, amount=1.574836350385667, reference=7317cbf1-b95b-4231-be31-85fef0615b49, customerId=988390996874898053)
INFO 6848 --- [   scheduling-1] c.j.k.i.i.spring.job.InvoiceJob          : End invoice job. Invoice has been sent

```

Hemos enviado tres eventos al broker. Veamos el log del consumidor:

```

INFO 20356 --- [container-0-C-1] c.j.k.i.a.u.ProcessInvoiceUseCase        : Execute process invoice use case. Hi invoice Invoice(id=5346144739450824144, amount=1.574836350385667, reference=fb94ca97-c1ac-45b0-879b-53bf2a009ec2, customerId=988390996874898053)
INFO 20356 --- [container-0-C-1] c.j.k.i.a.u.ProcessInvoiceUseCase        : Execute process invoice use case. Hi invoice Invoice(id=5346144739450824144, amount=1.574836350385667, reference=6deee9e9-fffe-4e63-8cee-71d0de834756, customerId=988390996874898053)
INFO 20356 --- [container-0-C-1] c.j.k.i.a.u.ProcessInvoiceUseCase        : Execute process invoice use case. Hi invoice Invoice(id=5346144739450824144, amount=1.574836350385667, reference=7317cbf1-b95b-4231-be31-85fef0615b49, customerId=988390996874898053)

```

Se reciben los tres eventos. Eureka! Funciona! Demasiado sencillo para ser cierto. Como con tan poca configuración y tan poco código se puede lograr este tipo de comunicación entre microservicios gracias a las abstracciones de Spring Cloud Streams y Spring Cloud Functions. Y como siempre, os dejo el link a mi repositorio de GitHub donde esta alojado el ejemplo de este artículo. En el usamos la siguiente tecnología:

- Java 17
- Spring Boot 2.7.8
- Spring Cloud Streams
- Spring Cloud Functions
- Lombok
- Mapstruct
- Binder Kafka

**Si encuentran útil el articulo no olviden de suscribirse al blog para mantenerse informado de los nuevos artículos que se vayan publicando. Muchas gracias.**