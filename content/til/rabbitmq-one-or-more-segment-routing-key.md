+++
date = '2026-03-15T20:54:37+01:00'
draft = true
title = 'Comment créer un binding avec une routing key aillant au moins un segment dans RabbitMQ'
+++

Récemment, j'ai découvert un bug dans un système que je développe et qui se base sur RabbitMQ pour de l'échange de messages.

On dispose d'un exchange `distribution` de type `topic` et d'une queue `foo.test.message`. Il existe aussi d'un binding reliant l'exchange et la queue avec la configuration suivante :

```yaml
source: distribution
destinationType: queue
destination: foo.test.message
routingKey: foo.test.*.message
```

*Note : il s'agit là d'un extrait de la configuration d'une CRD `Binding` de `rabbitmq.com/v1beta1`.*

Plusieurs applications du système publient des messages sur un exchange `distribution` de type `topic` avec un routing key dépendant d'une valeur métier au sein du message. La routing key est construite à la volée au moment de poster le message.

Un changement a été effectué dans le format de la valeur métier, rajoutant un segment au sein de celle-ci. Lors des tests d'intégration, on s'est aperçu que les messages avec le nouveau format ne circulaient plus correctement.

Je n'avais pas identifié d'impact lié au changement de format de la valeur métier car `*` correspondait, dans ma compréhension des routing key au niveau des bindings, à une wildcard complète, autorisant n'importe quels caractères.

Ce n'est en réalité pas du tout le cas. Dans RabbitMQ, les routing key sont construites par segments, correspondant à une chaîne alphanumérique séparés par un `.`. Par exemple la routing key `foo.test.hello.message` est composée de quatre segments. Il existe en plus deux symboles spéciaux utilisables :

- `*` qui signifie "exactement un segment"
- `#` qui signifie "zéro ou n'importe quel nombre de segments"

Dans mon cas, pour faire fonctionner le nouveau format, je devais autoriser "un segment ou plus " au lieu de "exactement un segment". Mais ce n'est visiblement pas possible avec les possibilités laissées par le format des routing key dans RabbitMQ.

Après 10 bonnes minutes à pester, j'ai finalement trouvé la solution, devant mes yeux depuis le début.

On peut combiner les deux symboles spéciaux. Ainsi "un segment ou plus" peut s'écrire `*.#`, ce qui donne dans mon cas `foo.test.*.#.message`.

La méthode peut être étendue pour le cas général "n segments ou plus" avec n entier naturel, qui donne `*.(<*. n-2 fois>).*.#`, même si je trouve cette notation honnêtement très peu lisible (et assez crado).
