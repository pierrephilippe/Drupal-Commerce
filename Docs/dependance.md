# Dépendances du module Drupal Commerce

Ce document explique le rôle des modules dont dépend directement le module principal de Drupal Commerce (`commerce`). Comprendre ces dépendances est essentiel pour saisir l'architecture globale et les choix techniques du projet.

## 1. Address (`address:address`)

*   **Ce que fait le module :** Le module `Address` fournit un type de champ complexe pour stocker, valider et formater des adresses postales physiques. Il respecte la norme xNAL (eXtensible Name and Address Language) et gère les spécificités de formatage de plus de 200 pays.

*   **Pourquoi Commerce en dépend :** Le commerce électronique repose sur la gestion d'adresses, que ce soit pour la facturation (billing) ou la livraison (shipping). Plutôt que de réinventer un système de gestion d'adresses, Commerce s'appuie sur ce module robuste et standardisé.

*   **Exemple dans Commerce :**
    *   L'entité `profile` (utilisée pour les informations client) contient un champ de type `address`.
    *   L'entité `commerce_store` (la boutique) possède également un champ `address` pour définir son emplacement physique.

## 2. Entity API (`entity:entity`)

*   **Ce que fait le module :** Le module `Entity API` étend l'API d'entité de Drupal Core. Il fournit des classes de base et des outils qui simplifient la création et la gestion d'entités : meilleurs fournisseurs de routes, gestionnaires de listes (`ListBuilder`), gestion des permissions, etc.

*   **Pourquoi Commerce en dépend :** L'architecture de Drupal Commerce est entièrement basée sur les entités (Produit, Variation, Commande, Boutique, etc.). `Entity API` fournit des fondations solides et des design patterns éprouvés que Commerce utilise abondamment pour rendre son propre code plus simple, plus propre et plus maintenable.

*   **Exemple dans Commerce :** La plupart des entités de Commerce, comme `Coupon` (`commerce_promotion_coupon`), utilisent des "handlers" (gestionnaires) définis dans leur annotation. `Entity API` fournit des implémentations par défaut pour beaucoup d'entre eux (par exemple, `Drupal\entity\Menu\DefaultEntityLocalTaskProvider` pour les onglets locaux), ce qui évite à Commerce de devoir réécrire ce code.

## 3. Inline Entity Form (`inline_entity_form:inline_entity_form`)

*   **Ce que fait le module :** Ce module fournit un widget de formulaire qui permet de créer, modifier et supprimer des entités référencées directement depuis le formulaire de l'entité parente.

*   **Pourquoi Commerce en dépend :** Pour l'expérience utilisateur des administrateurs de la boutique. Sans ce module, pour modifier les variations d'un produit, il faudrait quitter la page du produit, trouver la variation, la modifier, puis revenir. C'est fastidieux. `Inline Entity Form` rend l'administration beaucoup plus fluide.

*   **Exemple dans Commerce :** Sur le formulaire de création/modification d'un produit (`commerce_product`), les variations de produit (`commerce_product_variation`) sont gérées via un widget "Inline entity form". Cela permet d'ajouter, de modifier (le prix, le SKU...) ou de supprimer des variations sans jamais quitter la page du produit. C'est aussi utilisé pour les articles de commande (`order_item`) dans une commande.

## 4. Token (`token:token`)

*   **Ce que fait le module :** Le module `Token` fournit une API centralisée pour utiliser des "jetons". Les jetons sont des morceaux de texte entre crochets (ex: `[node:title]`) qui sont remplacés dynamiquement par leur valeur correspondante.

*   **Pourquoi Commerce en dépend :** Une boutique en ligne a besoin de communiquer des informations dynamiques à ses clients ou administrateurs. Les jetons sont le moyen standard dans Drupal pour construire des textes personnalisés.

*   **Exemple dans Commerce :** Lors de l'envoi d'un e-mail de confirmation de commande, le corps de l'e-mail peut contenir des jetons comme `[commerce_order:order_number]` pour afficher le numéro de la commande, `[commerce_order:total_price]` pour le montant total, ou encore `[commerce_order:billing_profile]` pour afficher l'adresse de facturation complète.

## 5. Datetime (`drupal:datetime`)

*   **Ce que fait le module :** C'est un module du cœur de Drupal qui fournit un type de champ pour stocker des dates et des heures, ainsi que les widgets et formateurs associés.

*   **Pourquoi Commerce en dépend :** La gestion du temps est cruciale en e-commerce. Il faut savoir quand une commande a été passée, quand une promotion commence ou se termine, etc.

*   **Exemple dans Commerce :**
    *   L'entité `commerce_promotion` a des champs `start_date` et `end_date` pour définir sa période de validité.
    *   L'entité `commerce_order` a un champ `placed` (date de passage de la commande) et `completed` (date de finalisation).

## 6. Views (`drupal:views`)

*   **Ce que fait le module :** `Views` est le puissant constructeur de requêtes et de listages de Drupal. Il permet de créer des affichages complexes (pages, blocs, flux RSS...) de n'importe quel type de contenu ou d'entité sans écrire une seule ligne de code.

*   **Pourquoi Commerce en dépend :** Pour fournir des vues d'administration prêtes à l'emploi. Au lieu de coder en dur les listes de commandes, de produits, etc., Commerce fournit des Vues par défaut. Cela permet aux développeurs et aux administrateurs de les personnaliser très facilement (ajouter un champ, un filtre, changer l'ordre de tri...).

*   **Exemple dans Commerce :** La page listant les commandes (`/admin/commerce/orders`), la liste des produits (`/admin/commerce/products`), ou encore le panier de l'utilisateur (`/cart`) sont des Vues. Vous pouvez les modifier depuis l'interface d'administration de Views pour les adapter à vos besoins.

## 7. Profile (`profile:profile`)

*   **Ce que fait le module :** Le module `Profile` fournit un type d'entité (`profile`) conçu pour stocker des ensembles d'informations relatives à un utilisateur, mais qui n'ont pas leur place directement sur l'entité `user` elle-même. C'est une solution idéale pour des données qui peuvent être multiples (par exemple, plusieurs adresses pour un même client).

*   **Pourquoi Commerce en dépend :** La gestion des informations client est au cœur du e-commerce. Commerce utilise les entités `profile` pour stocker les adresses de facturation (`billing`) et de livraison (`shipping`) associées à une commande. Cette approche permet de gérer ces informations de manière très flexible, y compris pour les clients anonymes (qui n'ont pas de compte `user`), et de construire la fonctionnalité de "carnet d'adresses" pour les clients connectés.

*   **Exemple dans Commerce :**
    *   L'entité `commerce_order` (la commande) possède des champs de référence d'entité (`billing_profile` et `shipping_profile`) qui pointent vers des entités de type `profile`.
    *   Lors du passage en caisse (checkout), les formulaires d'adresse que remplit le client créent ou mettent à jour ces entités `profile`.
    *   Le carnet d'adresses d'un client (`/user/{user}/addresses`) est une vue qui liste les entités `profile` associées à son compte.
