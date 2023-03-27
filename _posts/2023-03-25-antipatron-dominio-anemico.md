---
layout: single
title: "Antipatrón\: Dominio Anémico"
excerpt: "Hace un tiempo me di cuenta de que los desarrolladores (sobre todo los de Java) caemos en una mala práctica a la hora de desarrollar nuestros proyectos sin darnos cuenta. Te invito a que mires cualquier proyecto que tengas y mires tus clases de dominio. ¿Ya? Bien, seguramente solo veas clases que no suelen ser más que un conjunto de datos que no hacen nada (los llamados DTO, POJOS, Beans…). Pues si amigos, según nuestro amigo Martin Fowler, eso es un antipatrón de diseño (curiosamente, el término POJO fue creado por la misma persona). Pero vayamos a la cuestión principal, ¿qué es un dominio anémico? Pues dicho rápidamente, son clases que no hacen nada, solo contender datos y estados, lo cual rompe con el paradigma de la programación orientada a objetos. ¿Y esto es malo?"
date: 2023-03-25
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

Hace un tiempo me di cuenta de que los desarrolladores (sobre todo los de Java) caemos en una mala práctica a la hora de desarrollar nuestros proyectos sin darnos cuenta. Te invito a que mires cualquier proyecto que tengas y mires tus clases de dominio. ¿Ya? Bien, seguramente solo veas clases que no suelen ser más que un conjunto de datos que no hacen nada (los llamados DTO, POJOS, Beans…). Pues si amigos, según nuestro amigo Martin Fowler, eso es un antipatrón de diseño (curiosamente, el término POJO fue creado por la misma persona). Pero vayamos a la cuestión principal, ¿qué es un dominio anémico? Pues dicho rápidamente, son clases que no hacen nada, solo contender datos y estados, lo cual rompe con el paradigma de la programación orientada a objetos. ¿Y esto es malo?

Hay mucho debate sobre ello, personalmente depende de la lógica de negocio y el tipo de arquitectura del servicio: podemos tener meros objetos que sirvan de contenedores de datos para el paso de información entre capas de la arquitectura (los llamados Data Transfer Objects o DTO). Esto es esencial en este tipo de arquitecturas (MVC, Hexagonal…) en donde estas clases delegan su funcionalidad o transformación en entidades externas (como pueden ser los repositorios, servicios, casos de uso, etc).

En muchos casos nos encontramos que los datos no se transforman ni tienen una lógica de negocio determinada (véase un clásico ciclo de CRUD donde lo que entra por un formulario acaba almacenado en un sistema de persistencia tal como llega del formulario), pero cuando los datos deben de ser procesados para realizar un cálculo, una transformación o cualquier proceso que pueda considerarse como parte de la lógica de negocio de la entidad, este debería de realizarse dentro de dicha entidad, no delegarlo en otra entidad externa. Veamos un sencillo y rápido ejemplo para comprenderlo mejor: Tenemos una entidad de factura (Invoice) la cuál el importe debe de aplicarse siempre un recargo del 0,2% de la cantidad de la misma. Vemos la entidad:

```

@Builder(toBuilder = true)
@Getter
public class InvoiceDto {

    private Long id;
    private Double amount;
    private String reference;
    private Long customerId;

}

```

Es algo muy habitual encontrarse un repositorio de persistencia donde se delega la lógica de negocio anteriormente descrita a dicho repositorio, lo cual es algo incorrecto ya que la lógica de negocio no debe de mezclarse con los servicios o elementos de infraestructura. También podemos encontrar esto mismo dentro de una entidad de mapeado con el mismo problema:

```

@Service
public class SaveInvoicePersistenceRepository implements SaveInvoiceRepository {

    @Autowired
    private PersistenceClient client;

    @Autowired
    private InvoiceEntityMapper mapper;
   

    public void save(final InvoiceDto invoice) {
       InvoiceEntity entity = mapper.toEntity(invoice);   //aqui esta realizando el cálculo del recargo
       client.save(entity);
    }

}

...

@Component
public class InvoiceEntityMapper {
   
   public InvoiceEntity toEntity(final InvoiceDto dto) {
      return InvoiceEntity.builder().amount(dto.getAmount())
                          .surcharge(dto.getAmount() * 0.21)
                          .reference(dto.getReference())
                          .customerId(dto.getCustomerId()).build();
   }

}

```

La modificación y corrección más natural sería:

```

@Builder(toBuilder = true)
@Getter
public class Invoice {

    private Long id;

    @Getter(AccessLevel.NONE)
    private Double amount;

    @Getter(AccessLevel.NONE)
    private Double surcharge;

    private String reference;
    private Long customerId;

    public Double getAmount() {
       return amount != null ? amount : 0.0;
    }

    public Double getSurcharge() {
       return getAmount() * 0.21;
    }

}

```

Esto sería una solución correcta donde la lógica de negocio queda encapsulada en la entidad responsable de dicha lógica y evitando así el llamado antipatrón de dominio anémico.