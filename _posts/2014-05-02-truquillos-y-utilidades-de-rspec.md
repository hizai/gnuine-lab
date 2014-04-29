---
layout: post
title:  "Consejos y best practices de Rspec"
date:   2014-05-02 09:06:00
categories: rspec tips
author: Jesús Prieto
---

---- Text introductori ----

## Eager load

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

## Custom matchers

Se almacenan habitualmente en **/spec/support/matchers** y se usan de esta manera:

{% highlight ruby %} 
    it { should assign_results_for( users, scoped_users ) }
{% endhighlight %}

Si la sintaxi recuerda a [shoulda matchers][shoulda-matchers] es porque los [shoulda matchers][shoulda-matchers] son solamente un conjunto de custom matchers. Con ellos se evita repetición y se refactoriza código que de otra forma sería un largo conjunto de bloques **#it**, ya que ellos mismos son capaces de deducir a partir del nombre del matcher los mensajes de éxito y fallo que se printarán cuando se pase el suit de tests.

Definirlos es sencillo, basta con decirle qué tiene que **#match**: el matcher evaluará OK cuando dicha condición se cumpla. Los argumentos del bloque son los argumentos con los que se llama a matcher y **actual** es el objeto sobre el que se llama (generalmente será **subject**). Opcionalmente se puede customizar la descripción y el output del test de forma que sobreescriba el que tendría por defecto:

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

## Factory Girl syntax methods

Este es un oldie. 

[shoulda-matchers]: https://github.com/thoughtbot/shoulda-matchers
[dry]: http://es.wikipedia.org/wiki/No_te_repitas