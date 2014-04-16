---
layout: post
title:  "Construyendo un DSL en Ruby"
date:   2014-04-02 09:06:00
categories: ruby rails design-patterns
---

La expresividad sintáctica de Ruby es una de las características por las cuales es más querido y se debe en gran medida a las herramientas únicas que ofrece para construir DSLs. Las DSL en Ruby, de hecho y por ser tan habituales, se han convertido en algo intrínseco al lenguaje y a la forma con que escribimos código; tanto que muchas veces, por costumbre de verlas, llegamos a olvidarnos de lo que son.

## Caso de uso

Recientemente hemos estado desarrollando una herramienta para migrar información entre bases de datos diferentes. Durante la asignación de las correspondencias entre elemento migrado y elemento receptor nos encontrábamos con situaciones como esta:  

{% highlight ruby %}

def attributes_from_old_database_to_new_database_for_user
    new_db_user.balance = old_db_user.balance
    new_db_user.birthday = old_db_user.birthdate
    new_db_user.status = old_db_user.is_active ? "active" : "inactive"
end

{% endhighlight %}

El mapeo entre atributos de una a otra tabla se explicitaba en métodos como éste, dando lugar a un sistema efectivo a corto plazo pero engorroso y poco flexible; tratándose de Ruby, esto representa ya un *Code Smell*. Pero había además un problema añadido: cada vez que aplicábamos la migración nos arriesgábamos a perder información de la base de datos nueva, sobreescrita con información accidental proveniente de la base de datos antigua. Por ejemplo, el *balance* en la nueva base de datos podía ser sobreescrito por el *balance* "legacy" almacenado en la base de datos antigua aunque en ningún momento este segundo hubiese sido modificado, dando lugar a una situación incómoda para ambos cliente y desarrollador.

## La sintaxis del DSL

Para refactorizar -y reparar- métodos como el anterior necesitábamos una solución que idealmente fuese a la vez el esquema de la correspondencias entre bases de datos y la aplicación de dicho esquema. Pensamos en convertirlos en algo parecido a esto:

{% highlight ruby %}

migration :new_db_element, :old_db_element

correspondences_for :user do
    assigns( :balance  ).to :balance
    assigns( :birthday ).to :birthdate
    assigns( :status   ).to { old_db_user.is_active ? "active" : "inactive" }
end

{% endhighlight %}

La sintaxis pretende ser suficientemente autodescriptiva pero hay una sutileza: para mantener la generalidad de la DSL decidimos conservar el contexto del bloque que se pasa opcionalmente al método **#to**; por eso la variable old\_db\_user está disponible dentro suyo cuando en realidad **#assigns** está siendo llamado sobre la clase.

## Construyendo el DSL

Encapsulamos toda la lógica de nuestra DSL bajo el namespace SmartMigrations. Veamos cómo pinta su interfaz pública:

{% highlight ruby %}

module SmartMigrations

    def migration subject, sender = nil, options = {}
        @assignation = Assignation.new subject, sender, options
    end

    def correspondences_for resource_name, options = {}
        assignation = @assignation 

        define_method( method_name_from resource_name ) do
            assignation.attributes_to_update = @attributes.keys
            assignation.action = self
            yield
            instance_exec &( self.class::SM_SAVE_METHOD )
        end
    end

    def assigns key, options = {}
        @assignation.assign key, options
    end

private

    (...)

end

{% endhighlight %}

En primer lugar **#migration** inicializa una instancia de la clase Assignation -que es la que se encargará de hacer efectiva cada una de las correspondencias- con una referencia al objeto receptor de la migración y otra al objeto del cual se migra. Dado que esta instancia es el contenedor lógico para el proceso de asignación y por lo tanto ambos deben ser parte de su estado interno, ese es el lugar natural para almacenar dicha información. 

El método **#correspondences_for** define en la clase base -aquella que se extenderá con nuestro *module*- los métodos sustitutos de aquellos que antes de la refactorización ejecutaban la asignación. Aquí es donde reside gran parte de la magia de **SmartMigrations**, y ocurre gracias al hecho de que en el interior del bloque que se le pasa a *define_method* nos encontramos en el contexto de la instancia de la clase base, el método definido, pero el código yielded se ejecuta en su contexto de definición: efectivamente, si recordamos la sintaxi de la DSL veremos que lo que se pasa como bloque a **#correspondences_for** son las llamadas a assigns, que es un método de clase definido por **SmartMigrations**. Es justamente para lograr la interacción entre ambos contextos por lo que se hace esa extraña definición de la variable *assignation* al principio del método. Dentro del bloque de *define_method* no tendremos el contexto de la clase y por lo tanto tampoco su estado, pero sí las variables definidas en el método -[este post][instance_eval] profundiza en el asunto-. De esta manera conseguimos trasladar nuestro contenedor lógico para el proceso de asignación, *@assignation*, al contexto del método a definir y como resultado de ella ahora podemos introducir información relacionada con dicho contexto en la lógica de asociación. Justamente ese es el caso al seteamos *attributes\_to\_update* y *action*. Con todo ello logramos emular la definición de un método dinámico que se generará a si mismo sabiendo escoger qué asignaciones definir en función de unas condiciones que estipularemos en la clase **Assignation**. 

Las últimas líneas son una forma generalizada de guardar la asignación. Ejecutan dentro de la instancia, el contexto donde existe objeto asignado, un bloque construido (utilizando el operador *&*) a partir del *proc* definido en la constante *SM\_SAVE\_METHOD* de la clase base.

Finalmente **#assigns** delega en **Assignation** el resto del proceso:

{% highlight ruby %}

class Assignation

  attr_reader :key, :values, :value, :options
  attr_accessor :attributes_to_update, :action

  def initialize subject, sender, options = {}
    @subject = subject
    @sender  = sender
    @options = options
  end

  def assign key, options = {}
    @key     = key
    @options = @options.merge options

    self
  end

  def to *values, &block
    @values = values
    @value  = values.last
    @block  = block

    assign_value if fulfills_conditions_for_assignment
  end

(...)

private

    def assign_value
        subject.send key.to_s.to_setter, assignation_value
    end

(...)

end

{% endhighlight %}

**Assignation** es bastante sencilla. Vemos que **#assign** únicamente devuelve *self*, de forma que podamos concatenar otros métodos de instancia, tras modificar el estado interno de la instancia para representar las propiedades del sujeto y preparar la lógica que será lanzada cuando se llame a **#to** sobre ella. Es este segundo método el que realmente dará lugar a la asignación al ser invocado. Dado que toda la información relacionada con la migración en curso está almacenada en las diferentes variables de instancia tenemos plena libertad para encapsular dentro de **Assignation** toda la lógica relacionada con ella, permitiéndonos construir métodos como **#fulfills\_conditions\_for\_assignment** o **#assignation_value** según nuestras necesidades y con los únicos límites de nuestra imaginación.

## Extensibilidad

Aunque hemos expuesto aquí una funcionalidad mínima del DSL, la flexibilidad que se consigue con esta implementación hace de su extensibilidad algo natural ya sea añadiendo métodos a la interfaz de **SmartMigrations** o customizando los de **Assignation**. En particular nosotros hemos definido una asignación condicional, **#assigns_if**, y establecido la correspondencia entre sujeto y objeto de la migración a través de operadores diferentes al de asignación (por ejemplo usando *<<*).

[instance_eval]: http://www.dan-manges.com/blog/ruby-dsls-instance-eval-with-delegation