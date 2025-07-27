# Chapitre 3 : Premier Scénario BDD de A à Z : "Hello, BDD!"

Bienvenue au cœur de l'action ! Dans ce chapitre, nous allons prendre le scénario que nous avons écrit et faire en sorte
qu'il devienne un véritable test automatisé qui valide le comportement de notre application Spring Boot. Nous allons
suivre pas à pas le cycle BDD, qui est une extension du cycle TDD (Red-Green-Refactor).

Ce processus peut sembler un peu long au début, mais il deviendra vite une seconde nature. C'est une danse en plusieurs
temps : écrire la spec, la voir échouer, écrire juste assez de code pour la faire passer.

### Objectifs pédagogiques

À la fin de ce chapitre, vous serez capable de :

* **Comprendre et appliquer** le cycle de développement BDD.
* **Implémenter** des "Step Definitions" en utilisant les snippets générés par Cucumber.
* **Lier** vos classes de test au contexte de l'application Spring Boot avec les bonnes annotations.
* **Utiliser `MockMvc`** pour simuler des appels HTTP vers vos contrôleurs REST sans démarrer de serveur.
* **Écrire des assertions** avec `AssertJ` pour vérifier les résultats des appels API.
* **Faire passer un scénario** de l'état "undefined" à "failed" (rouge), puis à "passed" (vert).

### Introduction : La Danse du BDD

Imaginez un chorégraphe et un danseur. Le chorégraphe (le fichier `.feature`) décrit une série de mouvements : "Fais un
pas en avant", "Tourne sur toi-même", "Lève le bras". Au début, le danseur (notre code de production) ne sait pas
comment exécuter ces mouvements.

Notre travail, en tant que développeurs, est d'être l'interprète. Nous allons d'abord apprendre à reconnaître chaque
instruction (implémenter les Step Definitions). Puis, pour chaque instruction, nous allons enseigner au danseur comment
l'exécuter.

Le cycle BDD est cette danse :

1. **Écrire le scénario** (la chorégraphie).
2. **Exécuter et obtenir "undefined"** (le danseur ne connaît pas les pas).
3. **Implémenter les Step Definitions** (on associe une phrase à une méthode Java).
4. **Exécuter et obtenir "Failed" (Rouge)** (le danseur connaît le pas, mais l'exécution échoue car le code de l'
   application est manquant).
5. **Écrire le code de production** (on apprend au danseur à bien faire le mouvement).
6. **Exécuter et obtenir "Passed" (Vert)** (la danse est parfaite !).

C'est ce que nous allons faire, maintenant.

### 1. Le Scénario Cible

Reprenons le scénario de notre exercice précédent. Il est simple, clair et parfait pour un premier essai.

**Fichier `src/test/resources/features/book_search.feature`**

```gherkin
Feature: Recherche de livre par ISBN
  En tant qu'utilisateur, je veux rechercher un livre par son ISBN
  pour obtenir ses détails complets.

  Scenario: Recherche d'un livre existant avec succès
    Given un livre existe avec l'ISBN "978-0441013593"
    When l'utilisateur recherche le livre par cet ISBN
    Then les détails du livre "Dune" sont retournés
```

### 2. Étape 1 : Implémenter les Step Definitions (passer de "undefined" à "rouge")

Dans le chapitre précédent, Cucumber nous a gentiment fourni les "snippets" de code. Notre première tâche est de les
intégrer dans notre classe `BookStepDefinitions`.

<procedure title="Intégrer les snippets et lier à Spring">

1. Ouvrez la classe `BookStepDefinitions.java` (`src/test/java/.../bdd/BookStepDefinitions.java`).
2. Copiez-collez les snippets générés par Cucumber dans cette classe.
3. **Liez la classe au contexte Spring.** C'est l'étape la plus importante ! Nous devons dire à Cucumber : "Quand tu
   exécutes ces tests, démarre une application Spring Boot en arrière-plan pour que je puisse interagir avec."
    * Ajoutez l'annotation `@SpringBootTest` en haut de la classe.
    * Ajoutez `@AutoConfigureMockMvc` pour pouvoir simuler des appels HTTP.
4. Injectez (`@Autowired`) les dépendances dont nous aurons besoin : `MockMvc` pour faire les appels, et
   `BookRepository` pour le `Given`.
5. Stockez les données partagées entre les étapes (`Given`, `When`, `Then`) dans des variables d'instance de la classe.

</procedure>

Voici à quoi ressemble la classe `BookStepDefinitions.java` après ces modifications :

```java
package fr.formation.spring.librarymanagement.bdd;

import fr.formation.spring.librarymanagement.domain.Book;
import fr.formation.spring.librarymanagement.repository.BookRepository;
import io.cucumber.java.en.Given;
import io.cucumber.java.en.Then;
import io.cucumber.java.en.When;
import io.cucumber.spring.CucumberContextConfiguration;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.web.servlet.AutoConfigureMockMvc;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.web.servlet.MockMvc;
import org.springframework.test.web.servlet.ResultActions;

import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.get;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.status;

// Ces annotations sont la clé de l'intégration Spring/Cucumber
@SpringBootTest
@AutoConfigureMockMvc
@CucumberContextConfiguration
public class BookStepDefinitions {

    // MockMvc permet de simuler des requêtes HTTP sur nos contrôleurs
    @Autowired
    private MockMvc mockMvc;

    // Le repository pour préparer l'état de la base de données
    @Autowired
    private BookRepository bookRepository;

    // Variable pour stocker le résultat de l'action (When)
    // et le rendre disponible pour la vérification (Then)
    private ResultActions resultActions;

    @Given("un livre existe avec l'ISBN {string}")
    public void un_livre_existe_avec_l_isbn(String isbn) {
        // Pour ce test, nous nous assurons que les données
        // initialisées par DataInitializer sont bien présentes.
        // Dans un test plus complexe, nous pourrions créer le livre ici.
        assert bookRepository.findById(isbn).isPresent();
    }

    @When("l'utilisateur recherche le livre par cet ISBN")
    public void l_utilisateur_recherche_le_livre_par_cet_isbn() throws Exception {
        // Le endpoint n'existe pas encore, donc cet appel va échouer.
        // C'est l'étape "ROUGE" du cycle TDD/BDD.
        resultActions = mockMvc.perform(get("/api/books/978-0441013593"));
    }

    @Then("les détails du livre {string} sont retournés")
    public void les_details_du_livre_sont_retournes(String title) throws Exception {
        // Pour l'instant, nous nous contentons de vérifier
        // que la requête a réussi (statut 200 OK).
        // Cette assertion va échouer car l'appel précédent a échoué.
        resultActions.andExpect(status().isOk());
    }
}
```

**Exécutez `CucumberTestRunner` maintenant.**
Le résultat change ! Fini le "undefined". Vous devriez voir une erreur claire, probablement un `404 Not Found`.

```
java.lang.AssertionError: Status expected:<200> but was:<404>
```

**Bravo ! Vous êtes dans le ROUGE.** Cela signifie que votre test est bien écrit, il exécute un appel API, mais
l'application ne sait pas encore comment y répondre.

### 3. Étape 2 : Écrire le code de production (passer de "rouge" à "vert")

Notre test nous dit exactement ce qui manque : un endpoint `GET /api/books/{isbn}`. Notre mission est maintenant de
l'implémenter. Nous allons créer un `RestController` pour exposer cette fonctionnalité.

<procedure title="Créer le contrôleur REST">
<p>Créez un nouveau package <code>fr.formation.spring.librarymanagement.controller</code> et ajoutez-y la classe <code>BookController</code>.</p>

```java
package fr.formation.spring.librarymanagement.controller;

import fr.formation.spring.librarymanagement.domain.Book;
import fr.formation.spring.librarymanagement.repository.BookRepository;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

// Contrôleur REST pour la gestion des livres
@RestController
@RequestMapping("/api/books")
public class BookController {

    private final BookRepository bookRepository;

    public BookController(BookRepository bookRepository) {
        this.bookRepository = bookRepository;
    }

    // Endpoint pour récupérer un livre par son ISBN
    @GetMapping("/{isbn}")
    public ResponseEntity<Book> getBookByIsbn(@PathVariable String isbn) {
        // On cherche le livre dans le repository.
        // Si on le trouve, on retourne 200 OK avec le livre.
        // Sinon, on retourne 404 Not Found.
        return bookRepository.findById(isbn)
                .map(ResponseEntity::ok) // Équivalent à book -> ResponseEntity.ok(book)
                .orElse(ResponseEntity.notFound().build());
    }
}
```

</procedure>

Ce contrôleur est très simple : il reçoit une requête sur `/{isbn}`, utilise le `BookRepository` pour trouver le livre
correspondant en base de données, et retourne soit le livre avec un statut `200 OK`, soit une réponse `404 Not Found`.

### 4. Étape 3 : Valider le succès et affiner le test

**Relancez `CucumberTestRunner` !**

Magie ! Le test devrait maintenant passer. La console affiche une sortie verte et rassurante :

```
1 scenario (1 passed)
3 steps (3 passed)
...
```

**Félicitations, vous êtes dans le VERT !** Vous avez complété votre premier cycle BDD.

Maintenant, nous pouvons **affiner** notre test. Juste vérifier le statut 200, c'est bien, mais notre scénario est plus
précis : il veut vérifier que les *détails du livre "Dune"* sont retournés.

Améliorons notre étape `Then` en utilisant `jsonPath` (une fonctionnalité de `MockMvc`) et `AssertJ` pour des assertions
plus lisibles.

<procedure title="Améliorer l'étape `Then`">
<p>Modifiez la méthode <code>les_details_du_livre_sont_retournes</code> dans <code>BookStepDefinitions.java</code>.</p>

```java
// ... importations ...

import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.jsonPath;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.status;

// ... dans la classe BookStepDefinitions ...

@Then("les détails du livre {string} sont retournés")
public void les_details_du_livre_sont_retournes(String title) throws Exception {
    resultActions
            .andExpect(status().isOk()) // Vérifie le statut HTTP 200
            .andExpect(jsonPath("$.title").value(title)) // Vérifie le titre dans le JSON
            .andExpect(jsonPath("$.author").value("Frank Herbert")); // Vérifie l'auteur
}
```

</procedure>

Relancez le test une dernière fois pour confirmer que tout fonctionne toujours. C'est l'étape de **Refactor** : on
améliore la qualité du code (de test ou de production) sans en changer le comportement.

---

### Exercice 4 : Scénario du livre non trouvé

Maintenant, c'est à vous d'appliquer ce cycle pour un nouveau scénario.

**Votre mission :**

1. **Ajoutez** un nouveau scénario à votre fichier `book_search.feature` pour le cas où un livre n'est pas trouvé.
2. **Exécutez** les tests. Vous verrez que Cucumber vous signale qu'un scénario a des étapes "undefined". Copiez les
   nouveaux snippets.
3. **Implémentez** les nouvelles étapes dans `BookStepDefinitions.java`.
4. **Suivez le cycle Rouge-Vert** : votre test doit d'abord échouer (car votre code retourne déjà 404, mais votre test
   ne s'y attend pas encore), puis passer.

**Nouveau scénario à ajouter dans `book_search.feature` :**

```gherkin
  Scenario: Recherche d'un livre qui n'existe pas
    When l'utilisateur recherche le livre avec l'ISBN "000-0000000000"
    Then aucune information de livre n'est retournée
    And le statut de la réponse est "Not Found"
```

**Conseil :** Vous aurez besoin d'une nouvelle méthode `When` (car l'ISBN est différent) et de deux nouvelles méthodes
`Then`. Pour la dernière étape, utilisez `resultActions.andExpect(status().isNotFound());`.

### Correction exercice 4 {collapsible='true'}

Voici les étapes de la solution.

**1. `book_search.feature` mis à jour**

```gherkin
Feature: Recherche de livre par ISBN
  # ...

  Scenario: Recherche d'un livre existant avec succès
    # ...

  Scenario: Recherche d'un livre qui n'existe pas
    When l'utilisateur recherche le livre avec l'ISBN "000-0000000000"
    Then aucune information de livre n'est retournée
    And le statut de la réponse est "Not Found"
```

**2. Ajouts dans `BookStepDefinitions.java`**

En lançant les tests, Cucumber génère les snippets. On les ajoute et on les implémente.

```java
// ... dans la classe BookStepDefinitions ...

// Nouvelle étape WHEN pour un ISBN spécifique
@When("l'utilisateur recherche le livre avec l'ISBN {string}")
public void l_utilisateur_recherche_le_livre_avec_l_isbn(String isbn) throws Exception {
    // Cette étape est paramétrée, elle remplace l'ancienne étape 'When'
    // qui avait l'ISBN en dur. C'est beaucoup mieux !
    resultActions = mockMvc.perform(get("/api/books/" + isbn));
}

// Nouvelle étape THEN
@Then("aucune information de livre n'est retournée")
public void aucune_information_de_livre_n_est_retournee() throws Exception {
    // On vérifie que le corps de la réponse est vide.
    // Pour un 404, c'est généralement le cas.
    resultActions.andExpect(jsonPath("$").doesNotExist());
}

// Nouvelle étape AND (qui est un THEN)
@Then("le statut de la réponse est {string}")
public void le_statut_de_la_reponse_est(String status) throws Exception {
    if ("Not Found".equalsIgnoreCase(status)) {
        resultActions.andExpect(status().isNotFound()); // Vérifie 404
    }
}
```

**Amélioration (Refactoring) :**
Vous remarquerez que nous avons maintenant deux étapes `When` très similaires. On peut les fusionner en une seule étape
paramétrée.

* Supprimez l'ancienne méthode : `l_utilisateur_recherche_le_livre_par_cet_isbn()`.
* Modifiez le premier scénario pour qu'il utilise la nouvelle étape :

```gherkin
  Scenario: Recherche d'un livre existant avec succès
    Given un livre existe avec l'ISBN "978-0441013593"
    When l'utilisateur recherche le livre avec l'ISBN "978-0441013593"
    Then les détails du livre "Dune" sont retournés
```

Maintenant, une seule méthode Java gère les deux `When`, ce qui est plus maintenable. En relançant les tests, les deux
scénarios devraient passer.

---

### Auto-évaluation

1. **Quelle annotation est essentielle pour injecter `MockMvc` dans une classe de test Cucumber ?**
   a) `@SpringBootTest`
   b) `@EnableMockMvc`
   c) `@AutoConfigureMockMvc`
   d) `@Cucumber`

2. **Dans le cycle BDD, quel est l'état attendu juste après avoir implémenté les Step Definitions (avec le code de test)
   mais avant d'avoir écrit le code de production ?**
   a) Vert (Passed)
   b) Rouge (Failed)
   c) Jaune (Undefined)
   d) Bleu (Skipped)

3. **À quoi sert `MockMvc` ?**
   a) À se connecter à une fausse base de données.
   b) À créer des objets "mock" pour les services.
   c) À simuler des requêtes HTTP vers les contrôleurs REST sans démarrer un vrai serveur web.
   d) À générer des données de test.

4. **Expliquez pourquoi le fait de passer de "undefined" à "rouge" (failed) est une étape cruciale et positive.**

5. **Quelle est la différence entre `status().isOk()` et `jsonPath("$.title").value("Dune")` dans une assertion de
   test ? Pourquoi avons-nous besoin des deux ?**

---

### Conclusion : Vous avez donné vie à une spécification !

Félicitations, c'est une étape majeure ! Vous avez parcouru tout le chemin : d'une phrase en langage naturel dans un
fichier `.feature` à un test automatisé complet qui valide le comportement réel de votre code de production.

Vous avez appris à :

* **Traduire** des étapes Gherkin en méthodes Java.
* **Piloter** votre application avec `MockMvc`.
* **Vérifier** les résultats avec précision.
* **Suivre** le cycle Rouge-Vert-Refactor, qui est le cœur du développement agile guidé par les tests.

Vous détenez maintenant la compétence de base la plus importante en BDD. Dans les prochains modules, nous allons nous
appuyer sur cet acquis pour explorer des techniques plus avancées : comment gérer des données plus complexes, comment
partager des informations entre les étapes de test, et comment tester les différentes couches de notre application. Vous
êtes prêt pour la suite 