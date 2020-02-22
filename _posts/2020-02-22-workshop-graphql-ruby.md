---
layout: post
title:  graphql-ruby workshop
categories: [Ruby,Rails,GraphQL,gem]
---

Chez Tymate, la question de l'utilisation de GraphQL pour nos API se pose depuis longtemps. La plupart de nos applications desktop étant conçues avec React, c'était évidemment une demande récurrente de nos dévelopeurs front-end.

Je m'appelle Julien, j'ai 32 ans et je suis développeur Ruby depuis un peu plus d'un an, après une reconversion professionnelle. Cet article est une retranscription libre d'un workshop interne à Tymate au cours duquel j'ai présenté mon retour d'expérience sur une première utilisation de la gem GraphQL-ruby.

La suite présume une connaissance sommaire du fonctionnement de GraphQL.

## Structure des données

La première chose à savoir en commençant à travailler avec GraphQL, c’est que tout est typé. Chaque donnée qu’on utilise ou qu’on veut sérialiser doit être clairement définie. Ce n'est pas quelque chose à quoi on est naturellement habitué avec Ruby mais qui s'avère rapidement très confortable.

### Types

On va donc naturellement commencer par définir des types.

D’abord, on créera des types pour les modèles de notre api dont on aura besoin pour nos requêtes GraphQL. La définition des types GraphQL ressemble beaucoup à la logique qu’on retrouve dans les serializers sur nos API REST classiques, à savoir la définition des données qu’on pourra consulter, et la forme sous laquelle ces données sont mises à disposition, mais va évidemment plus loin en intégrant le typage, des validations, éventuellement une gestion de droits...

Ci-dessous, un exemple de type GraphQL correspondant à un model de notre API :

```ruby
# frozen_string_literal: true

module Types
  class ProviderType < Types::BaseObject
    # Ces lignes permettent d'intégrer les spécificités de Relay au type
    # on y reviendra plus bas dans l'article
    implements GraphQL::Relay::Node.interface
    global_id_field :id

    # Ici, on définit les champs qu'on va pouvoir requêter sur notre objet
    # il peut s'agit de champs en base, de méthodes issues du model
    # voire de méthodes directement définies dans le type.
    # Pour chaque champ, on doit spécifier le type et la possibilité d'être nulle.
    field :display_name, String, null: true
    field :phone_number, String, null: true
    field :siret, String, null: true
    field :email, String, null: true
    field :iban, String, null: true
    field :bic, String, null: true
    field :address, Types::AddressType, null: true
    field :created_at, GraphQL::Types::ISO8601DateTime, null: true
    field :updated_at, GraphQL::Types::ISO8601DateTime, null: true

    # Ici, on choisit de réécrire la relation address du model.
    # La simple définition du field address suffit pour retourner les bons objets.
    # mais ce faisant, on peut intégrer la logique de la gem BatchLoader pour
    # éradiquer les N+1 de notre requête.
    def address
      BatchLoader::GraphQL.for(object).batch do |providers_ids, loader|
        Address.where(
          addressable_type: 'Provider', addressable_id: providers_ids
        ).each { |address| loader.call(object, address) }
      end
    end  
  end
end

```

### Queries

Une fois qu'on a défini les types dont on a besoin, ils apparaissent certes directement dans la documentation générée automatiquement par GraphiQL mais ils ne peuvent pas être directement consultés par les consommateurs de l'API.

Pour cela, il est nécessaire de créer des queries. Il s'agit de l'équivalent d'une requête GET en REST. Le consommateur va demander à consulter soit un objet spécifique, soit une collection d'objets (qu'on appellera une *connection* dans le cadre de la méthologie Relay, j'y reviendrai).

#### Afficher un objet isolé

Les queries se définissent dans la classe `QueryType` :

```ruby
# frozen_string_literal: true

module Types
  class QueryType < Types::BaseObject
    # Ici, le field correspond au nom de la query
    # les arguments passés dans le block permettront de retrouver l'objet recherché
    field :item, Types::ItemType, null: false do
      argument :id, ID, required: true
    end

    # contrairement à la définition des types de modèles vu plus haut, pour les queries
    # le field ne se suffit pas à lui-même. Il est nécessaire de définir dans une
    # méthode du même nom la logique qui va résoudre la query.
    def item(id:)
      Item.find(id)
    end
  end
end

```

Dans son fonctionnement, une query est très proche de ce qu'on fait dans un controller REST. On définit la requête dans un *field*, auquel on associe le type du ou des objets qu'on va retourner, et des arguments qui vont permettre de récupérer l'objet voulu, par exemple un ID.

Dans la méthode item, on récupère les arguments passés dans le field, et on va chercher l'objet correspondant.

##### Alléger ses queries

Dès qu'on commence à avoir beaucoup de models différents qu'on veut pouvoir afficher dans une requête, on va être confronté à pas mal de duplication de code. Un peu de metaprogrammation permet d'alléger notre `query_type.rb` :

```ruby
# frozen_string_literal: true

module Types
  class QueryType < Types::BaseObject
    # On définit d'abord une méthode commune qui retrouve un objet
    # à partir d'un ID unique fourni par Relay
    def item(id:)
      SergicApiSchema.object_from_id(id, context)
    end

    # Et on crée les fields qui appelent la méthode sus-définie pour tous
    # les types pour lesquels on a besoin d'une query
    %i[
      account_code
      account_place_entry
      attachment
      biller
      identity
      place
      provider
      repartition_key
    ].each do |method|
      field method, "Types::#{method.to_s.camelize}Type".constantize, null: false do
        argument :id, ID, required: true
      end

      alias_method method, :item
    end
  end
end

```

#### Afficher une collection d'objets

Par défaut, il n'y a pas grande différence entre une query qui retourne un objet et une autre qui retourne une collection d'objets. On va tout simplement devoir spécifier que le résultat de la requête sera un tableau en passant le type lui-même entre `[]` :

```ruby
# frozen_string_literal: true

module Types
  class QueryType < Types::BaseObject
    field :items, [Types::ItemType], null: false do
      # on ne cherche plus un item par son ID, mais par son contenant
      argument :category_id, ID, required: true
    end

    def item(category_id:)
      Category.find(category_id).items
    end
  end
end
```

On appelle cette query ainsi :

```
query items {
  items {
    id
    displayName
  }
}

```

Et on obtient le json suivant en réponse :

```json
[
  {
    "id": "1",
    "displayName": "toto"
  },
  {
    "id": "2",
    "displayName": "tata"
  }
]

```

### Relay et les connections

Cela peut fonctionner pour des besoins très basiques, mais ça va être vite très limité.

Pour des requêtes mieux structurées, GraphQL-ruby permet d'utiliser Relay, une surcouche à GraphQL aussi créée par Facebook, qui introduit deux paradigmes très pratiques dans GraphQL : 

- on travaille avec des IDs globaux (des strings créé à partir d'un encodage en base64 de `["NomDuModel, ID"]`)
- une nomenclature bien spécifique pour organiser et consommer les collections d'objets : les `connections`

> *edit: J'ai réalisé le workshop initial en m'appuyant sur la version v1.9 de la gem graphql-ruby. La notion de connection a été extraite de Relay pour devenir le formatage par défaut des collections dans la v1.10.*

L'ID global à la place des ID classiques est là avant toute chose pour les applications JS qui vont consommer l'API. Cela permet notamment de toujours utiliser cet ID comme clé dans les boucles d'objets.

#### Nomenclature 

Pour ce qui est des connections, voici à quoi ressemble notre précédente query adaptée sous ce format :

```
{
  items(first: 2) {
    totalCount
    pageInfo {
      hasNextPage
      hasPreviousPage
      endCursor
      startCursor
    }
    edges {
      cursor
      node {
        id
        displayName
      }
    }
  }
}

```

Et la réponse correspondante :

```json
{
  "data": {
    "items": {
      "totalCount": 351,
      "pageInfo": {
        "hasNextPage": true,
        "hasPreviousPage": false,
        "endCursor": "Mg",
        "startCursor": "MQ"
      },
      "edges": [
        {
          "cursor": "MQ",
          "node": {
            "id": "UGxhY2UtMzUy",
            "displayName": "Redbeard"
          }
        },
        {
          "cursor": "Mg",
          "node": {
            "id": "UGxhY2UtMzUx",
            "displayName": "Frey of Riverrun"
          }
        }
      ]
    }
  }
}

```

La connection nous donne accès des informations complémentaires en plus de notre collection. Par défaut, la gem nous génère le `pageInfo` qui sert à la pagination par curseur, mais on peut également ajouter des champs personnalisés comme ici le totalCount rajouté pour gérer une pagination numérotée plus traditionnelle.

Les `edges` sont prévus pour contenir des informations liés au rapport entre l'objet et sa collection. Un peu comme une table de liaison. Par défaut, il va contenir le curseur, qui représente le positionnement du node qu'il contient au sein de la collection. Mais il est possible de définir ses propres champs personnalisés.

Les `nodes` sont tout simplement les objets de la collection.

#### Intégration

L'utilisation de relay apporte des données précieuses aux requêtes, mais elle requiert en conséquence plus de verbosité dans la définition des queries. Concrètement, au lieu d'une query, on va devoir définir 3 types :
- la query
- la connection
- le edge

##### ConnectionType

La définition du type de la connection comprend 3 éléments :
- la spécification du EdgeType à utiliser
- les paramètres que l'on peut appliquer à cette connection
- les champs personnalisés qu'on peut demander en retour

```ruby
# frozen_string_literal: true

class Types::Connections::ProvidersConnectionType < Types::BaseConnection
  # On commence par appeler la classe du EdgeType qu'on veut associer
  # à la conenction
  edge_type(Types::Edges::ProviderEdge)

  # En définissant ici des types Input, on pourra ensuite les appeler dans
  # les queries associées à cette connection.
  # Je reviens plus bas sur les arguments eq/ne/in plus spécifiquement.
  class Types::ProviderApprovedInput < ::Types::BaseInputObject
    argument :eq, Boolean, required: false
    argument :ne, Boolean, required: false
    argument :in, [Boolean], required: false
  end

  class Types::ProviderFilterInput < ::Types::BaseInputObject
    argument :approved, Types::ProviderApprovedInput, required: false
  end

  # Absent par défaut, on peut intégrer un compteur du nombre d'objets
  # dans la collection en créant un field et son resolver dans le ConnectionType
  field :total_count, Integer, null: false

  def total_count
    skipped = object.arguments[:skip] || 0
    object.nodes.size + skipped
  end
end

```

##### EdgeType

Le Edge peut être pensé comme une table de liaison, qui fait la passerelle entre la connection et les nodes qu'elle contient. Par défaut, on doit y définit le node_type pour identifier le type d'objet retournés par notre connection. Mais il est également possible d'y définir des méthodes personnalisés.
Je n'ai cependant pas encore rencontré de cas d'usage pour cette possibilité.

```ruby
# frozen_string_literal: true

class Types::Edges::ProviderEdge < GraphQL::Types::Relay::BaseEdge
  node_type(Types::ProviderType)
end

```

##### QueryType

Enfin, une fois la connection bien définie, il faut l'appeler dans une query (ou directement en dépendance dans un type de model). Pour cela, au lieu d'associer le field au type de l'objet final retourné, on l'associe au type de la connection.

```ruby
# frozen_string_literal: true

module Types
  class QueryType < Types::BaseObject
    field :providers, Types::Connections::ProvidersConnectionType, null: true

    def providers
      Provider.all
    end
  end
end

```

Par défaut, on peut passer tous les arguments de pagination de base à la requête (first, after, before, last...). Si nécessaire, on peut spécifier des arguments supplémentaires pour préciser notre requête :

```ruby
# frozen_string_literal: true

module Types
  class QueryType < Types::BaseObject
    field :providers, Types::Connections::ProvidersConnectionType, null: true do
      # filter appelle les InputTypes spécifiques à ProvidersConnectionType
      # qu'on a définit plus haut
      argument :filter, Types::ProviderFilterInput, required: false
    end

    def providers
      Provider.all
    end
  end
end

```

##### Extraction des queries

Toutes les queries qu'on veut exposer sur notre API doivent être définies dans le fichier `query_type.rb`montré juste avant. Mais avec le gain en complexité d'une API, le fichier va être vite surchargé. Alors il est évidemment possibler d'extraire la logique des queries dans d'autres fichiers, des resolvers.

Le fichier `query_type.rb` se présentera alors ainsi :

```ruby
# frozen_string_literal: true

module Types
  class QueryType < Types::BaseObject
    # ...

    field :all_providers_connection, resolver: Queries::AllProvidersConnection
    
    # ...
end
```

La logique de la query va se trouver dans un fichier à part :

```ruby
# frozen_string_literal: true

module Queries
  class AllProvidersConnection < Queries::BaseQuery
    description 'list all providers'

    type Types::Connections::ProvidersConnectionType, null: false

    argument :filter, Types::ProviderFilterInput, required: false
    argument :search, String, required: false
    argument :skip, Int, required: false
    argument :order, Types::Order, required: false

    def resolve(**args)
      res = connection_with_arguments(Provider.all, args)
      res = apply_filter(res, args[:filter])
      res
    end
  end
end

```

Les méthodes personnalisées `connection_with_arguments` et `apply_filter` sont définies dans `BaseQuery`.

##### Trier et filtrer les connections

`connection_with_arguments` me permet d'intégrer des arguments de tri et de pagination numérotée à mes queries.

```ruby
def connection_with_arguments(res, **args)
  o = args[:order] || { by: :id, direction: :desc }
  res = res.filter(args[:search]) if args[:search]
  res = res.offset(args[:skip]) if args[:skip]
  res = res.order("#{res.model.table_name}.#{o[:by]} #{o[:direction]}")
  res
end
```

`apply_filter` permet d'avoir une logique globale à tous les arguments `filter` de l'API.

Habituellement, dans nos API REST, nous avons l'habitude d'utiliser des scopes pour permettre aux utilisateurs de filtrer les résultats d'une requête. Mais ces filtres restent assez sommaires. Dans la conception de cette première API GraphQL, en travaillant avec mon collègue développeur React et consommateur de l'API, nous avons souhaité aller un peu plus loin. GraphQL permet de choisir précisément les données qu'on désire recevoir, alors autant donner également la possibilité de les filtrer avec précision.

On a donc cherché une structure existante pour formater les filtres et nous avons décidé de nous baser sur la norme proposée dans la documentation de GatsbyJS : https://www.gatsbyjs.org/docs/graphql-reference/#skip

> ### Complete list of possible operators
>
> *In the playground below the list, there is an example query with a description of what the query does for each operator.*
> 
> * `eq` : short for `equal`, must match the given data exactly
> * `ne` : short for `not equal`, must be different from the given data
> * `regex` : short for `regular expression`, must match the given pattern. Note that backslashes need to be escaped twice, so `/\w+/` needs to be written as `"/\\\\w+/"`.
> * `glob` : short for `global`, allows to use wildcard * which acts as a placeholder for any non-empty string
> * `in` : short for `in array`, must be an element of the array
> * `nin` : short for `not in array`, must NOT be an element of the array
> * `gt` : short for `greater than`, must be greater than given value
> * `gte` : short for `greater than or equal`, must be greater than or equal to given value
> * `lt` : short for `less than`, must be less than given value
> * `lte` : short for `less than or equal`, must be less than or equal to given value
> * `elemMatch` : short for `element match`, this indicates that the field you are filtering will return an array of elements, on which you can apply a filter using the previous operators

L'intégration actuelle ressemble à ceci :

```ruby
def apply_filter(scope, filters)
  return scope unless filters

  filters.keys.each do |filter_key|
    filters[filter_key].keys.each do |arg_key|
      value = filters[filter_key][arg_key]

      # Ici on traduit les ID globaux qu'on reçoit du consommateur de l'API
      # en ID classiques reconnus par Postgresql
      if filter_key.match?(/([a-z])_id/)
        value = [filters[filter_key][arg_key]].flatten.map do |id|
          GraphQL::Schema::UniqueWithinType.decode(id)[1].to_i
        end
      end

      scope = case arg_key
              when :eq, :in
                scope.where(filter_key => value)
              when :ne, :nin
                scope.where.not(filter_key => value)
              when :start_with
                scope.where("#{filter_key} ILIKE ?", "#{value}%")
              else
                scope
              end
    end
  end

  scope
end
```

C'est encore un travail en cours, le code est rudimentaire, mais cela suffit pour filtrer les premières requêtes.

### Pagination par curseur

Sur cette premier application, les écrans étaient encore pensés avec une pagination numérotée. J'ai donc dû intégrer des champs par défaut à toutes les connections :

```ruby
# frozen_string_literal: true

class Types::BaseConnection < GraphQL::Types::Relay::BaseConnection
  field :total_count, Integer, null: false
  field :total_pages, Integer, null: false
  field :current_page, Integer, null: false

  def total_count
    return @total_count if @total_count.present?

    skipped = object.arguments[:skip] || 0
    @total_count = object.nodes.size + skipped
  end

  def total_pages
    page_size = object.first if object.respond_to?(:first)
    page_size ||= object.max_page_size
    total_count / page_size + 1
  end

  def current_page
    page_size = object.first if object.respond_to?(:first)
    page_size ||= object.max_page_size
    skipped = object.arguments[:skip] || 0
    (skipped / page_size) + 1
  end
end

```

Cela fonctionne, mais c'est regrettable parce que GraphQL est vraiment pensé pour la navigation par curseur, on est donc obligé de réinventer la roue à plusieurs endroits alors qu'on a un fonctionnement clef en main et certainement plus performant à disposition.

On gagnerait certainement beaucoup de perf à pousser la pagination par curseur partout ou on pourrait se passer du nombre de page total, du numéro de page, du nombre d'éléments dans la collection.

## TODO: conclure