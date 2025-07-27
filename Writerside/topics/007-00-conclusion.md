# Conclusion Générale du Cours

Félicitations ! Vous êtes allé au bout de cette formation intensive sur le BDD. Ce n'est pas un sujet facile, car il
mêle des compétences techniques pointues à une approche philosophique de la conception logicielle.

En partant du simple constat d'un "fossé de communication" entre les métiers, vous avez découvert comment le BDD, avec
son **Langage Ubiquitaire** et ses **Spécifications par l'Exemple**, vient combler ce vide.

Vous avez ensuite mis les mains dans le code pour :

* Configurer un projet Spring Boot avec l'écosystème Cucumber.
* Écrire des spécifications claires et expressives avec Gherkin.
* Transformer ces spécifications en **Documentation Vivante** grâce à des tests automatisés robustes.
* Maîtriser les outils essentiels comme `MockMvc` pour tester vos API, et `Mockito` avec `@MockBean` pour isoler et
  tester votre logique métier en profondeur.
* Structurer vos tests de manière professionnelle avec des techniques comme le `TestContext` pour gérer l'état.

Plus important encore, vous avez appris à penser différemment : non plus "comment vais-je tester ce code ?", mais **"
quel comportement mon code doit-il avoir pour répondre au besoin ?"**. C'est ce changement de perspective qui fait toute
la différence et qui vous positionne en tant que véritable **Concepteur Développeur d'Application**.

## **Correction de toutes les auto-évaluations** {collapsible='true'}

**Chapitre 1 - L'essentiel**

1. c) Faciliter la collaboration et la communication au sein de l'équipe projet.
2. b) Le Product Owner (Métier), le Développeur, le Testeur (QA).
3. b) `When`
4. La "Documentation Vivante" est une spécification (en Gherkin) qui est aussi un test automatisé. Elle est puissante
   car elle ne peut jamais être obsolète : si le code change et ne respecte plus la spécification, le test échoue,
   forçant une mise à jour soit du code, soit de la spec. C'est une source de vérité unique et toujours à jour.
5. Le "Langage Ubiquitaire" est crucial, car il élimine les ambiguïtés entre les différents acteurs d'un projet. En
   utilisant les mêmes termes partout (discutions, code, tests, base de données), on évite les erreurs de "traduction".
   Exemple : si le métier parle de "Contrat Privilège" et que le code utilise une classe `GoldCustomer`, une nouvelle
   personne arrivant sur le projet pourrait ne pas faire le lien, créant confusion et bugs potentiels.

**Chapitre 1 - Pour aller plus loin**

1. b) Quand plusieurs scénarios dans le même fichier partagent les mêmes étapes `Given`.
2. c) Il évite la duplication de la logique de scénario pour tester différents jeux de données.
3. b) `When je soumets le formulaire d'inscription`
4. Le `Background` s'exécute **avant chaque scénario** du fichier pour mettre en place un contexte commun. Le premier
   `Given` d'un scénario, lui, est spécifique à ce scénario uniquement. Le `Background` sert à factoriser les prérequis,
   tandis que le `Given` d'un scénario ajoute des conditions spécifiques à ce cas de test précis.
5. Un `Data Table` est idéal lorsque l'on doit passer une liste d'objets ou un formulaire complexe. Par exemple, pour
   initialiser une bibliothèque avec 5 livres (`Given`), il est beaucoup plus lisible de fournir un tableau de 5 lignes
   que d'utiliser un `Scenario Outline` avec une variable qui ne contiendrait qu'une seule colonne de titres. De même,
   pour décrire le contenu d'un panier (`Then`), un tableau est plus structuré et lisible.

**Chapitre 2 - L'essentiel**

1. d) `src/test/resources`
2. b) `cucumber-spring`
3. c) C'est le point d'entrée qui permet à JUnit 5 de trouver et d'exécuter les tests Cucumber.
4. Obtenir "undefined" est une étape positive car cela prouve que toute la configuration (runner, dépendances, structure
   des répertoires) fonctionne. Cucumber lit bien le fichier `.feature`, mais ne trouve simplement pas le code Java
   correspondant. Il nous aide même en générant les squelettes des méthodes à implémenter. C'est la confirmation que
   la "plomberie" est bonne avant de commencer à coder la logique.
5. Si `cucumber-junit-platform-engine` est manquant, JUnit 5 ne saura pas comment interpréter l'annotation `@Cucumber`.
   Le plus probable est que l'IDE ou Maven signalera qu'**aucun test n'a été trouvé** ou exécuté, car le pont entre
   JUnit et Cucumber n'existe pas.

**Chapitre 3 - L'essentiel**

1. c) `@AutoConfigureMockMvc`
2. b) Rouge (Failed)
3. c) À simuler des requêtes HTTP vers les contrôleurs REST sans démarrer un vrai serveur web.
4. Passer de "undefined" à "rouge" est crucial car cela signifie que le test est maintenant "connecté" au code. Le test
   exécute une action réelle (un appel API), et il échoue parce que le code de production qui devrait répondre à cette
   action est manquant. C'est le signal qui guide le développeur sur ce qu'il doit coder ensuite.
5. `status().isOk()` vérifie le **méta-état** de la réponse (le statut HTTP, ici 200), confirmant que la communication a
   réussi. `jsonPath("$.title").value("Dune")` vérifie le **contenu métier** de la réponse (le corps JSON), confirmant
   que les bonnes données ont été retournées. On a besoin des deux pour s'assurer que l'appel a non seulement réussi
   techniquement, mais qu'il a aussi produit le résultat métier attendu.

**Chapitre 4 - L'essentiel**

1. b) Il garantit qu'une nouvelle instance d'un bean est créée pour chaque scénario.
2. c) Utiliser un `Scenario Outline` avec 5 lignes dans le tableau `Examples`.
3. c) Une dépendance injectée comme `MockMvc` ou `BookRepository`. Celles-ci sont gérées par Spring et injectées une
   fois, elles ne font pas partie de l'état volatile d'un scénario.
4. Un `TestContext` est supérieur car il isole l'état de chaque scénario, évitant les fuites de données entre les tests.
   Dans un projet qui grandit, les "Step Definitions" peuvent être réparties dans plusieurs classes. Le `TestContext`
   est un objet unique et partagé qui peut être injecté partout, agissant comme un conteneur de données propre et clair,
   alors que des variables d'instance seraient privées à chaque classe de steps, rendant le partage d'état impossible ou
   très complexe.
5. Grâce à la dépendance `cucumber-java`. Quand une méthode de step a un argument de type `DataTable`, Cucumber
   reconnaît automatiquement cette signature. Il parse alors le tableau Gherkin qui suit l'étape et le transforme en un
   objet `DataTable` qu'il passe à la méthode.

**Chapitre 5 - L'essentiel**

1. b) Quand on veut tester la logique d'un composant (ex: un Service) en l'isolant de ses dépendances (ex: un
   Repository).
2. c) `@DataJpaTest`
3. b) En mémoire, dans un objet créé par Mockito pour la durée du test.
4. Quand la logique métier est complexe et indépendante du protocole de communication. Par exemple, pour un service de
   calcul de taxes avec de nombreuses règles, il est plus simple et rapide de le tester directement en lui passant des
   objets Java, plutôt que de devoir construire des requêtes HTTP et parser des réponses JSON à chaque fois.
5. `@SpringBootTest` charge le contexte d'application Spring quasi-complet (couche web, services, persistance, etc.), ce
   qui est lourd. `@DataJpaTest` ne charge que ce qui est nécessaire pour JPA (la configuration de la source de données,
   les repositories, les entités). L'impact est significatif : les tests `@DataJpaTest` sont beaucoup plus rapides à
   démarrer et à exécuter que les tests `@SpringBootTest`.

**Chapitre 6 - Pour aller plus loin**

1. c) Une large base de tests unitaires rapides, et moins de tests d'intégration/BDD plus lents.
2. b) `Then le produit est ajouté à mon panier.`
3. c) Un échec dans un scénario peut provoquer des échecs en cascade dans les suivants, rendant le débogage très
   difficile.
4. La Documentation Vivante est une documentation qui est aussi un test automatisé. Contrairement à un document Word,
   elle ne peut pas être désynchronisée du code. Si le code ne respecte plus la documentation, les tests échouent,
   forçant une action. Elle est donc une source de vérité fiable et toujours à jour, validée par l'exécution continue
   des tests.
5. Le problème est que le test est **fragile** (cassé au moindre changement de l'UI) et **impératif** (décrit le "
   comment" au lieu du "quoi"). L'argument pour convaincre l'équipe serait de montrer qu'un style déclaratif rend les
   tests **plus robustes** (résistants aux refactorings), **plus lisibles** par les non-techniciens (favorisant la
   collaboration avec le métier), et **plus maintenables** (la complexité technique est cachée dans le code des steps,
   pas exposée dans le Gherkin).