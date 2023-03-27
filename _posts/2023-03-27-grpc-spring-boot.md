---
layout: single
title: gRPC - Caso práctico con Spring Boot
excerpt: "A pesar de las enormes ventajas que proporciona este sistema de llamadas pocas son las empresas que apuestan por su uso frente a una ya muy extendida y casi obligatoria arquitectura API REST. Pero, ¿por qué se debe esto?"
date: 2023-01-21
classes: wide
header:
  teaser: /assets/images/htb-writeup-delivery/delivery_logo.png
  teaser_home_page: true
  icon: /assets/images/hackthebox.webp
categories:
  - springboot
  - practise
tags:  
  - grpc
  - java
  - springboot
  - api
  - rest
---

![](/assets/images/htb-writeup-delivery/delivery_logo.png)

A pesar de las enormes ventajas que proporciona este sistema de llamadas pocas son las empresas que apuestan por su uso frente a una ya muy extendida y casi obligatoria arquitectura API REST. Pero, ¿por qué se debe esto?

## Un poco de historia

gRPC nace en 2015 de la necesidad de mejorar la comunicación entre servicios. Fue creada por `Google` y actualmente distribuida y mantenida por `Cloud Native Computation Foundation` y es de código abierto. Actualmente empresas gigantes como `Netflix` lo utilizan debido a los problemas de rendimiento y alta latencia que tenían ante el incremento de usuarios y demanda.

## ¿Qué es?

Es un sistema de llamadas a procedimientos remotos que actúa a nivel de procesos (al igual que su predecesor, `RPC`).

## ¿Cómo funciona?

Sirve para realizar comunicaciones entre cliente y servicios de un servidor cómo si se realizase una llamada local. Dichos servicios y mensajes de intercambio se definen mediante leguaje `Protobuf` (Protocol Buffers) que es un lenguaje extensible y neutral (los mensajes viajan serializados y en formato binario). Funciona sobre protocolo `HTTP/2` (más rápido que otros servicios de comunicación).

## Ventajas

- Baja latencia.
- Los mensajes viajan en formato binario, por lo que tiene menor tamaño frente a otros procedimientos de llamada como por ejemplo API REST.
- Streaming de datos bidireccional entre cliente-servidor, streaming entre servidor-cliente y cliente-servidor.
- Generación automática de clases servicios y mensajes.
- Permite cancelación de llamadas.

## Inconvenientes

- Actualmente los navegadores no soportan HTTP/2 por lo que no es posible usar un cliente sobre gRPC.
- Al usar mensajes serializados en binario no es posible leer de forma natural dichos mensajes.

## Ejemplo práctico

En esta entrada veremos como podemos usar llamadas gRPC entre un cliente y un servidor. Para mostrar la información de manera amigable definimos un contrato REST en el cliente.

Para este caso creamos 3 proyectos:

- Interfaz gRPC
- Cliente
- Servidor

### Interfaz

La interfaz define los mensajes y servicios gRPC que se van utilizar en clientes y servidores.

Configuración Maven (pom.xml)

```

     <dependencies>
        <dependency>
            <groupId>io.grpc</groupId>
            <artifactId>grpc-stub</artifactId>
            <version>${grpc.version}</version>
        </dependency>
        <dependency>
            <groupId>io.grpc</groupId>
            <artifactId>grpc-protobuf</artifactId>
            <version>${grpc.version}</version>
        </dependency>
        <dependency>
            <groupId>javax.annotation</groupId>
            <artifactId>javax.annotation-api</artifactId>
            <version>${javax.annotation.version}</version>
        </dependency>
    </dependencies>

```

Incluimos librerías de `grpc-stub`, que nos proporciona los stubs necesarios para la comunicación, y las de `protobuf` para el manejo de dicho lenguaje en el proceso de definir mensajes y servicios. La librería de anotaciones es necesaria para las anotaciones de las clases autogeneradas.

```

    <build>
        <extensions>
            <extension>
                <groupId>kr.motd.maven</groupId>
                <artifactId>os-maven-plugin</artifactId>
                <version>${kr.motd.maven.version}</version>
            </extension>
        </extensions>

        <plugins>
            <plugin>
                <groupId>org.xolstice.maven.plugins</groupId>
                <artifactId>protobuf-maven-plugin</artifactId>
                <version>${protobuf-plugin.version}</version>
                <configuration>
                    <protocArtifact>com.google.protobuf:protoc:${protobuf.version}:exe:${os.detected.classifier}</protocArtifact>
                    <pluginId>grpc-java</pluginId>
                    <pluginArtifact>io.grpc:protoc-gen-grpc-java:${grpc.version}:exe:${os.detected.classifier}</pluginArtifact>
                </configuration>
                <executions>
                    <execution>
                        <goals>
                            <goal>compile</goal>
                            <goal>compile-custom</goal>
                        </goals>
                    </execution>
                </executions>
            </plugin>
        </plugins>
    </build>
	
```

Definimos la extensión os-maven-plugin para extraer información de nuestro sistema necesaria para la generación de código. El plugin protobuf-maven-plugin es el encargado de leer los ficheros protobuf y generar clases de servicios y mensajes.

Sólo nos queda establecer servicios y mensajes en el fichero protobuf el cual lo ubicamos `src/main/proto`

```

syntax = "proto3";
option java_multiple_files = true;
package com.jfjara.grpc.infraestructure.proto;

message MovieShowtimesResponse {
  message Movie {
    string id = 1;
    string title = 2;
  }
  repeated Movie movies = 1;
}

message MovieRequest {
  string id = 1;
}

message SeatResponse {
    sfixed32 number = 1;
}

message EmptyMessage { }

service CinemaService {
  rpc getMovies(EmptyMessage) returns (MovieShowtimesResponse);
  rpc bookSeat(MovieRequest) returns (SeatResponse);
}

```

Como podemos observar e intuir, los message representan las clases de los objetos que viajarán serializados en la comunicación, mientras que los service definen el contrato de los servicios.

Para profundizar en más detalles sobre los tipos, elementos y demás, consultar la documentación del protocolo en la web de desarrollo de Google, donde todo se explica bastante detallado y fácil.

Una vez hecho esto, procedemos a compilar el proyecto e instalarlo en nuestro repositorio:

```

mvn clean install

```

Se deben de generar todas las clases necesarias para la comunicación dentro de nuestra carpeta `target/generated-sources/protobuf/`

### Servidor

Usaremos:

- Java 17
- Spring Boot 2.7.7
- grpc-server-spring-boot-starter
- Librería de interfaz gRPC previamente generada
- Lombok
- MapStruct
- Maven

Nuestras dependencias Maven (pom.xml):

```

    <dependencies>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-test</artifactId>
		</dependency>
		<dependency>
			<groupId>net.devh</groupId>
			<artifactId>grpc-server-spring-boot-starter</artifactId>
			<version>${net.devh.grpc.version}</version>
		</dependency>
		<dependency>
			<groupId>com.jfjara</groupId>
			<artifactId>grpc-interface</artifactId>
			<version>${com.jfjara.grpc-interfaces}</version>
		</dependency>
		<dependency>
			<groupId>org.projectlombok</groupId>
			<artifactId>lombok</artifactId>
			<optional>true</optional>
			<version>${org.projectlombok.version}</version>
		</dependency>
		<dependency>
			<groupId>org.mapstruct</groupId>
			<artifactId>mapstruct</artifactId>
			<version>${org.mapstruct.version}</version>
		</dependency>
	</dependencies>

```

La librería `grpc-server-spring-boot-starter` nos proporciona la funcionalidad necesaria para declarar los servicios gRPC de nuestro servidor y, al ser una librería integrada en el ecosistema Spring estos se inyectan en el IoC Container, lo que nos facilita mucho el trabajo de usar servicios.

A continuación vemos la implementación de un servicio gRPC:

```

@GrpcService
public class CinemaServiceImpl extends CinemaServiceGrpc.CinemaServiceImplBase {

    @Autowired
    private GetCurrentMoviesOnShowtimeUseCase getCurrentMoviesOnShowtimeUseCase;

...


```

La anotación `@GrpcService` hereda de otras anotaciones de Spring y extiende del servicio declarado en la interfaz gRPC que definimos en el paso anterior.
	
```

@Override
public void getMovies(EmptyMessage request, StreamObserver<MovieShowtimesResponse> responseObserver) {
    var movies = getCurrentMoviesOnShowtimeUseCase.execute();
    var moviesDto = movieMapper.toDtos(movies);
    var response = MovieShowtimesResponse.newBuilder().addAllMovies(moviesDto).build();
    responseObserver.onNext(response);
    responseObserver.onCompleted();
}

@Override
public void bookSeat(MovieRequest request, StreamObserver<SeatResponse> responseObserver) {
    var seatAssigned = bookASeatForMovieIdUseCase.execute(request.getId());
    var seatAssignedDto = seatMapper.toDto(seatAssigned);
    responseObserver.onNext(seatAssignedDto);
    responseObserver.onCompleted();
}

```

Implementamos los métodos de servicios definidos en el fichero protobuf del cual nuestra clase extiende. Se puede observar la entidad `StreamObserver`, la cual almacenará la respuesta del servicio mediante el método `onNext(value)` (en concreto un valor del stream) y `onCompleted()` que notifica que el streaming ha finalizado correctamente. El código restante se encarga de consultar, en el primer caso, un listado de películas en cartelera mediante un caso de uso y en el segundo de reservar un asiento para una película. Poner especial atención en la clase MovieRequest, ya que es uno de los mensajes generados en el fichero protobuf.

A continuación arrancamos nuestra aplicación de la manera habitual que se hace con una aplicación Spring Boot y observamos el log:

```

  .   ____          _            __ _ _
 /\\ / ___'_ __ _ _(_)_ __  __ _ \ \ \ \
( ( )\___ | '_ | '_| | '_ \/ _` | \ \ \ \
 \\/  ___)| |_)| | | | | || (_| |  ) ) ) )
  '  |____| .__|_| |_|_| |_\__, | / / / /
 =========|_|==============|___/=/_/_/_/
 :: Spring Boot ::                (v2.7.7)

2023-01-21 23:27:47.724  INFO 11896 --- [           main] com.jfjara.grpc.GrpcServiceApplication   : Starting GrpcServiceApplication using Java 17.0.5 on jjb with PID 11896 
2023-01-21 23:27:47.725  INFO 11896 --- [           main] com.jfjara.grpc.GrpcServiceApplication   : No active profile set, falling back to 1 default profile: "default"
2023-01-21 23:27:48.493  INFO 11896 --- [           main] g.s.a.GrpcServerFactoryAutoConfiguration : Detected grpc-netty-shaded: Creating ShadedNettyGrpcServerFactory
2023-01-21 23:27:48.874  INFO 11896 --- [           main] n.d.b.g.s.s.AbstractGrpcServerFactory    : Registered gRPC service: com.jfjara.grpc.infraestructure.proto.CinemaService, bean: cinemaServiceImpl, class: com.jfjara.grpc.infraestructure.grpc.service.CinemaServiceImpl
2023-01-21 23:27:48.874  INFO 11896 --- [           main] n.d.b.g.s.s.AbstractGrpcServerFactory    : Registered gRPC service: grpc.health.v1.Health, bean: grpcHealthService, class: io.grpc.protobuf.services.HealthServiceImpl
2023-01-21 23:27:48.874  INFO 11896 --- [           main] n.d.b.g.s.s.AbstractGrpcServerFactory    : Registered gRPC service: grpc.reflection.v1alpha.ServerReflection, bean: protoReflectionService, class: io.grpc.protobuf.services.ProtoReflectionService
2023-01-21 23:27:49.147  INFO 11896 --- [           main] n.d.b.g.s.s.GrpcServerLifecycle          : gRPC Server started, listening on address: *, port: 9090
2023-01-21 23:27:49.178  INFO 11896 --- [           main] com.jfjara.grpc.GrpcServiceApplication   : Started GrpcServiceApplication in 1.941 seconds (JVM running for 2.369)

```

Se puede observar que se registran los servicios de health del servicio, servicios propios de la comunicación gRPC y nuestro servicio CinemaService. Ya podemos consumir estos servicios desde un cliente.

### Cliente

Usaremos:

- Java 17
- Spring Boot 2.7.7
- grpc-client-spring-boot-starter
- Librería de interfaz gRPC previamente generada
- Lombok
- MapStruct
- API REST con Spring Boot Web

En nuestro fichero pom.xml establecemos las siguientes dependencias:

```

	<dependencies>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter</artifactId>
		</dependency>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-web</artifactId>
		</dependency>
		<dependency>
			<groupId>org.springframework</groupId>
			<artifactId>spring-web</artifactId>
		</dependency>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-test</artifactId>
			<scope>test</scope>
		</dependency>
		<dependency>
			<groupId>net.devh</groupId>
			<artifactId>grpc-client-spring-boot-starter</artifactId>
			<version>${net.devh.version}</version>
		</dependency>
		<dependency>
			<groupId>com.jfjara</groupId>
			<artifactId>grpc-interface</artifactId>
			<version>${com.jfjara.grpc-interface}</version>
		</dependency>
		<dependency>
			<groupId>org.projectlombok</groupId>
			<artifactId>lombok</artifactId>
			<optional>true</optional>
			<version>${org.projectlombok}</version>
		</dependency>
		<dependency>
			<groupId>org.mapstruct</groupId>
			<artifactId>mapstruct</artifactId>
			<version>${org.mapstruct.version}</version>
		</dependency>
	</dependencies>
	

```

Del mismo que el servidor, la lógica del cliente en su starter no varía. Introducimos librerías de Spring Boot Web para definir un controlador REST para nuestras llamadas al cliente.

Nuestro siguiente paso es definir las siguientes propiedades en nuestro application.yaml:

```

grpc:
  client:
    local-grpc-server:
      address: 'static://127.0.0.1:9090'
      negotiationType: plaintext

```

Se establece como nombre del cliente «local-grpc-server» al cuál haremos referencia desde nuestra clase cliente gRPC y cargue esta configuración. Se introduce la url donde está nuestro servidor y el puerto, así como el tipo de negociación usada para HTTP/2, que en nuestro caso será texto plano sin soporte SSL.

A continuación vemos la clase de cliente gRPC:

```

@Service
public class GrpcSpringClient implements GetMoviesRepository {

    @GrpcClient("local-grpc-server")
    private CinemaServiceGrpc.CinemaServiceBlockingStub service;

    @Autowired
    private MovieMapper mapper;

    @Override
    public List<MovieDto> getMovies() {
        var response = service.getMovies(EmptyMessage.newBuilder().build());
        return mapper.toDtos(response.getMoviesList());
    }
}

```

Hemos creado un adaptador del puerto GetMoviesRepository donde internamente inyectamos el cliente gRPC haciendo referencia a «local-grpc-server» que previamente configuramos en el yaml. Y para usarlo tan sencillo como llamar a service.getMovies().

NOTA: no he sido capaz de definir un servicio sin parámetros de entrada en la firma del servicio del fichero protobuf. De ahí que haya definido un mensaje vacío para pasarlo por parámetros mediante EmptyMessage.newBuilder().build().

Ahora toca arrancar el servicio de cliente y hacer una petición a nuestro endpoint de la API REST:

```

@RestController
public class CinemaController {

    @Autowired
    private GetMoviesUseCase getMoviesUseCase;

    @GetMapping("/movies")
    public ResponseEntity<List<MovieDto>> getMovies() {
        var movies = getMoviesUseCase.execute();
        return ResponseEntity.ok(movies);
    }

}


```

En nuestro controller tenemos un caso de uso que internamente llama al cliente gRPC mediante el puerto que implementamos previamente (GetMoviesRepository). Hacemos la llamada a la API:

```

curl --location --request GET 'http://localhost:8080/movies'

```

y obtenemos el resultado:

```

[
    {
        "id": "fbd2f58b-65ba-452b-b26f-ccc1edff60c5",
        "title": "Ghostbusters"
    },
    {
        "id": "e7a4829d-3a83-46d9-909c-7e2211c97448",
        "title": "Back to the future"
    },
    {
        "id": "9fa1d17a-bbcf-4126-9608-45fac76a6444",
        "title": "The big Lebowsky"
    },
    {
        "id": "e7659061-cd0c-46f0-bef1-f151bd8785b5",
        "title": "GoodFellas"
    }
] 

```

# Conclusión

Hemos generado de manera muy sencilla servicios y mensajes mediante el lenguaje protobuf, desarrollado un servidor implementando estos servicios y un cliente que los consume. Para poder observar los resultados hemos creado en el cliente una API REST con una llamada de ejemplo para obtener resultados consultados al servidor. Vemos que el uso y manejo de gRPC está integrado de manera muy sencilla e intuitiva al framework de Spring Boot por lo que facilita aún más si cabe el uso de este tipo de llamadas.

A continuación os invito a seguirme en GitHub, donde podrás encontrar la implementación completa de este proyecto.
