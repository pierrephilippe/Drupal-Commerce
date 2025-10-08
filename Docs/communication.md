# Gestion de la Communication Client dans Drupal Commerce

Ce document explique comment Drupal Commerce gère les communications avec les clients, notamment l'envoi d'e-mails et la génération de documents comme les factures.

## 1. Gestion des E-mails

Drupal Commerce envoie des e-mails à des moments clés du cycle de vie d'une commande (confirmation de commande, envoi, etc.).

### a. Comment ça marche ?

Le système d'e-mails de Commerce repose sur l'API Mail du cœur de Drupal.

1.  **Déclenchement** : Un événement se produit (ex: un client finalise sa commande).
2.  **Service Mail** : Le code de Commerce fait appel au service `plugin.manager.mail` de Drupal pour envoyer un e-mail d'un type spécifique (par exemple, `order_receipt`).
3.  **Contenu** : Le contenu de l'e-mail est généré en utilisant le système de rendu de Drupal. Pour l'e-mail de confirmation de commande, le contenu est en fait le rendu d'une **Vue** (`view`) nommée "Reçu de la commande" (`commerce_order_receipt`).
4.  **Envoi** : Le service d'envoi (par défaut, celui de PHP) expédie l'e-mail.

Le contenu de l'e-mail par défaut est donc une page HTML simple, affichant les informations de la commande.

### b. Comment améliorer le rendu des e-mails ?

Les e-mails par défaut sont fonctionnels mais visuellement très basiques. Pour un rendu professionnel et compatible avec les clients mail modernes (Gmail, Outlook...), il est indispensable d'utiliser des modules spécialisés.

**La solution moderne et recommandée : `Symfony Mailer`**

Le module `Symfony Mailer` (`drupal/symfony_mailer`) remplace le système de mail de Drupal par le puissant composant Mailer de Symfony.

1.  **Installation** : Installez le module via Composer.
2.  **Configuration** : Configurez un "transporteur" (ex: SMTP, SendGrid, Mailgun) pour un envoi fiable.
3.  **Politique d'e-mail** : Le module fournit une interface pour créer des "politiques d'e-mail". Vous pouvez cibler un type d'e-mail spécifique (ex: `order_receipt` de Commerce) et lui appliquer un template HTML personnalisé.
4.  **Templating avec Twig** : Vous pouvez créer des fichiers de template Twig spécifiques pour vos e-mails, avec une structure HTML complète (header, footer, styles).
5.  **CSS Inlining** : `Symfony Mailer` peut automatiquement "inligner" vos styles CSS. C'est une étape cruciale, car la plupart des clients de messagerie ignorent les balises `<style>` et ne lisent que les styles en ligne (`style="..."`) sur les éléments HTML.

En utilisant ce module, vous pouvez créer des e-mails transactionnels riches, "responsive" et à l'image de votre marque.

**Autres méthodes (moins complètes) :**

*   **Surcharger le template Twig** : Vous pouvez copier le fichier `commerce_order_receipt.html.twig` depuis `modules/contrib/commerce/modules/order/templates/` dans votre thème pour en modifier la structure. Cependant, cela ne résout pas les problèmes de CSS inlining et de compatibilité.
*   **Modifier la Vue** : Comme le contenu est une Vue, vous pouvez la modifier via l'interface (`/admin/structure/views/view/commerce_order_receipt`) pour ajouter/supprimer des champs. C'est utile pour changer les données, mais pas le design global.

---

## 2. Génération de Documents (Factures, Devis...)

Par défaut, Drupal Commerce ne propose pas de système de facturation ou de devis à proprement parler. Le seul document généré est le "Reçu de la commande" (`/order/{id}/receipt`), qui est une simple page HTML.

### a. Existe-t-il une version PDF par défaut ?

**Non.** Le cœur de Drupal Commerce ne contient aucune fonctionnalité de génération de PDF. Le reçu de la commande est une page web que l'utilisateur peut imprimer via son navigateur (qui propose souvent une option "Imprimer en PDF"), mais ce n'est pas une solution automatisée ni professionnelle.

### b. Comment générer des factures en PDF ?

Pour cela, il faut utiliser des modules contribués. La solution la plus robuste et la plus utilisée par la communauté est une combinaison de deux modules :

1.  **Commerce Invoice (`drupal/commerce_invoice`)**
2.  **Entity Print (`drupal/entity_print`)**

#### Étape 1 : Le module `Commerce Invoice`

*   **Ce qu'il fait** : Ce module fournit la logique métier de la facturation.
    *   Il crée un nouveau type d'entité : **`commerce_invoice`**.
    *   Il permet de générer des factures (manuellement ou automatiquement) à partir d'une commande.
    *   Il gère la numérotation des factures via le module `Commerce Number Pattern`.
    *   Il fournit une page HTML pour chaque facture, qui peut être thématisée comme n'importe quelle autre page Drupal.

Ce module fournit le **"QUOI"** (les données de la facture), mais pas le **"COMMENT"** (la conversion en PDF).

#### Étape 2 : Le module `Entity Print`

*   **Ce qu'il fait** : Ce module est un utilitaire générique qui permet d'exporter n'importe quel type d'entité Drupal en PDF.
    *   Il fournit une route générique (`/print/pdf/{entity_type}/{entity_id}`) pour déclencher le téléchargement du PDF.
    *   Il s'intègre avec des bibliothèques de conversion HTML vers PDF. La plus courante est **`dompdf/dompdf`**.

Ce module fournit le **"COMMENT"**, mais il a besoin d'un moteur pour faire la conversion.

#### Étape 3 : Le moteur de PDF (ex: Dompdf)

*   **Ce qu'il fait** : C'est une bibliothèque PHP (installée via Composer) qui prend du code HTML et CSS en entrée et génère un fichier PDF en sortie.

### Résumé du processus de génération de PDF

1.  **Installation** :
    ```bash
    composer require drupal/commerce_invoice drupal/entity_print dompdf/dompdf
    ```
    Activez les modules `commerce_invoice` et `entity_print`.

2.  **Configuration** :
    *   Dans les paramètres de `Entity Print` (`/admin/config/content/entity_print`), sélectionnez **Dompdf** comme moteur de PDF.
    *   Dans les paramètres de `Commerce Invoice` (`/admin/commerce/config/invoice-types`), configurez votre type de facture pour qu'il envoie la facture par e-mail lors de la validation de la commande, et activez la génération de PDF.

3.  **Fonctionnement** :
    *   Quand une commande est passée, `Commerce Invoice` crée une entité `commerce_invoice`.
    *   Un lien "Télécharger le PDF" apparaît sur la page de la commande et de la facture.
    *   Quand on clique sur ce lien, `Entity Print` est appelé.
    *   `Entity Print` rend la page HTML de la facture en utilisant un mode de vue spécifique ("Print").
    *   Il passe ce HTML à la bibliothèque `Dompdf`.
    *   `Dompdf` convertit le HTML en PDF et le renvoie au navigateur pour téléchargement.

### c. Personnaliser l'apparence de la facture PDF

La facture PDF est générée à partir d'un template Twig. Pour la personnaliser :

1.  `Entity Print` utilise un template de base : `entity-print.html.twig`.
2.  Vous pouvez le surcharger dans votre thème avec un nom plus spécifique, comme **`entity-print--commerce-invoice.html.twig`**.
3.  Copiez le fichier de base depuis `modules/contrib/entity_print/templates/` dans le dossier `templates` de votre thème, renommez-le et modifiez sa structure HTML et ses styles CSS directement dans la balise `<style>`. `Dompdf` a ses propres contraintes CSS, il faut donc privilégier des mises en page simples (tables, etc.).

Cette combinaison de modules offre une solution complète et flexible pour la génération de factures PDF professionnelles dans Drupal Commerce.
