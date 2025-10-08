# Les Entités de Drupal Commerce

L'architecture de Drupal Commerce est entièrement construite sur l'API d'entité de Drupal. Cela signifie que presque chaque concept (un produit, une commande, une promotion...) est une entité. Cette approche rend le système extrêmement flexible : chaque entité peut avoir des champs, être affichée via des modes de vue, listée avec Views, et manipulée via l'API standard de Drupal.

On distingue deux grands types d'entités : les **entités de contenu** (données créées par les utilisateurs ou les clients) et les **entités de configuration** (définissent le comportement et la structure des entités de contenu).

## 1. Entités de Contenu

Ce sont les entités qui stockent les données de la boutique au quotidien.

*   **Store (`commerce_store`)**
    *   **Module** : `commerce_store`
    *   **Description** : Représente une boutique. Dans une configuration multi-boutiques, chaque `Store` a sa propre monnaie, son pays, son adresse, etc. Une commande est toujours associée à un `Store`.

*   **Product (`commerce_product`)**
    *   **Module** : `commerce_product`
    *   **Description** : Le produit au sens marketing. C'est un conteneur qui regroupe des informations communes (titre, description, images...) pour plusieurs versions d'un même article. Il n'a ni prix, ni SKU et n'est pas directement achetable.

*   **Product Variation (`commerce_product_variation`)**
    *   **Module** : `commerce_product`
    *   **Description** : L'entité réellement achetable. C'est la variation qui porte le SKU (référence unique), le prix et les attributs qui la distinguent (ex: "T-shirt, Taille L, Couleur Bleu"). C'est cette entité qui est ajoutée au panier.

*   **Product Attribute Value (`commerce_product_attribute_value`)**
    *   **Module** : `commerce_product`
    *   **Description** : Représente une valeur spécifique pour un attribut de produit. Par exemple, "Bleu", "Rouge", "S", "M", "L" sont des entités de valeur d'attribut.

*   **Order (`commerce_order`)**
    *   **Module** : `commerce_order`
    *   **Description** : La commande. Elle centralise toutes les informations d'une transaction : les articles, les informations client, les adresses, les promotions appliquées, les paiements, etc. **Le panier est une commande avec le statut `draft`**.

*   **Order Item (`commerce_order_item`)**
    *   **Module** : `commerce_order`
    *   **Description** : Une ligne dans une commande. Elle fait le lien entre une commande et une entité achetable (une `Product Variation`), et stocke la quantité ainsi que le prix unitaire au moment de l'ajout.

*   **Profile (`profile`)**
    *   **Module** : `profile` (dépendance)
    *   **Description** : Stocke un ensemble d'informations sur un utilisateur. Commerce l'utilise principalement pour les adresses (`customer` profile). Cela permet à un client d'avoir plusieurs adresses (facturation, livraison) et de les gérer dans un carnet d'adresses.

*   **Promotion (`commerce_promotion`)**
    *   **Module** : `commerce_promotion`
    *   **Description** : Définit une promotion, ses conditions (ex: "plus de 50€ d'achat") et son offre (ex: "10% de réduction").

*   **Coupon (`commerce_promotion_coupon`)**
    *   **Module** : `commerce_promotion`
    *   **Description** : Représente un code de coupon. Chaque coupon est lié à une promotion et peut avoir ses propres limites d'utilisation.

*   **Payment (`commerce_payment`)**
    *   **Module** : `commerce_payment`
    *   **Description** : Représente une transaction financière, qu'il s'agisse d'un paiement ou d'un remboursement. Elle est liée à une commande et à une passerelle de paiement.

*   **Payment Method (`commerce_payment_method`)**
    *   **Module** : `commerce_payment`
    *   **Description** : Représente une méthode de paiement réutilisable enregistrée par un client (par exemple, une carte de crédit sauvegardée via Stripe). Cela permet au client de ne pas avoir à saisir ses informations à chaque achat.

*   **Log (`log`)**
    *   **Module** : `commerce_log`
    *   **Description** : Entité de journalisation utilisée en interne par Commerce pour tracer les événements importants liés à d'autres entités (par exemple, les changements de statut d'une commande, l'utilisation d'une promotion, etc.).

## 2. Entités de Configuration

Ces entités sont gérées par les administrateurs du site pour configurer le fonctionnement de la boutique. Elles sont généralement exportables dans la configuration du site (`config sync`).

*   **Store Type (`commerce_store_type`)**
    *   **Module** : `commerce_store`
    *   **Description** : Le "bundle" pour les entités `Store`. Permet de définir différents types de boutiques si nécessaire. Par défaut, un seul type est utilisé.

*   **Product Type (`commerce_product_type`)**
    *   **Module** : `commerce_product`
    *   **Description** : Le "bundle" pour les entités `Product`. Permet de créer différents types de produits (ex: "Vêtement", "Livre", "Abonnement") qui peuvent avoir des champs différents et être associés à des types de variations différents.

*   **Product Variation Type (`commerce_product_variation_type`)**
    *   **Module** : `commerce_product`
    *   **Description** : Le "bundle" pour les entités `Product Variation`. C'est ici que l'on définit les champs spécifiques à une variation, comme le prix ou les attributs.

*   **Product Attribute (`commerce_product_attribute`)**
    *   **Module** : `commerce_product`
    *   **Description** : Définit un attribut de produit, comme "Couleur" ou "Taille". C'est une entité de configuration qui sert de conteneur pour les `Product Attribute Value`.

*   **Order Type (`commerce_order_type`)**
    *   **Module** : `commerce_order`
    *   **Description** : Le "bundle" pour les entités `Order`. Permet de définir différents flux de commande. Par défaut, le type "default" est utilisé pour le panier et les commandes.

*   **Order Item Type (`commerce_order_item_type`)**
    *   **Module** : `commerce_order`
    *   **Description** : Le "bundle" pour les entités `Order Item`. Permet de définir différents types de lignes de commande, bien que le type par défaut soit suffisant dans la plupart des cas.

*   **Profile Type (`profile_type`)**
    *   **Module** : `profile` (dépendance)
    *   **Description** : Le "bundle" pour les entités `Profile`. Commerce en crée un type "customer" par défaut pour stocker les adresses.

*   **Payment Gateway (`commerce_payment_gateway`)**
    *   **Module** : `commerce_payment`
    *   **Description** : Configure une méthode de paiement (ex: "Paiement par Stripe", "Virement Bancaire"). C'est ici que l'on stocke les clés d'API, le mode (test/production), et que l'on choisit le plugin qui contient la logique de paiement.

*   **Number Pattern (`number_pattern`)**
    *   **Module** : `commerce_number_pattern`
    *   **Description** : Définit des modèles pour générer des numéros séquentiels ou basés sur des jetons. Commerce l'utilise pour générer les numéros de commande, mais il peut aussi servir pour les factures ou les SKU.

*   **Log Category (`log_category`)**
    *   **Module** : `commerce_log`
    *   **Description** : Définit des catégories pour les entrées de journal (`Log`). Cela permet de structurer et de filtrer les journaux par type d'événement.

*   **Tax Type (`commerce_tax_type`)**
    *   **Module** : `commerce_tax`
    *   **Description** : Définit un type de taxe (ex: "TVA Européenne", "Taxe de vente québécoise"). Chaque type de taxe est associé à un plugin qui définit comment elle est calculée, et contient les différents taux applicables par zone géographique.

## 3. Entités des Modules Contribués Courants

L'écosystème de Commerce est riche et de nombreux modules contribués ajoutent leurs propres entités pour étendre les fonctionnalités de base.

### Commerce Shipping

*   **Shipment (`commerce_shipment`)**
    *   **Module** : `commerce_shipping`
    *   **Description** : Entité de contenu qui représente un envoi. Elle est liée à une commande, contient les articles à expédier, la méthode de livraison choisie et les frais de port associés sous forme d'ajustement.

*   **Shipping Method (`commerce_shipping_method`)**
    *   **Module** : `commerce_shipping`
    *   **Description** : Entité de configuration qui définit une méthode de livraison (ex: "Colissimo Domicile", "Retrait en magasin"). Elle est basée sur des plugins pour le calcul des tarifs.

### Commerce Recurring

*   **Subscription (`commerce_subscription`)**
    *   **Module** : `commerce_recurring`
    *   **Description** : Entité de contenu qui représente un abonnement. Elle lie un client, une méthode de paiement, et une variation de produit "abonnable". Elle gère le cycle de vie de l'abonnement (actif, annulé, expiré) et planifie les renouvellements de commande.

*   **Billing Schedule (`commerce_billing_schedule`)**
    *   **Module** : `commerce_recurring`
    *   **Description** : Entité de configuration qui définit la fréquence de facturation d'un abonnement (ex: "Tous les mois", "Tous les ans le 1er janvier").

### Commerce Wishlist

*   **Wishlist (`commerce_wishlist`)**
    *   **Module** : `commerce_wishlist`
    *   **Description** : Entité de contenu qui représente la liste de souhaits d'un client.

*   **Wishlist Item (`commerce_wishlist_item`)**
    *   **Module** : `commerce_wishlist`
    *   **Description** : Entité de contenu qui représente un article dans une liste de souhaits.

### Commerce License

*   **License (`commerce_license`)**
    *   **Module** : `commerce_license`
    *   **Description** : Entité de contenu conçue pour la vente de produits immatériels. Elle représente une licence générée lors d'un achat (ex: une clé de logiciel, un accès limité dans le temps) et gère son cycle de vie.
