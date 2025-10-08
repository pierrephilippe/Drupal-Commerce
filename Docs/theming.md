# Theming dans Drupal Commerce

Ce document explique comment personnaliser l'apparence (le "thème") des différentes parties de votre boutique Drupal Commerce en surchargeant ses fichiers de template Twig.

## 1. Principes de Base et Débogage Twig

Drupal Commerce, comme le reste de Drupal, utilise le moteur de template Twig pour générer le HTML. Pour personnaliser l'affichage, vous n'avez pas besoin de modifier les modules de Commerce directement. À la place, vous copiez les fichiers de template dans votre propre thème et les modifiez là.

### Activer le mode de débogage Twig

C'est l'étape la plus importante. Le mode de débogage vous montrera dans le code source de votre page :
*   Quel template est utilisé pour chaque partie de la page.
*   Des suggestions de noms de fichiers pour créer des surcharges plus spécifiques.
*   L'emplacement du fichier de template original.

Pour l'activer :

1.  Localisez le fichier `sites/development.services.yml`. S'il n'existe pas, copiez `sites/default/default.services.yml` et renommez-le.
2.  Assurez-vous que votre site utilise ce fichier. Dans `sites/default/settings.php`, décommentez la ligne suivante si elle existe :
    ```php
    $settings['container_yamls'][] = DRUPAL_ROOT . '/sites/development.services.yml';
    ```
3.  Dans `development.services.yml`, modifiez les paramètres de Twig comme suit :
    ```yaml
    parameters:
      twig.config:
        debug: true
        auto_reload: true
        cache: false
    ```
4.  Videz les caches de Drupal : `drush cr` ou via l'interface (`/admin/config/development/performance`).

Maintenant, en inspectant le code source de votre page produit, vous verrez des commentaires HTML comme ceci :

```html
<!-- BEGIN OUTPUT from 'core/themes/classy/templates/content/page-title.html.twig' -->
...
<!-- END OUTPUT from 'core/themes/classy/templates/content/page-title.html.twig' -->

<!-- THEME DEBUG -->
<!-- THEME HOOK: 'commerce_product' -->
<!-- FILE NAME SUGGESTIONS:
   * commerce-product--t-shirt--full.html.twig
   * commerce-product--t-shirt.html.twig
   x commerce-product.html.twig
-->
<!-- BEGIN OUTPUT from 'modules/contrib/commerce/modules/product/templates/commerce-product.html.twig' -->
...
<!-- END OUTPUT from 'modules/contrib/commerce/modules/product/templates/commerce-product.html.twig' -->
```

## 2. Surcharger un Template : Le Processus

Le processus est toujours le même :

1.  **Identifier** : Utilisez le débogage Twig pour trouver le nom du template de base (celui avec un `x` à côté) et les suggestions de nommage.
2.  **Localiser** : Le chemin vers le fichier de base est indiqué dans le commentaire de débogage (ex: `modules/contrib/commerce/modules/product/templates/commerce-product.html.twig`).
3.  **Copier** : Copiez ce fichier de template depuis le dossier du module Commerce vers le dossier `templates` de votre thème personnalisé.
    *   **Bonne pratique** : Pour garder votre thème organisé, il est conseillé de recréer l'arborescence du module. Par exemple, copiez le template du produit dans `themes/mon_theme/templates/commerce/product/`.
4.  **Renommer** : Renommez le fichier copié en utilisant l'une des suggestions de nommage pour cibler un affichage spécifique. La suggestion la plus spécifique est prioritaire.
5.  **Modifier** : Ouvrez votre nouveau fichier de template et modifiez le HTML et les variables Twig selon vos besoins.
6.  **Vider le cache** : Videz les caches de Drupal pour qu'il détecte votre nouveau fichier.

## 3. Exemples Concrets

### a. Personnaliser la page produit

Imaginons que nous voulons déplacer les variations de produit (sélecteurs de taille/couleur) au-dessus de la description pour le type de produit "T-shirt".

1.  **Identifier** : Le débogage Twig sur la page produit nous donne :
    *   Template de base : `commerce-product.html.twig`
    *   Suggestion spécifique : `commerce-product--t-shirt.html.twig`

2.  **Localiser** : Le fichier se trouve dans `modules/contrib/commerce/modules/product/templates/commerce-product.html.twig`.

3.  **Copier/Renommer** :
    *   Copiez le fichier original vers `themes/mon_theme/templates/commerce/product/commerce-product--t-shirt.html.twig`.

4.  **Modifier** : Le fichier original ressemble à ceci :
    ```twig
    {# themes/mon_theme/templates/commerce/product/commerce-product--t-shirt.html.twig #}

    {#
      @see https://www.drupal.org/node/3015623
    #}
    {{ attach_library('commerce_product/product') }}

    <article{{ attributes }}>
      {{ product.body }}
      {{ product.variations }}
    </article>
    ```
    Nous inversons simplement les deux lignes :
    ```twig
    {# ... #}
    <article{{ attributes }}>
      {{ product.variations }}
      {{ product.body }}
    </article>
    ```

5.  **Vider le cache** : Après avoir vidé le cache, seules les pages des produits de type "T-shirt" auront ce nouvel agencement.

### b. Modifier le formulaire "Ajouter au panier"

C'est l'un des besoins les plus courants. Par exemple, pour mettre le champ de quantité et le bouton d'ajout sur la même ligne.

1.  **Identifier** : Le template est `commerce-order-item-add-to-cart-form.html.twig`.

2.  **Localiser** : Il se trouve dans `modules/contrib/commerce/modules/cart/templates/commerce-order-item-add-to-cart-form.html.twig`.

3.  **Copier/Renommer** :
    *   Copiez le fichier vers `themes/mon_theme/templates/commerce/cart/commerce-order-item-add-to-cart-form.html.twig`.
    *   Vous pouvez utiliser des suggestions plus spécifiques si nécessaire, par exemple `commerce-order-item-add-to-cart-form--commerce-product--t-shirt.html.twig` pour ne cibler que le formulaire pour les t-shirts.

4.  **Modifier** : Le fichier original contient `{{ form.quantity }}` et `{{ form.actions }}`. Vous pouvez les envelopper dans des `div` et utiliser CSS (Flexbox, Grid) pour les aligner.

    ```twig
    {# themes/mon_theme/templates/commerce/cart/commerce-order-item-add-to-cart-form.html.twig #}

    <div class="add-to-cart-form-wrapper">
      <div class="quantity-wrapper">
        {{ form.quantity }}
      </div>
      <div class="actions-wrapper">
        {{ form.actions }}
      </div>
    </div>

    {# Affichez le reste des champs du formulaire, qui sont cachés. #}
    {{ form|without('quantity', 'actions') }}
    ```
    Vous pouvez ensuite styler `.add-to-cart-form-wrapper` avec `display: flex;` dans le CSS de votre thème.

5.  **Vider le cache** : Le formulaire d'ajout au panier aura maintenant votre nouvelle structure HTML.

## 4. Explorer les Variables Disponibles

Dans n'importe quel fichier de template Twig, vous pouvez voir toutes les variables disponibles en utilisant la fonction `dump()`. Cela nécessite que le module **Devel** et la librairie **Kint** (`drupal/devel_kint_extras`) soient installés.

```twig
{{ dump() }} {# Affiche toutes les variables #}
{{ dump(product) }} {# Affiche uniquement la variable 'product' #}
```

Cela vous aidera à découvrir tous les champs et toutes les données que vous pouvez afficher dans vos templates personnalisés.
