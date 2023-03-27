---
layout: single
title: "Spring Cloud Streams, Apache Kafka y reactividad con Webflux"
excerpt: "Retomando el artículo anterior (que si no lo has visto, te invito a echarle un vistazo) vemos que tenemos la posibilidad de implementar productores y consumidores bajo el paradigma de desarrollo reactivo. En el presente artículo no vamos a entrar en detalles sobre dicho paradigma, vamos a implementar directamente como los productores y consumidores puede intercambiar mensajes con dicho paradigma, eso lo dejamos para una futura entrada del blog. Pues bien, vamos con ello"
date: 2023-02-22
classes: wide
header:
  teaser: /assets/images/htb-writeup-delivery/delivery_logo.png
  teaser_home_page: true
  icon: /assets/images/hackthebox.webp
categories:
  - springboot
  - practise
  - kafka
  - reactive
tags:  
  - springboot
  - java
  - POO
  - kafka
  - stream
  - reactive
---

![](/assets/images/htb-writeup-delivery/delivery_logo.png)

Retomando el artículo anterior (que si no lo has visto, te invito a echarle un vistazo) vemos que tenemos la posibilidad de implementar productores y consumidores bajo el paradigma de desarrollo reactivo. En el presente artículo no vamos a entrar en detalles sobre dicho paradigma, vamos a implementar directamente como los productores y consumidores puede intercambiar mensajes con dicho paradigma, eso lo dejamos para una futura entrada del blog. Pues bien, vamos con ello.

## Productor

```

spring:
  cloud:
    function:
      definition: invoice;
    stream:
      bindings:
        invoice-out-0:
          destination: invoice.topic
      kafka:
        bindings:
          invoice-out-0:
            producer:
              configuration:
                value:
                  serializer: org.springframework.kafka.support.serializer.JsonSerializer
        binder:
          brokers: localhost:9092

```

Prácticamente es una configuración similar a la del productor del artículo anterior. Solo vamos a cambiar el serializador por el de Json. Y en esta ocasión usaremos Spring Cloud Functions para la producción de mensajes. Vamos a crear nuestro repositorio de envío de mensajes:

```

@Service
public class SendInvoiceKafkaRepository implements SendInvoiceRepository {

    @Autowired
    private InvoiceMapper mapper;
    private Sinks.Many<InvoiceEvent> communicator = Sinks.many().unicast().onBackpressureBuffer();

    public Sinks.Many getCommunicator() {
        return communicator;
    }

    public void send(final Invoice invoice) {
        communicator.emitNext(mapper.toEvent(invoice), Sinks.EmitFailureHandler.FAIL_FAST);
    }

}

```

Ahora nos detenemos en los detalles: declaramos la variable «communicator», la cual no es más que un flujo de datos al cual se suscribirá el productor. Después, se realiza la serialización del mensaje y publicación de estos mediante la función emitNext(). Y ahora vemos la siguiente pieza del puzzle:

```

@Configuration
public class Producer {
    @Autowired
    SendInvoiceKafkaRepository producerService;

    @Bean
    public Supplier<Flux<InvoiceEvent>> invoice() {
        return () -> producerService.getCommunicator().asFlux();
    }

}

```

En esta clase de configuración creamos un bean que sera un Supplier de Spring Cloud Functions. Recuerden, el nombre del bean debe de ser exactamente igual a la definición del function en el fichero de configuración (en nuestro caso, invoice). En el referenciamos la clase anterior descrita con un Autowired y dentro del supplier devolvemos un Flux de InvoiceEvent, que si os fijais bien de donde sale dicho Flux, es de la llamada al comunicador anteriormente descrita. Ahí tenemos la pieza que faltaba por encajar. Ahora, desde nuestro job comenzamos a lanzar eventos a diestro y siniestro:

```

@Component
public class InvoiceJob {

    private final Logger logger = LoggerFactory.getLogger(InvoiceJob.class);

    @Autowired
    SendInvoice sendInvoice;

    @Scheduled(cron = "*/5 * * * * *")
    public void sendInvoice() {
        logger.info("#### Start invoice job. Send a Invoice {}", invoice);
        Flux<Invoice> data = Flux.just(invoiceEvent1, invoiceEvent2, ...);
        data.subscribe(d -> sendInvoice.execute(d));
        logger.info("##### End invoice job. ######");
    }

```

Establecemos un Flux de datos y lo suscribimos. Cuando tengamos la señal de recibo ejecutamos por cada elemento un caso de uso que se encarga del envío:

```

public class SendInvoiceUseCase implements SendInvoice {

    private final SendInvoiceRepository sendInvoiceRepository;

    public SendInvoiceUseCase(final SendInvoiceRepository sendInvoiceRepository) {
        this.sendInvoiceRepository = sendInvoiceRepository;
    }

    @Override
    public void execute(final Invoice invoice) {
        sendInvoiceRepository.send(invoice);
    }
}

```

Y con esto cerramos el círculo. Arrancamos y probamos. Aquí tenemos el log del productor con los eventos publicados en el topic de Kafka:

```

c.j.k.i.a.usecases.SendInvoiceUseCase    : #### Start invoice job. Send a Invoice Invoice(id=991669146557400102, amount=1.0576967615293782, reference=e6fb672f-6b25-49be-a710-bc258d0f315c, customerId=1903839370086246983)
c.j.k.i.a.usecases.SendInvoiceUseCase    : #### Start invoice job. Send a Invoice Invoice(id=3726587132569629103, amount=1.4602125107378479, reference=e2d3fd55-fe04-487c-9f3b-e712042f1cb4, customerId=5454320455135883580)
c.j.k.i.a.usecases.SendInvoiceUseCase    : #### Start invoice job. Send a Invoice Invoice(id=2332587141353668490, amount=1.387607036167386, reference=3410c8f4-3873-4621-82f0-1ad7d00945c3, customerId=4899250297148527777)
c.j.k.i.a.usecases.SendInvoiceUseCase    : #### Start invoice job. Send a Invoice Invoice(id=3890847605298212830, amount=1.5873597815595062, reference=6ec25f0b-8104-41a7-a6c6-c9c3710f7bfa, customerId=5426517152519098910)
c.j.k.i.i.spring.job.InvoiceJob          : ##### End invoice job. ######
c.j.k.i.a.usecases.SendInvoiceUseCase    : #### Start invoice job. Send a Invoice Invoice(id=1180768803989629, amount=1.6857004020948936, reference=a0f45023-a633-4e87-b9b9-d8db56886fc0, customerId=7978686853068181965)
c.j.k.i.a.usecases.SendInvoiceUseCase    : #### Start invoice job. Send a Invoice Invoice(id=8605538571518124979, amount=1.9401517156353945, reference=18fb9a71-52f5-4a4c-836a-92e1db262718, customerId=567428594202876049)
c.j.k.i.a.usecases.SendInvoiceUseCase    : #### Start invoice job. Send a Invoice Invoice(id=5384032416006514187, amount=1.7984872544094452, reference=56f5b7b0-4519-44c9-8529-ef9f86945002, customerId=5613768594884872552)
c.j.k.i.a.usecases.SendInvoiceUseCase    : #### Start invoice job. Send a Invoice Invoice(id=4596150706779810898, amount=1.1882133535527004, reference=0a5ef682-810c-4aa8-9156-8528787dd4e3, customerId=2042262147192677443)
c.j.k.i.i.spring.job.InvoiceJob          : ##### End invoice job. ######
Consumidor

```

Vamos primero con la configuración del módulo de Spring Cloud Streams:

```

spring:
  cloud:
    function:
      definition: invoice;
    stream:
      bindings:
        invoice-in-0:
          destination: invoice.topic
      kafka:
        bindings:
          invoice-in-0:
            consumer:
              configuration:
                max.poll.records: 10
                spring.json.trusted.packages: "*"
                value:
                  deserializer: org.springframework.kafka.support.serializer.JsonDeserializer
        binder:
          brokers: localhost:9092

```

Ya explicamos en un artículo anterior casi todos los parámetros de configuración aquí presente. Dejo enlace al artículo en cuestión y a la documentación oficial, y pasemos a comentar las propiedades que difieren, que en nuestro caso son dos:

- max.poll.records: Indica el número máximo de registros que se retornan en una única llama a poll(). Indicamos un máximo de 10 registros.
- spring.json.trusted.packages: Define un array de paquetes que se permiten para la deserialización. En nuestro caso vamos a poner todos (‘*’).

Sólo queda implementar la consumidor «invoice» con Spring Cloud Function. Aquí no dista mucho del ejemplo anterior, solo que en vez de recibir un evento, recibimos un Flux de eventos:

```

@Component
public class InvoiceReceiver {

    @Autowired
    private ProcessInvoice processInvoice;

    @Autowired
    private InvoiceMapper mapper;

    @Bean
    public Consumer<Flux<InvoiceEvent>> invoice() {
        return event -> {
            event.concatMap(i -> {
                final Invoice invoice = mapper.toModel(i);
                processInvoice.execute(invoice);
                return Mono.empty();
            }).subscribe();
        };
    }

}

```

Destacar que devolvemos un Mono\<Void\> en cada uno de los mapeos de los eventos. Arrancamos el servicio y aquí tenemos la lectura de los eventos publicados anteriormente:

```

c.j.k.i.a.u.ProcessInvoiceUseCase        : Execute process invoice use case. Hi invoice Invoice(id=991669146557400102, amount=1.0576967615293782, reference=e6fb672f-6b25-49be-a710-bc258d0f315c, customerId=1903839370086246983)
c.j.k.i.a.u.ProcessInvoiceUseCase        : Execute process invoice use case. Hi invoice Invoice(id=3726587132569629103, amount=1.4602125107378479, reference=e2d3fd55-fe04-487c-9f3b-e712042f1cb4, customerId=5454320455135883580)
c.j.k.i.a.u.ProcessInvoiceUseCase        : Execute process invoice use case. Hi invoice Invoice(id=2332587141353668490, amount=1.387607036167386, reference=3410c8f4-3873-4621-82f0-1ad7d00945c3, customerId=4899250297148527777)
c.j.k.i.a.u.ProcessInvoiceUseCase        : Execute process invoice use case. Hi invoice Invoice(id=3890847605298212830, amount=1.5873597815595062, reference=6ec25f0b-8104-41a7-a6c6-c9c3710f7bfa, customerId=5426517152519098910)
c.j.k.i.a.u.ProcessInvoiceUseCase        : Execute process invoice use case. Hi invoice Invoice(id=1180768803989629, amount=1.6857004020948936, reference=a0f45023-a633-4e87-b9b9-d8db56886fc0, customerId=7978686853068181965)
c.j.k.i.a.u.ProcessInvoiceUseCase        : Execute process invoice use case. Hi invoice Invoice(id=8605538571518124979, amount=1.9401517156353945, reference=18fb9a71-52f5-4a4c-836a-92e1db262718, customerId=567428594202876049)
c.j.k.i.a.u.ProcessInvoiceUseCase        : Execute process invoice use case. Hi invoice Invoice(id=5384032416006514187, amount=1.7984872544094452, reference=56f5b7b0-4519-44c9-8529-ef9f86945002, customerId=5613768594884872552)
c.j.k.i.a.u.ProcessInvoiceUseCase        : Execute process invoice use case. Hi invoice Invoice(id=4596150706779810898, amount=1.1882133535527004, reference=0a5ef682-810c-4aa8-9156-8528787dd4e3, customerId=2042262147192677443)

```

Como conclusión podemos afirmar que adaptar el ejemplo anterior para convertirlo en reactivo es bastante sencillo. Aquí dejamos el link de [GitHub](https://github.com/jfjara/spring-cloud-streams-kafka) al proyecto de ejemplo.

Stack utilizado:

- Java 17
- Spring Boot 2.7.8
- Spring Cloud Streams
- Spring Cloud Functions
- Lombok
- Mapstruct
- Binder Kafka

**Si encuentran útil el articulo no olviden de suscribirse al blog para mantenerse informado de los nuevos artículos que se vayan publicando. Muchas gracias.**



