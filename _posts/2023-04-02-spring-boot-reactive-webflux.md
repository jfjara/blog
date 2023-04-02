---
layout: single
title: "Reactividad con Spring Boot y Webflux"
excerpt: "En el siguiente post mostramos un ejemplo de uso con algunas de las características que nos brinda Webflux a la hora de desarrollar nuestros proyectos mediante el paradigma de programación reactiva"
date: 2023-04-02
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
tags:  
  - springboot
  - java
  - reactive
  - webflux
---

![](/assets/images/htb-writeup-delivery/delivery_logo.png)

En el siguiente post mostramos un ejemplo de uso con algunas de las características que nos brinda Webflux a la hora de desarrollar nuestros proyectos mediante el paradigma de programación reactiva.

Esta entrada al tratarse esctrictamente de una práctica no entraremos en detalles sobre lo qué es la programación reactiva ni sus características. Ya existen muchas webs donde se explica muy bien, no entraremos en esa competencia. Nos dedicaremos exclusivamente a resolver un problema práctico donde necesitemos el uso de dicha paradigma de programación.

Tenemos un programa que expone 3 endpoints GET para obtener información de facturas ficticias. Las operaciones son las siguientes:

- Obtener todas las facturas
- Obtener una factura por identificador
- Obtener detalles extendidos de una factura con una tienda asociada

Para tener varios repositorios de datos hemos creado dos fuentes de datos, ambos son puertos mocks que nos devuelven datos autogenerados, de ese modo podemos poner en practica algunas de las características de la programación reactiva. A futuro incorporaremos una cache y una base de datos mediante contenedores para realizar pruebas más ajustadas a un entorno real. Para este punto, con puertos mocks nos es suficiente para llegar al objetivo de la práctica. Detallamos a continuación las operaciones que vamos a utilizar para manejar nuestros flujos de datos:

- zip: tomaremos dos flujos de datos y los unimos sincronamente para realizar una transformación de datos. Es decir, se ejecutan dos procesos y cuando ambos finalicen, tomamos el resultados de ambos y montamos nuestro objeto de factura extendida.
- merge: como hemos comentado, tenemos dos fuentes de datos que son puertos mocks, en ambos nos devuelven facturas. Con esta operación unimos los flujos de ambas fuentes para servirlas por la misma operación GET.

Para la práctica hemos utilizado el siguiente stack tecnológico:

- Java 17
- Spring Boot 2.7.10
- Lombok
- Mapstruct
- Webflux
- Arquitectura Hexagonal

A continuación dejamos el enlace al [repositorio de Github](https://github.com/jfjara/reactive-springboot) donde se encuentra el proyecto. En futuras actualizaciones realizaremos las operaciones que anteriormente hemos comentado.

