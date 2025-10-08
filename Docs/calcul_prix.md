# Le Calcul du Prix dans Drupal Commerce

Ce document explique la chaîne complète de calcul qui permet d'aboutir au prix final d'une commande dans Drupal Commerce. Ce processus est divisé en deux grandes étapes :

1.  **La résolution du prix de base** : Déterminer le prix d'une seule unité d'un article (une variation de produit).
2.  **Le traitement de la commande** : Ajuster le prix total de la commande en appliquant les promotions, les taxes, la livraison, etc.

Ces deux étapes utilisent des design patterns similaires, proches de la **Stratégie** et de la **Chaîne de Responsabilité**, pour offrir une flexibilité maximale.

## 1. Étape 1 : La Résolution du Prix de Base (Le Pattern "Resolver")

Lorsqu'un prix doit être affiché pour une variation de produit (sur la page produit ou dans le panier), Drupal Commerce ne se contente pas de lire une valeur dans un champ. Il utilise un service appelé **Chain Price Resolver**.

*   **Service clé** : `commerce_product.chain_variation_price_resolver`
*   **Pattern utilisé** : Chaîne de Responsabilité (Chain of Responsibility).

### Comment ça marche ?

Le `ChainPriceResolver` ne calcule rien lui-même. Son rôle est d'appeler, dans un ordre défini par une priorité, une série d'autres services, appelés des "resolvers". Chacun de ces resolvers implémente l'interface `PriceResolverInterface`.

Le processus est le suivant :
1.  Le `ChainPriceResolver` est appelé avec l'entité achetable (la variation) et un objet `Context` (contenant l'utilisateur, la boutique, l'heure, etc.).
2.  Il parcourt sa liste de resolvers, du plus prioritaire au moins prioritaire.
3.  Pour chaque resolver, il appelle sa méthode `resolve()`.
4.  Si un resolver parvient à déterminer un prix, il retourne un objet `Price`. La chaîne s'arrête immédiatement et ce prix est utilisé.
5.  Si un resolver ne peut pas déterminer de prix pour la situation actuelle (par exemple, un resolver de "prix VIP" pour un utilisateur normal), il retourne `NULL`. La chaîne continue alors avec le resolver suivant.

### Les Resolvers par Défaut

Par défaut, Drupal Commerce est livré avec un resolver principal :

*   **`DefaultPriceResolver`** (priorité -100) : C'est le resolver de dernier recours. Sa logique est très simple : il lit le prix dans le champ `price` de la variation de produit et le retourne.

Cela signifie que vous pouvez créer vos propres resolvers avec une priorité plus élevée (ex: 0, 10, 100) pour surcharger le prix par défaut dans certaines conditions.

### Exemple : Créer un Resolver pour un Prix "Membre"

Imaginons que les utilisateurs ayant le rôle "membre" bénéficient d'un prix spécial sur certains produits.

1.  **Créez un service qui implémente `PriceResolverInterface`** dans votre module personnalisé.

    ```php
    // dans src/Resolver/MemberPriceResolver.php
    namespace Drupal\my_module\Resolver;

    use Drupal\commerce\Context;
    use Drupal\commerce_price\Resolver\PriceResolverInterface;
    use Drupal\commerce_product\Entity\ProductVariationInterface;

    class MemberPriceResolver implements PriceResolverInterface {

      public function resolve(ProductVariationInterface $variation, $quantity, Context $context) {
        $customer = $context->getCustomer();
        // Si l'utilisateur n'est pas membre, on ne fait rien. La chaîne continue.
        if ($customer->isAnonymous() || !in_array('membre', $customer->getRoles())) {
          return NULL;
        }

        // Si la variation a un champ "prix membre" (field_member_price) et qu'il est rempli...
        if ($variation->hasField('field_member_price') && !$variation->get('field_member_price')->isEmpty()) {
          // ... on retourne ce prix. La chaîne s'arrête ici.
          return $variation->get('field_member_price')->first()->toPrice();
        }

        // Sinon, on ne fait rien. Le resolver suivant (DefaultPriceResolver) prendra le relais.
        return NULL;
      }
    }
    ```

2.  **Enregistrez ce service en le taguant** dans le fichier `my_module.services.yml`.

    ```yaml
    services:
      my_module.member_price_resolver:
        class: Drupal\my_module\Resolver\MemberPriceResolver
        tags:
          # On le tague comme un resolver de prix avec une priorité de 100.
          # Il s'exécutera donc avant le DefaultPriceResolver (priorité -100).
          - { name: commerce_product.price_resolver, priority: 100 }
    ```

---

## 2. Étape 2 : Le Traitement de la Commande (Le Pattern "Processor")

Une fois que les articles sont dans le panier (qui est une entité `commerce_order`), leurs prix de base sont connus. Cependant, le total de la commande doit encore être ajusté pour inclure les promotions, les taxes, etc. C'est le rôle de l'**Order Processor**.

*   **Service clé** : `commerce_order.order_processor`
*   **Pattern utilisé** : Chaîne de Responsabilité (Chain of Responsibility).

### Comment ça marche ?

Le fonctionnement est très similaire à celui des resolvers de prix. Le service `OrderProcessor` parcourt une chaîne de "processeurs de commande" (des services implémentant `OrderProcessorInterface`).

Contrairement à la chaîne des resolvers, **la chaîne des processeurs ne s'arrête pas**. Chaque processeur est exécuté l'un après l'autre.

Le processus est le suivant :
1.  Le service `OrderProcessor` est appelé avec l'entité `commerce_order`.
2.  Il parcourt sa liste de processeurs, du plus prioritaire au moins prioritaire.
3.  Chaque processeur exécute sa logique :
    *   Il peut lire des informations sur la commande (articles, client...).
    *   S'il doit modifier le prix, il n'altère pas directement le total. Il ajoute un objet **`Adjustment`** à la commande.
    *   Un `Adjustment` contient le type de l'ajustement (promotion, taxe...), son libellé, et son montant (qui peut être positif ou négatif).
4.  À la fin, un processeur final (`OrderTotalProcessor`) calcule le total final en additionnant le sous-total des articles et tous les ajustements qui ont été ajoutés.

### Les Processeurs par Défaut

L'ordre des processeurs est crucial. Par défaut, il est généralement le suivant :

1.  `OrderTotalSummary` (priorité 100) : Calcule le sous-total en additionnant le prix de tous les articles.
2.  `PromotionOrderProcessor` (priorité 0) : Cherche les promotions applicables et ajoute des ajustements de type `promotion`.
3.  `TaxOrderProcessor` (priorité -100) : Calcule les taxes sur le sous-total (généralement après les promotions) et ajoute des ajustements de type `tax`.
4.  `OrderTotalProcessor` (priorité -200) : Calcule le total final de la commande.

Si vous installez `commerce_shipping`, un `ShippingOrderProcessor` viendra s'insérer dans cette chaîne pour ajouter les frais de port (un autre `Adjustment`).

### Exemple : Créer un Processeur pour des Frais de Dossier

Imaginons que nous voulons ajouter 5€ de frais de dossier pour toute commande inférieure à 20€.

1.  **Créez un service qui implémente `OrderProcessorInterface`**.

    ```php
    // dans src/OrderProcessor/HandlingFeeProcessor.php
    namespace Drupal\my_module\OrderProcessor;

    use Drupal\commerce_order\Adjustment;
    use Drupal\commerce_order\Entity\OrderInterface;
    use Drupal\commerce_order\OrderProcessorInterface;
    use Drupal\commerce_price\Price;

    class HandlingFeeProcessor implements OrderProcessorInterface {

      public function process(OrderInterface $order) {
        $subtotal = $order->getSubtotalPrice();
        $fee_limit = new Price('20.00', $subtotal->getCurrencyCode());

        // Si le sous-total est inférieur à 20€...
        if ($subtotal->isLessThan($fee_limit)) {
          // ... on ajoute un ajustement de 5€.
          $order->addAdjustment(new Adjustment([
            'type' => 'handling_fee',
            'label' => 'Frais de dossier',
            'amount' => new Price('5.00', $subtotal->getCurrencyCode()),
            'source_id' => 'my_module',
          ]));
        }
      }
    }
    ```

2.  **Enregistrez ce service en le taguant** dans `my_module.services.yml`.

    ```yaml
    services:
      my_module.handling_fee_processor:
        class: Drupal\my_module\OrderProcessor\HandlingFeeProcessor
        tags:
          # On lui donne une priorité de -50 pour qu'il s'exécute après les promotions
          # mais avant le calcul des taxes, si on veut que les taxes s'appliquent aussi
          # sur ces frais.
          - { name: commerce_order.order_processor, priority: -50 }
    ```

En combinant ces deux mécanismes, Drupal Commerce permet de construire des logiques de tarification extrêmement complexes et spécifiques à chaque besoin métier, tout en gardant une architecture propre et extensible.
