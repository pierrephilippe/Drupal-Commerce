# Documentation Technique de Drupal Commerce

Cette documentation s'adresse aux développeurs Drupal souhaitant comprendre, utiliser et étendre l'écosystème Drupal Commerce.

## 1. Architecture et Modules Clés

Drupal Commerce est une suite de modules qui fournissent un cadre de commerce électronique flexible et extensible. Son architecture est modulaire, chaque module ayant une responsabilité claire.

### Modules principaux

*   **Commerce** (`commerce`): Le module de base qui fournit les API fondamentales et les concepts partagés (monnaies, etc.).
*   **Price** (`commerce_price`): Gère les prix, les monnaies, le formatage et les calculs.
*   **Store** (`commerce_store`): Permet de gérer un ou plusieurs magasins (boutiques). Chaque magasin a ses propres informations (nom, adresse, monnaie par défaut, etc.).
*   **Order** (`commerce_order`): Définit les commandes et les articles de commande (`order item`). Le panier est en réalité une commande à l'état "draft".
*   **Product** (`commerce_product`): Fournit les types d'entités `commerce_product` et `commerce_product_variation`. C'est la base du catalogue.
*   **Checkout** (`commerce_checkout`): Gère le processus de commande (tunnel d'achat).
*   **Payment** (`commerce_payment`): Fournit l'infrastructure pour les paiements et les passerelles de paiement.
*   **Tax** (`commerce_tax`): Offre une API pour la gestion et le calcul des taxes.
*   **Promotion** (`commerce_promotion`): Permet de créer des promotions et des coupons de réduction.

Ces modules sont interdépendants et s'appuient les uns sur les autres pour former une solution complète.

### API Fondamentales du module `commerce`

Le module `commerce` lui-même, en plus de lier les autres modules, fournit des API de bas niveau qui sont utilisées dans tout l'écosystème.

*   **Country API**
    *   **Service clé** : `commerce.country_repository`
    *   **Rôle** : Fournit une liste de pays et leurs subdivisions (états, provinces, etc.) bien plus complète que celle du cœur de Drupal. Cette API est utilisée par le module `Address` pour la validation et le formatage des adresses.

*   **Currency API**
    *   **Service clé** : `commerce.currency_repository`
    *   **Rôle** : Gère toutes les informations relatives aux monnaies (nom, code ISO 4217, symbole, règles de formatage). C'est la fondation sur laquelle le module `Price` (`commerce_price`) s'appuie pour tous les calculs et affichages de prix.

*   **Locale API**
    *   **Service clé** : `commerce.locale_resolver`
    *   **Rôle** : Détermine le "locale" (contexte géographique et linguistique) de l'utilisateur. Ce locale est ensuite utilisé pour déterminer la monnaie, le pays et les formats de nombres par défaut, ce qui est essentiel pour les boutiques internationales.

*   **Purchasable Entity Interface (`PurchasableEntityInterface`)**
    *   **Rôle** : C'est l'une des abstractions les plus puissantes de Commerce. Il s'agit d'une interface PHP que n'importe quel type d'entité Drupal peut implémenter pour devenir "achetable". Une entité qui implémente cette interface peut être ajoutée à un panier et vendue. Par défaut, `commerce_product_variation` l'implémente, mais vous pourriez l'utiliser pour vendre des `nodes` (ex: des articles premium), des `users` (ex: des abonnements à un rôle), ou tout autre type d'entité personnalisée.

## 2. Design Patterns et Concepts Fondamentaux

Drupal Commerce utilise intensivement les design patterns de Drupal pour offrir une flexibilité maximale.

### a. Entités (Entities)

Presque tout dans Commerce est une entité, ce qui permet d'utiliser l'API Entity de Drupal pour les manipuler (CRUD, champs, vues, etc.).

*   **Store (`commerce_store`)**: La boutique. Une commande est toujours rattachée à un Store.
*   **Product (`commerce_product`)**: Le "produit" au sens marketing. C'est un conteneur pour les variations. Il n'a ni prix ni SKU. Il sert principalement à l'affichage (titre, description, images partagées).
*   **Product Variation (`commerce_product_variation`)**: L'entité achetable. C'est elle qui porte le SKU, le prix, les attributs (taille, couleur...) et qui est ajoutée au panier. Un produit peut avoir plusieurs variations.
*   **Order (`commerce_order`)**: La commande. Elle contient les articles de commande, les informations client, les paiements, les promotions, etc. Le panier est une commande avec le statut `draft`.
*   **Order Item (`commerce_order_item`)**: Une ligne dans une commande. Elle référence une `Product Variation` et contient la quantité et le prix unitaire.
*   **Payment Gateway (`commerce_payment_gateway`)**: Entité de configuration qui définit une méthode de paiement (ex: PayPal, Stripe).
*   **Payment (`commerce_payment`)**: Entité qui représente une transaction financière (un paiement ou un remboursement).

#### Distinction : Product vs. Product Variation (Cas d'usage)

La séparation entre le Produit et la Variation de Produit est fondamentale.

*   Le **Produit (`commerce_product`)** est un conteneur. Il représente le concept général de l'article que vous vendez. Il contient les informations partagées par toutes ses versions : titre principal, description longue, marque, photos génériques, etc. **Il n'est pas achetable directement.**
*   La **Variation de Produit (`commerce_product_variation`)** est l'article spécifique que le client ajoute à son panier. C'est elle qui possède un SKU (référence article), un prix, et des attributs qui la différencient des autres variations (taille, couleur, etc.). **C'est l'entité achetable.**

**Cas d'usage concret : Vendre un T-shirt**

Imaginez que vous vendez un "T-shirt Logo Drupal" disponible en plusieurs tailles et couleurs.

1.  **Vous créez un seul `Product` :**
    *   **Titre** : "T-shirt Logo Drupal"
    *   **Description** : "Un t-shirt de haute qualité, 100% coton bio, parfait pour les DrupalCamps."
    *   **Images** : Une galerie de photos montrant des personnes portant le t-shirt en bleu et en blanc.

2.  **Vous créez plusieurs `Product Variation` pour ce produit :**
    *   **Variation 1** :
        *   SKU : `TSHIRT-DRU-BL-S`
        *   Attributs : `Couleur: Bleu`, `Taille: S`
        *   Prix : `19.99€`
    *   **Variation 2** :
        *   SKU : `TSHIRT-DRU-BL-M`
        *   Attributs : `Couleur: Bleu`, `Taille: M`
        *   Prix : `19.99€`
    *   **Variation 3** :
        *   SKU : `TSHIRT-DRU-WH-M`
        *   Attributs : `Couleur: Blanc`, `Taille: M`
        *   Prix : `21.99€` (peut-être plus cher pour une raison quelconque)

Sur la page du produit, le client verra la description générale et les photos. Il utilisera ensuite des sélecteurs (listes déroulantes pour la taille et la couleur) pour choisir une variation spécifique. Le prix et le bouton "Ajouter au panier" se mettront à jour dynamiquement pour correspondre à la variation sélectionnée.

### b. Plugins

Les plugins sont utilisés partout pour permettre de remplacer ou d'étendre des fonctionnalités sans modifier le code de base.

*   **`CommerceCheckoutFlow`**: Définit les étapes du tunnel de commande. Vous pouvez créer votre propre plugin pour un checkout en une seule page ou avec des étapes personnalisées.
    *   *Exemple*: `MultistepDefault` est le plugin par défaut.
*   **`CommerceCheckoutPane`**: Chaque étape du checkout est composée de "panes" (panneaux), qui sont des plugins. Un panneau est un formulaire (infos de contact, livraison, etc.). Vous pouvez créer les vôtres et les réorganiser.
    *   *Exemple*: `ContactInformation`, `CompletionMessage`.
*   **`CommercePaymentGateway`**: Pour intégrer un nouveau fournisseur de paiement.
*   **`CommerceTaxType`**: Pour définir des types de taxes (ex: TVA française, taxe de vente canadienne).
*   **`CommercePromotionOffer`**: Définit l'action d'une promotion (ex: "10% de réduction sur le total", "Frais de port offerts").
*   **`CommerceCondition`**: Conditions pour les promotions, taxes, etc. (ex: "Le total de la commande est supérieur à 50€").

### c. Services

La logique métier est encapsulée dans des services, suivant les principes d'injection de dépendance de Symfony.

*   **`commerce_order.cart_manager`**: Service pour gérer le panier de l'utilisateur (créer un panier, ajouter des articles).
*   **`commerce_order.order_processor`**: Service qui applique des "processeurs" à une commande pour en recalculer les totaux. Les promotions et les taxes sont implémentées comme des processeurs.
*   **`commerce_price.rounder`**: Service pour arrondir les prix selon les règles de la monnaie.
*   **`commerce_checkout.checkout_order_manager`**: Gère le flux de checkout pour une commande donnée.

### d. Événements (Le Pattern "Observer")

Commerce utilise le composant EventDispatcher de Symfony, qui est une implémentation du **pattern "Observer"**. Commerce (le "sujet") émet des événements à des moments clés de son exécution. Vos modules personnalisés peuvent "s'abonner" (s'enregistrer comme "observateurs") à ces événements pour déclencher des actions spécifiques sans avoir à modifier le code de Commerce.

*   `commerce_cart.entity_add`: Déclenché lorsqu'un article est ajouté au panier.
*   `commerce_order.order.paid`: Déclenché lorsqu'une commande est entièrement payée.
*   `commerce_checkout.completion`: Déclenché juste avant qu'une commande ne soit marquée comme "complétée".
*   `commerce_product.product_variation_ajax_change`: Déclenché lors du changement de variation via AJAX sur la page produit.

Pour souscrire à un événement, créez un `EventSubscriber` dans votre module personnalisé.

### e. Processeurs de Commande (Le Pattern "Chain of Responsibility")

Le service `commerce_order.order_processor` est une implémentation du pattern **"Chain of Responsibility"** (Chaîne de responsabilité).

**Comment ça marche ?**
Quand une commande doit être recalculée (par exemple, après l'ajout d'un article au panier ou l'application d'un coupon), elle est passée à travers une chaîne de "processeurs". Chaque processeur est un service qui a une tâche spécifique :
1.  Un processeur peut ajouter les frais de port.
2.  Un autre peut appliquer les promotions.
3.  Un troisième calcule les taxes.
4.  Un dernier calcule le total final.

Chaque processeur reçoit la commande, la modifie si nécessaire (par exemple en ajoutant un `Adjustment`), et la passe au processeur suivant dans la chaîne. L'ordre des processeurs est défini par une priorité dans la déclaration du service, ce qui est crucial (les promotions sont généralement appliquées avant les taxes). Ce pattern permet d'ajouter ou de réorganiser très facilement des étapes complexes dans le calcul du total d'une commande.

## 3. Comment Utiliser et Personnaliser

### a. Créer un nouveau type de produit

1.  Allez dans `Administration > Commerce > Configuration > Produits > Types de produit`.
2.  Créez un nouveau type de produit (ex: "Vêtement").
3.  Allez dans `Administration > Commerce > Configuration > Produits > Types de variation de produit`.
4.  Créez un type de variation (ex: "Variation de vêtement") et ajoutez-lui des champs (ex: `field_taille`, `field_couleur`).
5.  Modifiez le type de produit "Vêtement" pour qu'il utilise le type de variation "Variation de vêtement".

### b. Personnaliser le tunnel de commande (Checkout)

1.  Allez dans `Administration > Commerce > Configuration > Tunnels de commande`.
2.  Cliquez sur "Modifier" pour le tunnel que vous souhaitez altérer (ex: "Par défaut").
3.  Sur cet écran, vous pouvez :
    *   Réorganiser les panneaux (`panes`) par glisser-déposer.
    *   Changer le panneau d'étape (`login`, `order_information`, etc.).
    *   Désactiver un panneau en le déplaçant dans la section "Désactivé".
    *   Configurer un panneau en cliquant sur l'icône d'engrenage.

Pour des changements plus profonds, comme ajouter une étape :
1.  Créez un nouveau plugin `CommerceCheckoutFlow` dans votre module en étendant `CheckoutFlowWithPanesBase`.
2.  Dans la méthode `getSteps()`, définissez vos propres étapes.
3.  Créez un nouveau tunnel de commande dans l'interface et sélectionnez votre plugin.

### c. Gérer les passerelles de paiement (Payment Gateways)

Les passerelles de paiement sont le cœur du processus de transaction. Ce sont des plugins qui contiennent la logique pour interagir avec un service de paiement externe (comme Stripe, PayPal) ou pour gérer des paiements manuels (virement, chèque).

#### Types de passerelles

Drupal Commerce définit trois grands types de passerelles, qui se traduisent par des classes de base pour vos plugins :

1.  **Off-site (`OffsitePaymentGatewayBase`)** :
    *   **Fonctionnement** : L'utilisateur est redirigé vers le site du fournisseur de paiement pour y saisir ses informations et valider la transaction. Une fois le paiement effectué, il est redirigé vers la boutique.
    *   **Exemples** : PayPal Standard, la plupart des passerelles de banques françaises.
    *   **Avantages** : La boutique n'a pas à gérer les données de paiement sensibles, ce qui simplifie grandement la conformité PCI.
    *   **Méthodes clés à implémenter** : `onReturn()` (gère le retour de l'utilisateur sur le site) et souvent `onNotify()` (gère un "webhook" ou une "IPN" envoyée par le serveur du fournisseur pour confirmer le statut du paiement de manière asynchrone).

2.  **On-site (`OnsitePaymentGatewayBase`)** :
    *   **Fonctionnement** : L'utilisateur saisit ses informations de carte de crédit directement sur le site de la boutique, dans le tunnel de commande. Le serveur de la boutique communique ensuite avec l'API du fournisseur de paiement pour traiter la transaction.
    *   **Exemples** : Stripe (avec Stripe.js, c'est un cas hybride mais la logique est `On-site`), Braintree.
    *   **Avantages** : Expérience utilisateur plus fluide, car il ne quitte jamais le site.
    *   **Inconvénients** : Nécessite une attention particulière à la sécurité et à la conformité PCI, car des données sensibles transitent par le site (même si des solutions comme Stripe.js permettent de ne jamais les stocker sur le serveur).
    *   **Méthodes clés à implémenter** : `createPayment()` (crée et capture le paiement), `createPaymentMethod()` (sauvegarde les informations de paiement pour une utilisation future).

3.  **Manual (`ManualPaymentGatewayBase`)** :
    *   **Fonctionnement** : Pour les paiements qui ne sont pas traités électroniquement en temps réel. La commande est placée, mais le paiement est en attente d'une action manuelle.
    *   **Exemples** : Virement bancaire, paiement par chèque, paiement à la livraison.
    *   **Méthodes clés à implémenter** : `createPayment()`.

#### Créer une passerelle de paiement personnalisée

Si un module contribué n'existe pas pour votre service de paiement, vous pouvez créer le vôtre.

1.  **Créez un module personnalisé**.
2.  **Créez votre plugin** dans `src/Plugin/Commerce/PaymentGateway/`. Le nom du fichier sera `MonService.php`.
3.  **Définissez votre classe de plugin** en étendant l'une des classes de base (`OffsitePaymentGatewayBase`, `OnsitePaymentGatewayBase`, `ManualPaymentGatewayBase`).

    ```php
    // Dans src/Plugin/Commerce/PaymentGateway/MonService.php
    namespace Drupal\mon_module\Plugin\Commerce\PaymentGateway;

    use Drupal\commerce_payment\Plugin\Commerce\PaymentGateway\OffsitePaymentGatewayBase;
    use Drupal\Core\Form\FormStateInterface;

    /**
     * @CommercePaymentGateway(
     *   id = "mon_service",
     *   label = "Mon Service de Paiement",
     *   display_label = "Mon Service",
     *   forms = {
     *     "offsite-payment" = "Drupal\commerce_payment\PluginForm\PaymentOffsiteForm",
     *   },
     *   payment_method_types = {"credit_card"},
     * )
     */
    class MonService extends OffsitePaymentGatewayBase {

      // Implémentez ici les méthodes nécessaires comme :
      // defaultConfiguration(), buildConfigurationForm(), submitConfigurationForm(),
      // onReturn(), onCancel(), onNotify()...

    }
    ```
4.  **Implémentez la logique** : Vous devrez implémenter les méthodes pour le formulaire de configuration (`buildConfigurationForm`, `submitConfigurationForm`) et les méthodes pour gérer le flux de paiement (`onReturn`, `createPayment`, etc.).
5.  **Activez et configurez** : Une fois votre module activé, votre nouvelle passerelle apparaîtra dans `Administration > Commerce > Configuration > Paiement > Passerelles de paiement` où vous pourrez l'ajouter et la configurer.
### d. Implémenter une logique métier personnalisée

**Exemple : Appliquer une réduction si un utilisateur a le rôle "Membre"**

1.  Créez un module personnalisé.
2.  Créez un service qui implémente `OrderProcessorInterface`. Ce service sera votre processeur de commande.
3.  Dans la méthode `process(OrderInterface $order)`, vérifiez si le propriétaire de la commande a le rôle "Membre".
4.  Si c'est le cas, créez un `Adjustment` et appliquez-le à la commande.
    ```php
    use Drupal\commerce_order\Adjustment;
    use Drupal\commerce_price\Price;

    // ... dans la méthode process()
    $customer = $order->getCustomer();
    if ($customer->hasRole('membre')) {
        $amount = new Price('-5.00', $order->getTotalPrice()->getCurrencyCode());
        $order->addAdjustment(new Adjustment([
            'type' => 'custom_discount',
            'label' => 'Réduction Membre',
            'amount' => $amount,
            'source_id' => 'my_module_member_discount',
        ]));
    }
    ```
5.  Enregistrez votre service en le taguant avec `commerce_order.order_processor` et en lui donnant une priorité (ex: avant le processeur de taxes).

    ```yaml
    # my_module.services.yml
    services:
      my_module.order.member_discount_processor:
        class: Drupal\my_module\OrderProcessor\MemberDiscountProcessor
        tags:
          - { name: commerce_order.order_processor, priority: 150 }
    ```

### e. Gérer les stocks avec Commerce Stock

La gestion des stocks n'est pas incluse dans le module principal de Commerce, mais est fournie par le module contribué très populaire **Commerce Stock**. Cette approche modulaire permet aux sites qui n'ont pas besoin de gestion de stock (ex: vente de produits numériques) de ne pas être surchargés par cette fonctionnalité.

**Principe de fonctionnement :**

1.  **Installation** : `composer require drupal/commerce_stock` puis activez le module `commerce_stock`.
2.  **Niveau de suivi** : Le stock est suivi au niveau de la **variation de produit** (`commerce_product_variation`), car c'est l'entité achetable.
3.  **Services** : Le module fournit une API et des services pour interagir avec les niveaux de stock. Le service principal est `commerce_stock.service_manager`.
4.  **Intégration** : Il s'intègre au processus d'ajout au panier et de passage de commande pour :
    *   Vérifier la disponibilité d'un produit avant de l'ajouter au panier.
    *   Empêcher la finalisation d'une commande si un produit n'est plus en stock.
    *   Décrémenter le stock lorsqu'une commande est passée.

**Mise en place de base :**

1.  Après avoir installé le module, allez dans `Administration > Commerce > Configuration > Stock`.
2.  Ici, vous pouvez configurer les **emplacements de stock** (par défaut, il utilise les `Stores` de la boutique).
3.  Allez dans `Administration > Commerce > Configuration > Produits > Types de variation de produit`.
4.  Modifiez le type de variation pour lequel vous souhaitez suivre le stock (ex: "Variation de vêtement").
5.  Activez la case à cocher **"Gérer le stock pour ce type de variation"**. Cela ajoutera un champ de gestion de stock à toutes les variations de ce type.
6.  Modifiez une variation de produit. Vous verrez maintenant une section "Stock" où vous pouvez :
    *   Activer ou désactiver la gestion du stock pour cette variation spécifique.
    *   Définir le niveau de stock.

**Personnalisation avancée :**

Le module `commerce_stock` est lui-même extensible. Vous pouvez par exemple :

*   **Créer des types de transactions de stock personnalisés** pour gérer des cas complexes (ex: retours, transferts entre entrepôts).
*   **Remplacer le service de vérification de disponibilité** (`StockAvailabilityChecker`) par votre propre logique.
*   **Utiliser les événements** déclenchés par `commerce_stock` pour synchroniser les niveaux de stock avec un système externe (ERP) à chaque mise à jour.

Par exemple, pour obtenir le niveau de stock d'une variation en code :
```php
$variation_storage = \Drupal::entityTypeManager()->getStorage('commerce_product_variation');
$variation = $variation_storage->load(123); // ID de la variation
$stock_service = \Drupal::service('commerce_stock.service_manager');
$stock_level = $stock_service->getStockLevel($variation);
```

## 4. Informations Utiles

*   **Le panier est une commande** : Ne cherchez pas une entité "panier". Toute la logique de panier se fait sur une entité `commerce_order` avec le statut `draft`. Le service `CartProvider` (`commerce_order.cart_provider`) vous aide à récupérer ou créer le panier pour l'utilisateur courant.
*   **Injecteur de prix de variation** : Le prix affiché sur la page produit est injecté via un service (`commerce_product.chain_variation_price_resolver`). Vous pouvez créer votre propre `resolver` pour afficher des prix complexes (ex: "à partir de X €").
*   **Utilisez Xdebug** : L'écosystème est vaste. Pour comprendre comment les données circulent, notamment lors du recalcul d'une commande par les `OrderProcessor`, l'utilisation d'un débogueur pas à pas est quasi indispensable.
*   **Consultez la documentation officielle** et les exemples dans le module `commerce_examples` pour des cas d'utilisation concrets.

## 5. Écosystème de Modules Contribués Essentiels

L'installation de base de Drupal Commerce fournit le cadre, mais sa véritable puissance réside dans son écosystème de modules contribués. Voici une liste de modules non-fournis par défaut qui sont souvent indispensables pour construire une boutique complète.

*   **Commerce Shipping (`commerce_shipping`)**
    *   **Rôle** : Fournit une infrastructure complète pour la gestion des livraisons. Il permet de définir différentes méthodes de livraison (ex: Colissimo, Point Relais) et de calculer les frais de port en fonction de règles complexes (poids, destination, montant de la commande).

*   **Commerce Recurring (`commerce_recurring`)**
    *   **Rôle** : Permet de mettre en place des abonnements et des paiements récurrents. Il gère la planification des paiements, les renouvellements et les cycles de facturation. Indispensable pour vendre des services par abonnement.

*   **Commerce License (`commerce_license`)**
    *   **Rôle** : Spécialisé dans la vente de produits immatériels nécessitant une clé ou une licence (logiciels, accès à une API, etc.). Il peut générer, assigner et gérer le cycle de vie de ces licences.

*   **Commerce File (`commerce_file`)**
    *   **Rôle** : Facilite la vente de fichiers numériques (e-books, photos, musique). Il sécurise l'accès aux fichiers en ne fournissant un lien de téléchargement limité dans le temps qu'après le paiement de la commande.

*   **Commerce Reports (`commerce_reports`)**
    *   **Rôle** : Offre une interface de reporting pour analyser les ventes. Il fournit des rapports par défaut (ventes par jour, par mois, meilleurs produits) et peut être étendu pour des analyses plus poussées.

*   **Commerce Wishlist (`commerce_wishlist`)**
    *   **Rôle** : Permet aux clients (connectés ou anonymes) de créer et gérer des listes de souhaits. Une fonctionnalité standard sur la plupart des sites e-commerce.

*   **Passerelles de paiement spécifiques (ex: `commerce_stripe`, `commerce_paypal`)**
    *   **Rôle** : Alors que `commerce_payment` fournit le cadre, chaque intégration avec un fournisseur de paiement (Stripe, PayPal, Braintree, etc.) est un module contribué à part entière. Vous devez installer le module correspondant à la passerelle que vous souhaitez utiliser.
