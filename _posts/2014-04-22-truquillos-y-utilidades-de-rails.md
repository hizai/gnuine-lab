---
layout: post
title:  "Truquillos y utilidades para la vida del desarrollador de Rails"
date:   2014-04-22 09:06:00
categories: rails tips
author: Jesús Prieto
---

Cada framework o lenguaje de programación está lleno de capacidades infrautilizadas y lugares menos transitados de lo que deberían, cosas de esas que una vez descubres te preguntas cómo habías suportado tanto tiempo sin ellas. Esta es una lista de algunos de los que usamos los desarrolladores de Gnuine cuando escribimos aplicaciones en Rails.

## El modo sandbox de la consola de Rails

Pasándole el flag sandbox a la inicialización de la consola de Rails la abrirás en modo **rollback changes**. Cualquier modificación que hagas en la base de datos será revertida cuando llames **exit**. 

{% highlight bash %}
    rails console --sandbox
{% endhighlight %}

Púlete la base de datos de un solo **#delete_all**, fuerza la creación usuarios que no pasan las validaciones a tutiplen. Un mundo de maravillosas y alocadas posibilidades se despliega ante tus ojos en sandbox mode.

## El comando _ de IRB

¿Cuántas veces has tenido que repetir lo último evaluado en la consola para guardarlo en una variable? Con el comando **\_** de IRB eso se va acabar ya que **\_** almacena el último valor devuelto por el intérprete. 

## Pry como rails console

Añadir **gem 'pry'** al Gemfile. Bundle install. Crear un initializer:

{% highlight ruby %}
    # pry.rb

    if Rails.env.development?
        silence_warnings do
          begin
            require 'pry'
            IRB = Pry
          rescue LoadError
          end
        end
    end
{% endhighlight %}

Vale la pena.

(La gema pry-rails hace exactamente esto).

## Queries SQL mostradas por consola

¿Tienes una versión antigua de Rails y algo o alguien está toqueteando tu base de datos sin que sepas cómo ni por qué? Loguea las queries por consola. Mano de santo.

{% highlight ruby %}
    ActiveRecord::Base.connection.instance_variable_set :@logger, Logger.new(STDOUT)
{% endhighlight %}

Si las quieres activas por defecto, añadir esta línea al archivo **.irbrc** dará el pego.

## Bundle open

Con este comando de bundle tan injustamente desconocido se abre el directorio local de la gema indicada  con el editor definido en la variable de entorno **EDITOR**.

{% highlight bash %}
  export EDITOR=subl && bundle open rails
{% endhighlight %}

## Código fuente y localización de un método

Puedes ver tanto el código de un método como su localización utilizando los métodos **#source** y **source_location** del objecto **method** asociado a un método. Nivel dragón de metaprogramación.

{% highlight ruby %}
  object.method(:foo).source
  object.method(:foo).source_location
{% endhighlight %}

Y si usas **pry** como **rails console** incluso puedes abrir el método en el editor de texto al hacer un edit,

{% highlight ruby %}
  edit object.method(:foo)
{% endhighlight %}

## Testear integración desde la consola

Desde testear rutas a lanzar acciones de controlador haciendo requests desde la consola, el método **#app** de Rails está dispuesto a todo.

{% highlight ruby %}
  app.root_path
  
  app.get edit_post_path( post )
  app.response.body
{% endhighlight %}

## Testear helpers desde la consola

Primo hermano de **#app**, **#helper** devuelve una instancia de **ActionView::Base** que es la inclusora por excelencia de los helpers.

{% highlight ruby %}
  helper.link_to "Home", app.root_path
  helper.my_custom_helper
{% endhighlight %}

Si el **my_custom_helper*** en cuestión depende de **params** se puede utilizar el siguiente workaround para maquearlo:

{% highlight ruby %}
  helper.controller = OpenStruct.new params: { user: User.last }
  helper.my_custom_helper
{% endhighlight %}

## Generar la documentación de Rails en local

¿Aislado en las montañas, sin rastro de ondas wifi y con mono de Rails? Rails viene con un conjunto de rake tasks para autogenerar HTML de la documentación en local. Échales un ojo con:

{% highlight ruby %}
  rake -T | grep doc
{% endhighlight %}

## POROs

Si Rails es un Framework y ese Framework es un conjunto de librerías, Rails es un conjunto de librerías. Una de las cosas que marcan la diferencia es no olvidar que pase lo que pase estás trabajando con una aplicación de Ruby; y Ruby es un lenguaje orientado a objetos, característica ésta que a veces, cuando alguien escribe una aplicación Rails en lugar de una aplicación que usa Rails, se pasa por alto. La moraleja de todo esto son los Pure Old Ruby Objects. Si tienes modelos gordos cual [Snorlaxs][snorlax] y controladores donde cada método ocupa más de cinco líneas seguramente has olvidado que tanto unos como otros son objetos de Ruby que, tal como están, violan conceptos básicos sin los cuales la orientación a objetos pierde la razón de ser -como [Single Responsability Principle][single_responsability_principle]-. Una primera lectura interesante para introducirse a este nuevo paradigma es [este post de Codeclimate][code_climate_poros].

[snorlax]: http://www.dltk-kids.com/pokemon/adoptions/143.gif
[dry]: http://es.wikipedia.org/wiki/No_te_repitas
[single_responsability_principle]: http://es.wikipedia.org/wiki/Single_responsibility_principle
[code_climate_poros]: http://blog.codeclimate.com/blog/2012/10/17/7-ways-to-decompose-fat-activerecord-models/