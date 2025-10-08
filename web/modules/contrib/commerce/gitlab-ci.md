# Documentation sur GitLab CI/CD pour Drupal Commerce

Ce document explique le rôle et le fonctionnement de GitLab CI/CD dans le contexte du module Drupal Commerce. Il se base sur le fichier de configuration `.gitlab-ci.yml` qui est la méthode d'intégration continue standard pour les projets hébergés sur `drupal.org`.

## 1. Qu'est-ce que GitLab CI/CD ?

**GitLab CI/CD** est un outil d'intégration, de livraison et de déploiement continus (CI/CD) intégré directement dans la plateforme GitLab. Son rôle est d'automatiser les étapes du cycle de vie d'un projet logiciel, de la validation du code à son déploiement.

Pour un module Drupal comme Commerce, il est principalement utilisé pour l'**Intégration Continue (CI)** :
1.  **Automatiser les tests** : Lancer automatiquement les suites de tests (unitaires, fonctionnels, etc.) à chaque modification du code.
2.  **Vérifier la qualité du code** : S'assurer que le code respecte les standards de codage de Drupal.
3.  **Garantir la non-régression** : Détecter immédiatement si un nouveau changement casse une fonctionnalité existante.

Le tout est orchestré par un fichier de configuration nommé `.gitlab-ci.yml`.

## 2. Comment fonctionne GitLab CI/CD ?

Le processus est centré autour de concepts clés :

*   **Pipeline** : C'est l'ensemble du processus de CI/CD pour un commit donné. Un pipeline est composé de plusieurs *stages* (étapes).
*   **Stage** : Un stage regroupe des *jobs* (tâches) qui peuvent s'exécuter en parallèle. Un stage ne commence que lorsque tous les jobs du stage précédent ont réussi. Exemples de stages : `build`, `test`, `deploy`.
*   **Job** : C'est la tâche atomique. Un job est une série de commandes (un script) qui s'exécute dans un environnement isolé. Par exemple, un job pour l'analyse du code, un autre pour les tests unitaires.
*   **Runner** : C'est une machine (physique ou virtuelle) qui exécute les jobs. Les projets sur `drupal.org` utilisent des runners partagés qui sont mis à disposition par l'infrastructure Drupal.
*   **Image Docker** : Chaque job s'exécute dans un conteneur Docker basé sur une image spécifiée. Cette image contient tous les outils nécessaires (PHP, Composer, Node.js, etc.).

Le flux est le suivant :
1.  Un développeur pousse du code (`git push`) vers le dépôt GitLab.
2.  GitLab détecte la présence du fichier `.gitlab-ci.yml` et démarre un nouveau pipeline.
3.  Il assigne les jobs des premiers stages à des runners disponibles.
4.  Pour chaque job, le runner télécharge l'image Docker spécifiée, clone le code du projet, et exécute les commandes définies dans la section `script` du job.
5.  Le résultat (succès ou échec) est rapporté dans l'interface de GitLab, souvent directement sur la *Merge Request* concernée.

## 3. Analyse du fichier `.gitlab-ci.yml` de Commerce

Le fichier `.gitlab-ci.yml` de Drupal Commerce est un excellent exemple de configuration de tests pour un module Drupal complexe. Il est beaucoup plus explicite et modulaire que l'ancien fichier `.travis.yml`.

Voici une description de sa structure typique :

### `image` et `variables` globales

```yaml
image: drupalci/php-7.3-apache

variables:
  DB_DRIVER: mysql
  # ...
```

*   `image`: Définit l'image Docker par défaut pour tous les jobs. `drupalci/php-x.x-apache` est une image maintenue par la communauté Drupal, contenant PHP, Apache, et les extensions PHP courantes nécessaires.
*   `variables`: Définit des variables d'environnement qui seront disponibles dans tous les jobs.

### `stages`

```yaml
stages:
  - build
  - test
  - quality
```

Le pipeline est divisé en étapes logiques. Les jobs du stage `test` ne commenceront que si ceux du stage `build` ont réussi.

### `before_script`

```yaml
before_script:
  - composer install --prefer-dist --no-progress --no-suggest
  - ./vendor/bin/run-tests.sh ...
```

Cette section définit des commandes qui seront exécutées **avant** le script principal de chaque job. C'est typiquement ici que l'on installe les dépendances du projet et que l'on met en place l'environnement de test Drupal.

### Définition des Jobs

Chaque job est un bloc de configuration qui hérite des paramètres globaux mais peut les surcharger.

**Exemple 1 : Analyse de la qualité du code (`coding-standards`)**

```yaml
coding-standards:
  stage: quality
  script:
    - ./vendor/bin/phpcs --standard=./phpcs.xml.dist --report-full
```

*   Ce job appartient au stage `quality`.
*   Son `script` exécute `PHP CodeSniffer` (`phpcs`) pour vérifier que le code respecte les standards de Drupal. S'il trouve des erreurs, le job échoue.

**Exemple 2 : Tests unitaires (`phpunit-unit`)**

```yaml
phpunit-unit:
  stage: test
  script:
    - ./vendor/bin/phpunit -c ./phpunit.xml.dist --group Unit
```

*   Ce job appartient au stage `test`.
*   Il exécute la suite de tests PHPUnit en ne ciblant que les tests du groupe `Unit`. Ces tests sont très rapides car ils n'ont pas besoin d'une base de données ni d'un site Drupal fonctionnel.

D'autres jobs sont définis sur le même modèle pour les tests `Kernel`, `Functional` et `FunctionalJavascript`, chacun avec ses propres spécificités (par exemple, les tests fonctionnels nécessitent une base de données et une installation complète de Drupal).

## 4. Différences avec Travis CI

La transition de Travis CI vers GitLab CI/CD pour les projets Drupal représente une modernisation significative :

1.  **Intégration vs. Service Tiers** : GitLab CI est une fonctionnalité native de GitLab, là où Travis CI est un service externe qui se connecte à GitHub. L'intégration est donc plus profonde et plus fluide.
2.  **Explicite vs. Abstrait** : La configuration GitLab est plus verbeuse mais aussi plus explicite. Chaque étape (installation, test) est clairement définie dans un script. Travis CI, avec l'aide de `drupal-ti`, masquait une grande partie de cette complexité derrière des commandes simples comme `drupal-ti install`, ce qui rendait le débogage plus difficile.
3.  **Contrôle de l'environnement** : GitLab CI met l'accent sur les conteneurs Docker (`image: ...`). Cela donne un contrôle total et une grande reproductibilité de l'environnement de test, alors que Travis CI gérait des environnements plus "magiques".
4.  **Modularité et Parallélisme** : La structure en `stages` et `jobs` de GitLab permet de définir des pipelines très granulaires et d'optimiser le parallélisme, ce qui peut accélérer considérablement le temps de retour pour les développeurs.
