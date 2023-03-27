---
layout: single
title: "Contract testing con Spring Cloud Contract"
excerpt: ""
date: 2023-01-25
classes: wide
header:
  teaser: /assets/images/htb-writeup-delivery/delivery_logo.png
  teaser_home_page: true
  icon: /assets/images/hackthebox.webp
categories:
  - springboot
  - practise
  - testing
tags:  
  - contract
  - java
  - springboot
  - api
  - rest
  - testing
---

![](/assets/images/htb-writeup-delivery/delivery_logo.png)

¿A quién no le ha ocurrido alguna vez en su trabajo que desde el equipo front te indiquen que falta algún campo en el json de respuesta y tu juras que no falta nada y luego resulta que dicho campo no estaba definido en el contrato inicial? ¿O que cambias algo en el contrato de un productor y de repente los consumidores de dicho mensaje ya no funcionan?

Es un clásico entre los clásicos y de esta problemática nace una metodología de desarrollo donde se especifica previamente el contrato de las APIS: API First. Aún así, ¿quién nos garantiza que nos envíen en el cuerpo de una solicitu un json que no concuerda con el especificado en la firma? Absolutamente nadie, Y más hoy en día, donde los proyectos suelen estar estructurados en arquitecturas de microservicios donde varios equipos se encargan de desarrollar de manera independiente estos servicios que interactúan unos con otros.

## ¿Qué es?

Es una librería de testing encargada de comprobar los contratos entre servicios. Se basa en el uso de dos actores, productores (quien produce el contrato) y consumidores (el que lo consume). También disponemos de un elemento más, los denominados brokers, que no son más que repositorios donde se almacenan los contratos a testear.

El objetivo es crear un stub del servicio productor con la definición del contrato para que el consumidor interactúe con el desde una clase de testing. Veámoslo en un ejemplo.

## Práctica

Vamos a disponer de dos proyectos Maven donde simularemos un microservicio que consulta información de una granja, y un cliente que consume dicho servicio:

farm-contract-producer (API Rest)
farm-contract-consumer (Cliente que consume la API)
Contract-Producer

Especificación de Maven (pom.xml):

```

 	<dependencies>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-web</artifactId>
		</dependency>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-test</artifactId>
			<scope>test</scope>
		</dependency>
		<dependency>
			<groupId>org.springframework.cloud</groupId>
			<artifactId>spring-cloud-starter-contract-verifier</artifactId>
			<scope>test</scope>
		</dependency>
	</dependencies>

```
Incluimos las librerías de Spring Boot para configurar nuestros endpoints de la API así como spring-cloud-starter-contract-verifier que proporcionara la funcionalidad de establecer los contractos a testear como los stubs que se generarán.

```

<plugin>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-contract-maven-plugin</artifactId>
    <version>${org.springframework.cloud.version}</version>
    <extensions>true</extensions>
    <configuration>
        <testFramework>JUNIT5</testFramework>
        <baseClassForTests>com.jfjara.farm.contract.BaseClass</baseClassForTests>
    </configuration>
</plugin>

```

En la sección de plugins definimos uno donde indicamos donde se encontrará el controller que contendrá el contrato a testear, en este caso la clase que se encuentra en com.jfjara.farm.contract.BaseClass

```

@ExtendWith(SpringExtension.class)
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.MOCK)
@DirtiesContext
@AutoConfigureMessageVerifier
public abstract class BaseClass {

    @Autowired
    private FarmController farmController;

    @BeforeEach
    public void setup() {
        StandaloneMockMvcBuilder standaloneMockMvcBuilder = MockMvcBuilders.standaloneSetup(farmController);
        RestAssuredMockMvc.standaloneSetup(standaloneMockMvcBuilder);
    }

}

```

Definimos el test mediante el Mock Mvc y le asignamos el controller. Luego se inicia RestAssureMockMvc de modo independiente con los controller a testear (también se puede hacer por contexto).

## Definiendo la prueba de contrato y respuesta

A continuación vamos a la carpeta de recursos. Debemos configurar el contrato en formato yml:

```

name: Animal Types Contract
description: Get all type of farm animals
request:
  url: /farm/animals
  method: GET
response:
  status: 200
  headers:
    Content-Type: application/json
  bodyFromFile: responses/get_farm_animals.json

```

Se indica la url de la petición (que debe de ser la/s del controller a probar) como su método HTTP, la respuesta o respuestas a esperar con las cabeceras y el cuerpo del mensajes, que en nuestro caso lo definimos en un fichero aparte que contiene un json ubicado en responses/get_farm_animals.json (nuestro método devuelve un listado de String)

```

 [ 
    "COW",
    "CHICKEN",
    "PIG"
]

```

Procedemos a compilar e instalar en nuestro repositorio local (en un entorno productivo es recomendable la instalación en un repositorio servidor tipo Nexus):

```

mvn clean install

```

y obtenemos por consola el resultado de la compilación. Fíjense en la línea remarcada, indica el stub para la prueba de contrato que se ha generado e instalado en el repositorio.

```

[INFO] Installing C:\Users\juanf\Documents\GitHub\Spring-Cloud-Contract-Examples\farm\target\farm-0.0.1-SNAPSHOT-stubs.jar to C:\Users\juanf\.m2\repository\com\jfjara\farm\0.0.1-SNAPSHOT\farm-0.0.1-SNAPSHOT-stubs.jar
[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
[INFO] Total time:  15.370 s
[INFO] Finished at: 2023-01-25T10:47:25+01:00
[INFO] ------------------------------------------------------------------------

```

## Contract-Consumer

Una vez generado el productor del contrato, vamos a crear el consumidor. Son pasos muy sencillos. Veamos el pom.xml de este proyecto:

```

<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-contract-stub-runner</artifactId>
    <scope>test</scope>
</dependency>

```

Incluimos la libreria para realizar la prueba de contrato, que no es más que para la carga y consumo del stab que vamos a definir que consuma.

Y ahora le toca el turno a definir el test a realizar. Creamos una clase en:

```

@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.MOCK)
@AutoConfigureMockMvc
@AutoConfigureStubRunner(ids = {"com.jfjara:farm:+:stubs:8080"}, stubsMode = StubRunnerProperties.StubsMode.LOCAL)
public class ConsumerTest {

    @Autowired
    private MockMvc mockMvc;

    @Test
    void verify_get_heart() throws Exception {
        mockMvc.perform(MockMvcRequestBuilders.get("/animals")
                        .contentType(MediaType.APPLICATION_JSON))
                .andExpect(status().isOk())
                .andExpect(content().json("[\"COW\", \"CHICKEN\", \"PIG\"]"));
    }

}

```

Especial atención al AutoConfigureStubRunner: en el definimos donde estara nuestro stub previamente generado (se monta con el groupId y artifact) y que puerto tiene (en nuestro caso el 8080), así como indicarle que es un stub local. El test es como cualquier test realizado con MockMVC. La respuesta debe coincidir con la que nos devuelve el stub y que previamente hemos definido.

Ejecutamos el test y obtenemos en consola:

```

127.0.0.1 - GET /farm/animals

Accept: [application/json, application/*+json]
Host: [localhost:8080]
Connection: [keep-alive]
User-Agent: [Apache-HttpClient/4.5.13 (Java/17.0.5)]
Accept-Encoding: [gzip,deflate]



Matched response definition:
{
  "status" : 200,
  "body" : "[\"COW\",\"CHICKEN\",\"PIG\"]",
  "headers" : {
    "Content-Type" : "application/json"
  },
  "transformers" : [ "response-template", "spring-cloud-contract" ]
}

Response:
HTTP/1.1 200
Content-Type: [application/json]
Matched-Stub-Id: [b0251746-7548-4f57-a6d8-917cd462fdc3]

```

Se comprueba la respuesta que nos devuelve el stub con la que esperamos. Como vemos, es correcto el test. A continuación vamos a forzar un error para ver que obtenemos. Para ello, modificamos el json de respuesta de nuestro test indicando que esperamos una lista vacía:

```

@Test
void verify_get_heart() throws Exception {
    mockMvc.perform(MockMvcRequestBuilders.get("/animals")
                    .contentType(MediaType.APPLICATION_JSON))
            .andExpect(status().isOk())
            .andExpect(content().json("[]"));
}

```

Ejecutamos y observamos la respuesta en consola:

```

Matched response definition:
{
  "status" : 200,
  "body" : "[\"COW\",\"CHICKEN\",\"PIG\"]",
  "headers" : {
    "Content-Type" : "application/json"
  },
  "transformers" : [ "response-template", "spring-cloud-contract" ]
}

Response:
HTTP/1.1 200
Content-Type: [application/json]
Matched-Stub-Id: [b0251746-7548-4f57-a6d8-917cd462fdc3]



MockHttpServletRequest:
      HTTP Method = GET
      Request URI = /animals
       Parameters = {}
          Headers = [Content-Type:"application/json;charset=UTF-8"]
             Body = null
    Session Attrs = {}

Handler:
             Type = com.jfjara.farmcontractconsumer.ConsumerController
           Method = com.jfjara.farmcontractconsumer.ConsumerController#getFarmAnimals()

Async:
    Async started = false
     Async result = null

Resolved Exception:
             Type = null

ModelAndView:
        View name = null
             View = null
            Model = null

FlashMap:
       Attributes = null

MockHttpServletResponse:
           Status = 200
    Error message = null
          Headers = [Content-Type:"application/json"]
     Content type = application/json
             Body = ["COW","CHICKEN","PIG"]
    Forwarded URL = null
   Redirected URL = null
          Cookies = []

java.lang.AssertionError: []: Expected 0 values but got 3

```

Vemos que en cuanto estructura es lo que espera, sin embargo falla en los datos, por lo que no pasa el test. Hacemos otra prueba esta vez cambiando la estructura que esperamos en la respuesta:

```

@Test
void verify_get_heart() throws Exception {
    mockMvc.perform(MockMvcRequestBuilders.get("/animals")
                    .contentType(MediaType.APPLICATION_JSON))
            .andExpect(status().isOk())
            .andExpect(content().json("{\"name\": \"CHICKEN\"}"));
}

```

Ejecutamos y vemos la consola:

```

Matched response definition:
{
  "status" : 200,
  "body" : "[\"COW\",\"CHICKEN\",\"PIG\"]",
  "headers" : {
    "Content-Type" : "application/json"
  },
  "transformers" : [ "response-template", "spring-cloud-contract" ]
}

Response:
HTTP/1.1 200
Content-Type: [application/json]
Matched-Stub-Id: [b0251746-7548-4f57-a6d8-917cd462fdc3]



MockHttpServletRequest:
      HTTP Method = GET
      Request URI = /animals
       Parameters = {}
          Headers = [Content-Type:"application/json;charset=UTF-8"]
             Body = null
    Session Attrs = {}

Handler:
             Type = com.jfjara.farmcontractconsumer.ConsumerController
           Method = com.jfjara.farmcontractconsumer.ConsumerController#getFarmAnimals()

Async:
    Async started = false
     Async result = null

Resolved Exception:
             Type = null

ModelAndView:
        View name = null
             View = null
            Model = null

FlashMap:
       Attributes = null

MockHttpServletResponse:
           Status = 200
    Error message = null
          Headers = [Content-Type:"application/json"]
     Content type = application/json
             Body = ["COW","CHICKEN","PIG"]
    Forwarded URL = null
   Redirected URL = null
          Cookies = []

java.lang.AssertionError: 
Expected: a JSON object
     got: a JSON array

```

En este caso nos indica que esperamos un objeto json (que solo contiene un atributo name) pero que obtenemos del stub un array, por lo que no cumple el contrato.

Y como siempre, aquí tienen un ejemplo completo en mi repositorio de GitHub. https://github.com/jfjara/Spring-Cloud-Contract-Examples