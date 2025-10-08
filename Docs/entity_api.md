# Dépendance : Entity API (`entity:entity`)

Ce document explique le rôle du module `Entity API` en tant que dépendance de Drupal Commerce. Comprendre cette relation est crucial pour saisir comment Commerce est architecturé et pourquoi il est si extensible.

## 1. Ce que fait le module `Entity API`

Le module `Entity API` est une extension de l'API d'entité du cœur de Drupal. Il ne fournit pas de fonctionnalités visibles pour l'utilisateur final, mais offre aux développeurs un ensemble d'outils, de classes de base et de design patterns qui simplifient et standardisent la création et la gestion d'entités personnalisées.

En résumé, il prend l'API d'entité de Drupal et la rend plus puissante et plus facile à utiliser.

## 2. Pourquoi Commerce en dépend ?

L'architecture de Drupal Commerce est **entièrement basée sur les entités**. Presque tout est une entité :

*   `commerce_store` (la boutique)
*   `commerce_product` (le produit)
*   `commerce_product_variation` (la variation achetable)
*   `commerce_order` (la commande)
*   `commerce_promotion` (la promotion)
*   etc.

Plutôt que de réécrire toute la logique de bas niveau pour gérer ces entités (création des routes, des pages de listing, des permissions, des onglets locaux...), Commerce s'appuie sur les fondations robustes et éprouvées fournies par `Entity API`.

Cela permet au code de Commerce d'être :
*   **Plus léger :** Il n'a pas besoin de réimplémenter une logique déjà fournie par `Entity API`.
*   **Plus propre :** Il suit des conventions et des design patterns bien établis.
*   **Plus maintenable :** En cas de mise à jour de l'API d'entité de Drupal, une grande partie du travail d'adaptation est gérée par `Entity API`.

## 3. Éléments de `Entity API` utilisés par Commerce

Commerce utilise principalement les **"Handlers" (gestionnaires) par défaut** fournis par `Entity API`. Les handlers sont des classes responsables d'une partie spécifique de la logique d'une entité. Ils sont définis dans l'annotation de la classe de l'entité (le bloc de commentaire au-dessus de `class ...`).

Voici les handlers les plus importants que Commerce délègue à `Entity API` :

### a. Fournisseur de Routes (`route_provider`)

*   **Ce que c'est :** Une classe qui génère automatiquement les routes (URL) standards pour une entité : la page de visualisation, le formulaire d'ajout, le formulaire d'édition, le formulaire de suppression, et la page de collection (listing).
*   **Handler `Entity API` utilisé :** `Drupal\entity\Routing\DefaultHtmlRouteProvider`
*   **Comment Commerce l'utilise :** La plupart des entités de Commerce n'ont pas besoin de définir leur propre `route_provider`. Elles héritent de celui fourni par `Entity API`, qui crée pour elles toutes les routes nécessaires. C'est un gain de temps et de code considérable.

### b. Constructeur de Listes (`list_builder`)

*   **Ce que c'est :** Une classe qui génère la page de listing d'administration pour une entité (par exemple, la table sur `/admin/commerce/products`).
*   **Handler `Entity API` utilisé :** `Drupal\entity\EntityListBuilder` (en tant que classe de base).
*   **Comment Commerce l'utilise :** Pour les pages de listing, Commerce crée ses propres classes de `ListBuilder` (ex: `Drupal\commerce_product\ProductListBuilder`), mais celles-ci **étendent** presque toujours la classe `EntityListBuilder` de `Entity API`. Commerce ne fait que surcharger les colonnes à afficher, mais toute la logique de base (requête, pagination, rendu) est héritée.

### c. Fournisseur de Tâches Locales (`local_task_provider`)

*   **Ce que c'est :** Une classe qui génère les onglets de navigation locaux sur la page d'une entité (par exemple, les onglets "Voir", "Modifier", "Variations", "Supprimer" sur la page d'un produit).
*   **Handler `Entity API` utilisé :** `Drupal\entity\Menu\DefaultEntityLocalTaskProvider`
*   **Comment Commerce l'utilise :** C'est un exemple parfait. La plupart des entités de Commerce n'ont même pas besoin de déclarer ce handler dans leur annotation. Le système utilise automatiquement celui de `Entity API`, qui crée les onglets standards en se basant sur les routes qui ont été générées par le `route_provider`. Commerce n'a donc rien à coder pour obtenir cette fonctionnalité standard.

### d. Fournisseur de Permissions (`permission_provider`)

*   **Ce que c'est :** Une classe qui génère dynamiquement la liste des permissions pour un type d'entité (ex: "Voir les produits", "Modifier n'importe quel produit", "Créer des produits", etc.).
*   **Handler `Entity API` utilisé :** `Drupal\entity\EntityPermissionProvider`
*   **Comment Commerce l'utilise :** Comme pour les autres handlers, Commerce s'appuie sur la classe de base de `Entity API` pour générer automatiquement les permissions CRUD (Create, Read, Update, Delete) pour chaque type d'entité (Store, Product, Order...). Cela garantit une gestion des permissions cohérente à travers tout l'écosystème.

### Exemple concret dans le code

Si vous regardez l'annotation de l'entité `commerce_store` (`modules/contrib/commerce/modules/store/src/Entity/Store.php`), vous verrez la section `handlers`. Certains sont définis par Commerce (comme `list_builder`), mais beaucoup d'autres sont omis. C'est parce que Commerce se repose sur les implémentations par défaut fournies par `Entity API` pour tout le reste, ce qui illustre parfaitement cette dépendance.
