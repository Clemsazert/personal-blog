+++
date = '2026-03-14T09:24:24+01:00'
draft = false
title = 'Comment stocker des certificats SSL dans un secret Kubernetes'
+++

Dans un des systèmes que je maintiens, on établit des connexions mTLS entre applications pour qu'elles puissent communiquer les unes avec un broker RabbitMQ. Ce mode de connexion repose sur la présentation de certificats SSL, avec vérifications des autorités de confiance, mais je ne vais pas présenter le fonctionnement de ce mécanisme en détail ici.

En ajoutant une nouvelle application au système, j'ai eu besoin d'exposer le certificat lui permettant de s'authentifier. J'ai stocké celui-ci dans notre instance Vault au format `PEM`, permettant une lecture plus simple et un meilleur support par Vault qu'un fichier binaire au format `.p12`.

On utilise le Vault Secret Operator pour faire le lien entre les secrets stockés dans Vault et les secrets Kubernetes, via des CRDs `VaultStaticSecret`. Au moment de créer le `VaultStaticSecret` associé à mon nouveau certificat, je pensais simplement référencer le chemin d'accès du secret dans Vault qui disposait de deux champs, un pour le certificat et un pour la clé privé. Je pourrais alors envoyer le tout dans un secret Kubernetes de type `Opaque` comme j'ai déjà eu l'occasion de le faire avec d'autres informations sensibles stockées dans Vault.

Par curiosité, je suis allé consulter la documentation de la CRD `VaultStaticSecret`. Dans l'objet `destination`, on trouve un champ `type` qui correspond au type de secret Kubernetes souhaité. Connaissant uniquement le type `Opaque`, j'ai voulu voir quels autres options étaient à ma disposition.

J'ai alors découvert dans la documentation de Kubernetes le type `tls` pour les secrets, qui correspond exactement à mon cas d'usage, le stockage de certificats.

Fondamentalement, il n'y a pas une grosse différence avec la solution utilisant un secret de type `Opaque`, les deux peuvent fonctionner.

Cependant, je vois plusieurs avantages à utiliser ce type si l'on en a la possibilité.

1. Le secret est composé de deux clés, `tls.crt` pour la chaîne de certificats, et `tls.key` pour la clé privée. Il permet une standardisation de la manière dont les certificats sont stockés et peut servir d'interface figée entre un producteur de certificat et une application qui va les consommer.

2. À la création, l'API Kubernetes valide la présence des deux champs. Il ne valide cependant pas le contenu.

3. C'est une convention déjà bien en place dans l'écosystème Kubernetes que l'on retrouve dans `certmanager` par exemple.

4. Ce type de secret peut être référencé dans les `Ingress` et s'intégrer de manière native avec cette ressource. Cela fonctionne très bien avec Traefik sans avoir à lui indiquer comment extraire la chaîne de certificats et la clé privée.

5. Il est possible d'avoir une ségrégation encore plus fine dans les permissions accordées par ressource au moment de définitions de RBAC.

Dans quelques autres cas, je trouve que le type `Opaque` peut quand même être intéressant.

1. Une application (souvent legacy) sur laquelle on n'a pas la possibilité de changer le nom du certificat injecté et le nom de la clé.

2. Une application aillant besoin d'un format autre que `PEM`. Ici, on peut malgré tout s'en sortir avec un `InitContainer` container qui gère la transformation en `.p12` ou `.pfx` par exemple.
