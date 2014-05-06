---
layout: post
title:  "Consejos y best practices de Rspec"
date:   2014-05-02 09:06:00
categories: rspec tips
author: Jesús Prieto
---

Siempre ha habido un gran debate en la comunidad de Ruby sobre por qué usar una capa de lógica intermedia para los tests, como es el framework de RSpec, cuando se puede usar el Pure old Ruby de Test unit. Pero al margen de la discusión todo el mundo admite las bondades no subjectivas de RSpec por las que fue inventado encabezadas por la conveniencia sintáctica y de estructuración de código de su DSL para hacer un approach del desarrollo basado en el TDD. Si estás pensando en darle una oportunidad, ya sea para introducirte en el testing de Ruby o para hacer el cambio desde otra herramienta, o si a ti ya te ha convencido, aquí tienes unos poco consejos para que el código de tus tests sea más elegante y eficiente. 

## Eager load

Todos los **subject** o variables **let** que llamas en Rspec se evalúan en el momento de la llamada, son eager load. Esto te permite dejarlos indicados de forma genérica y customizar los valores que toman en cada test, como se ilustra a continuación:  

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
          context "when valid variables" do
            it "returns a UTMCoordinates instance" do
              expect( subject.class ).to eq UTMCoordinates
            end
          end

          context "when invalid variables" do
            let( :x ) { -23  }
            let( :y ) { 3241 }
            it "raises error" do
              expect{ subject }.to raise_error( InvalidCoordinates )
            end
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

Este es un oldie. Si usas [FactoryGirl][factory-girl], como seguramente deberías, incluye el módulo **FactoyGirl::Syntax::Methods** a tu configuración para poder llamar a todos sus métodos a través del objeto de RSpec de forma que puedas usarlos sin prefijar **FactoryGirl**.

{% highlight ruby %} 
  # random_spec.rb

  # Sin Factory Girl syntax methods
  FactoryGirl.create :user

  # Lo mismo con ellos
  create :user
{% endhighlight %}

Para ello, dentro del bloque **configure** de ***spec\_helper.rb***:

{% highlight ruby %} 
  # spec_helper.rb

  RSpec.configure do |config|
    config.include FactoryGirl::Syntax::Methods
    (...)
  end
{% endhighlight %}

## Let!

Como decíamos en el primer consejo **#let** tiene eager load, de forma que no se ejecuta cuando se declara sino cuando es llamado. El método **#Let!** tiene la misma funcionalidad pero en este caso no es eager load. Así, si declaras un **#let!** el bloque evaluado se ejecutará antes de cada ***example***. Estos [ejemplos de la documentación en RelishApp][let-let] son muy ilustrativos.

## Imagen de Carrierwave con FactoryGirl

Si utilizas [Carrierwave][carrierwave] para el upload de imágenes y quieres testear algo alrededor de un recurso subido a través de un uploader puedes configurar su factoría de este modo. Digamos que tenemos un modelo ***Photo*** con una imagen asociada a través del método definido por **#mount_uploader** llamado ***image***:

{% highlight ruby %} 
  class Photo < ActiveRecord::Base
    mount_uploader :image, ImageUploader
  end
{% endhighlight %}

A la factoría añadimos la definición de ***image*** de esta forma:

{% highlight ruby %} 
  FactoryGirl.define do
    factory :photo do
      image Rack::Test::UploadedFile.new( File.open( File.join( Rails.root.to_s, 'tu_ruta_a_la_imagen' ) ) )
    end
  end
{% endhighlight %}

Donde solo tienes que sustituir tu\_ruta\_a\_la\_imagen por la ruta a la imagen de prueba que quieras utilizar (nosotros las ponemos en '/spec/fixtures/nombre\_del\_modelo') y los nombres de cada atributo por los de tu modelo. Así tus fotos se crearán con una instancia válida de su uploader asociada.

## Shared examples

Son una gran herramienta para [DRY][dry] el código de tus specs. Consisten en encapsular los examples compartidos por más de un **#context** o **#describe** dentro de un **#shared\_examples\_for**:

{% highlight ruby %}
  shared_examples_for 'animal' do |animal|
    it "breaks things" do
      expect( animal.move.breaks? ).to be_true 
    end
  end
{% endhighlight %}

Y reutilizarlos donde sea necesario, por ejemplo aquí:

{% highlight ruby %}
  describe Hulk do
    subject { Hulk.new }
    it_behaves_like 'animal', subject
  end
{% endhighlight %}

## Mocks y stubs en RSpec

En primer lugar necesitamos saber qué son los mocks y los stubs en RSpec, y en qué se diferencian. Un stub reemplaza un método por código que se evaluará en su lugar. Un mock es lo mismo que un stub pero añade la expectativa de que ese método sea llamado.

Su utilidad es incalculable para contener la extensión de un conjunto de tests al contexto que estan testeando de manera que cuando testees una clase o un método solo estés testeando su responsabilidad y no la de partes del código externas a ello, haciéndolos así confusos (no conviene el test de una clase falle por un error en una clase diferente) y potencialmente lentos (sobretodo si tienen que interactuar con la base de datos). Y esto es algo que quieres en todos tus tests unitarios. Pondremos un par de ejemplos de uso:

{% highlight ruby %}
  describe UserManager do
    subject { UserManager.new user }
    let( user ) { create( :user ).stub( email: 'test@email.com' ) }
  
    it "creates users" do
      User.should_receive( :create ).and_return true
      subject.create_user
    end

    it "reads user attributes" do
      ( subject.read_user user, :email ).to eq 'test@email.com'
    end
  end
{% endhighlight %}

En el primer example se utiliza un mock: **#should_receive** genera una expectación y si esta no se cumple el test va a fallar; en el segundo se utiliza un stub para evitar una conexión en potencia con **ActiveRecord** (para hacer un retrieve del email). Ambos métodos tienen un gran espectro de posibilidades: **#stub** puede recibir bloques o strubear cualquier instancia de una clase, **#should_receive** puede expectar argumentos en concreto o un número de invocaciones, y un largo etcétera, con lo que recomendamos encarecidamente echar un ojo a la documentación tanto de [uno][should_receive] como del [otro][stub] para abrir los ojos a todas sus posibilidades. 

{% highlight ruby %}
  describe Hulk do
    subject { Hulk.new weapon: weapon }
    let( :enemy  ) { double hp: 1000 }
    let( :weapon ) { double dmg: 10  }

    it "kills with a weapon" do
      subject.weapon = weapon
      expect( subject.kill? enemy ).to be_true
    end
  end
{% endhighlight %}

En este caso introducimos el método **#double** que devuelve un objeto al que se han stubeado los métodos de las keys de su inicialización con los valores asociados. Haciendo el test de esta forma la parte de **#kill?** relacionada con Hulk es la única que está siendo sujeta a testeo, en lugar de involucrar a los objetos que hubiesen sido enemy y weapon de otra manera y aislando el test.

<!-- Falta shoulda matchers y metadatos -->

[shoulda-matchers]: https://github.com/thoughtbot/shoulda-matchers
[dry]: http://es.wikipedia.org/wiki/No_te_repitas
[factory-girl]: https://github.com/thoughtbot/factory_girl
[let-let]: https://www.relishapp.com/rspec/rspec-core/docs/helper-methods/let-and-let
[carrierwave]: https://github.com/carrierwaveuploader/carrierwave
[should_receive]: https://www.relishapp.com/rspec/rspec-mocks/v/2-6/docs/message-expectations
[stub]: https://www.relishapp.com/rspec/rspec-mocks/v/2-6/docs/method-stubs