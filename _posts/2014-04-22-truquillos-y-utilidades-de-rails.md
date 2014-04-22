---
layout: post
title:  "Truquillos y utilidades de Rails"
date:   2014-04-22 09:06:00
categories: rails tips
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

## Queries SQL mostradas por consola

¿Algo o alguien está toqueteando tu base de datos y no sabes cómo ni por qué? Loguea las queries por consola. Mano de santo.

{% highlight ruby %}
    ActiveRecord::Base.connection.instance_variable_set :@logger, Logger.new(STDOUT)
{% endhighlight %}

Si las quieres activas por defecto añadir esta línea al archivo **.irbrc** dará el pego.

## Eager load en Rspec

Todos los **subject** o variables **let** que llamas en Rspec se evalúan en el momento de la llamada y son eager load. Esto te permite dejarlos indicados de forma genérica y customizar los valores que toman en cada test, como se ilustra a continuación:  

{% highlight ruby %}
    require 'spec_helper'

    describe Coordinates do 
      subject { Coordinates.parse coords, format }

      context "when UTM coordinates" do 
        let( :coords ) { "#{x}, #{y}" }
        let( :format ) { :UTM }

        let( :x ) { 300581.44  }
        let( :y ) { 5919603.33 }
        describe "#parse" do 
          it "returns a UTMCoordinates instance" do
            expect( subject.class ).to eq UTMCoordinates
          end
        end
      end

      context "when geographical coordinates" do 
        let( :coords ) { "#{longitude}, #{latitude}" }
        let( :format ) { :geographic }

        let( :longitude ) { -36.8079 }
        let( :latitude  ) { 174.2345 }
        describe "#parse" do 
          it "returns a UTMCoordinates instance" do
            expect( subject.class ).to eq GeographicCoordinates
          end
        end 
      end

    end
{% endhighlight %}

## Rspec custom matchers

Se almacenan en **/spec/support/matchers** y se usan en los tests de esta manera:

{% highlight ruby %} 
    it { should assign_results_for( users, scoped_users ) }
{% endhighlight %}

Definir uno es sencillo, basta con decirle qué tiene que **#match**: el matcher evaluará OK cuando esto se cumpla. Los argumentos del bloque son los argumentos con los que se llama a matcher y **actual** es el objeto sobre el que se llama al matcher (generalmente **subject**). Opcionalmente se puede customizar la descripción y el output del test de forma que sobreescriba el que tendría por defecto y que él mismo es capaz de deducir a partir del nombre del matcher:

{% highlight ruby %} 
    # assign_results_for_matcher.rb

    require 'rspec/expectations'

    RSpec::Matchers.define :assign_results_for do |full_scope, scoped|
      match do |actual|
        @results = assigns( :results )
        @expected_results = { total_items: full_scope.size, current_items: scoped.size }

        @results == @expected_results
      end

      description do
        "assigns the results variable appropiately"
      end

      failure_message_for_should do |text|
        "expected #{@results || 'nil'} to match #{@expected_results}"
      end
     
      failure_message_for_should_not do |text|
        "expected #{@results || 'nil'} to not match #{@expected_results}"
      end
    end
{% endhighlight %}

Y recuerda, mantén tus tests [DRY][dry]: ellos también son personas.

## POROs

Si Rails es un Framework y ese Framework es un conjunto de librerías, Rails es un conjunto de librerías. Una de las cosas que marcan la diferencia es no olvidar que pase lo que pase estás trabajando con una aplicación de Ruby; y Ruby es un lenguaje orientado a objetos, característica ésta que a veces, cuando alguien escribe una aplicación Rails en lugar de una aplicación que usa Rails, es pasada por alto. La moraleja de todo esto son los Pure Old Ruby Objects. Si tienes modelos gordos cual [Snorlaxs][snorlax] y controladores donde cada método ocupa más de cinco líneas seguramente has olvidado que tanto unos como otros son objetos de Ruby que, tal como están, violan conceptos básicos de orientación a objetos -como el [Single Responsability Principle][single_responsability_principle]- sin los cuales la orientación a objetos pierde la razón de ser. Una primera lectura interesante para introducirse a este nuevo paradigma es [este post de Codeclimate][code_climate_poros].

[snorlax]: http://www.dltk-kids.com/pokemon/adoptions/143.gif
[dry]: http://es.wikipedia.org/wiki/No_te_repitas
[single_responsability_principle]: http://es.wikipedia.org/wiki/Single_responsibility_principle
[code_climate_poros]: http://blog.codeclimate.com/blog/2012/10/17/7-ways-to-decompose-fat-activerecord-models/