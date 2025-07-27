# Chapitre 6 : Projet de Synthèse et Bonnes Pratiques

Bienvenue dans ce dernier chapitre ! Jusqu'à présent, nous avons appris les différents mouvements, les techniques, les
outils. Nous avons pratiqué chaque "kata" (forme) du BDD isolément. Il est maintenant temps de les assembler dans un "
kumite" (combat) réel : la réalisation d'une fonctionnalité complète.

Ce projet de synthèse est votre épreuve du feu. Vous partirez d'un besoin métier (une User Story) pour arriver à une
fonctionnalité complète, testée et validée, en appliquant rigoureusement le cycle et les principes du BDD.

### Objectifs pédagogiques

À la fin de ce chapitre, vous serez capable de :

* **Traduire** une User Story en une série de scénarios Gherkin.
* **Appliquer** le cycle BDD complet (Spec -> Rouge -> Vert -> Refactor) pour implémenter une fonctionnalité CRUD.
* **Combiner** les techniques apprises : `Scenario Outline`, `Test Context`, `MockMvc`.
* **Mettre en œuvre** une fonctionnalité complète de manière guidée par les tests de comportement.
* **Ressentir** la confiance que procure le fait d'avoir une suite de tests qui valide chaque aspect du besoin métier.

### Introduction : Devenir le maître d'œuvre

Imaginez que vous êtes un architecte. Vous avez appris à dessiner des fenêtres, des portes, des murs. Maintenant, un
client vous demande de construire une maison entière. C'est à la fois intimidant et exaltant.

C'est exactement notre position. Le "client" (notre Product Owner imaginaire) nous a donné des besoins pour notre API de
bibliothèque. Notre mission est de transformer ces besoins en un logiciel fonctionnel et robuste. Le BDD sera notre plan
directeur, notre boussole, nous assurant à chaque étape que ce que nous construisons est bien ce qui a été demandé.

### Le Projet : Finaliser le CRUD de notre API de Livres

Notre mission est de compléter les fonctionnalités de gestion des livres.

**User Stories (Besoins métier) :**

1. **US1 (Création) :** En tant que bibliothécaire, je veux pouvoir ajouter un nouveau livre au catalogue pour enrichir
   notre collection. *(Nous avons déjà bien avancé sur celle-ci)*.
2. **US2 (Lecture) :** En tant qu'utilisateur, je veux pouvoir lister tous les livres disponibles pour parcourir le
   catalogue.
3. **US3 (Mise à jour) :** En tant que bibliothécaire, je veux pouvoir modifier les informations d'un livre pour
   corriger des erreurs.
4. **US4 (Suppression) :** En tant que bibliothécaire, je veux pouvoir supprimer un livre du catalogue s'il est perdu ou
   retiré.

Nous allons implémenter ensemble la User Story 4 (Suppression) pas à pas. Votre exercice sera d'implémenter la User
Story 2 (Lecture de tous les livres).

### Phase 1 : La Spécification (Écrire le Gherkin pour la suppression)

Le premier réflexe : se réunir avec les "Trois Amigos" et définir les comportements attendus avec des exemples.

**`src/test/resources/features/book_management.feature`**
(Nous pouvons créer un nouveau fichier pour regrouper tout le CRUD)

```gherkin
Feature: Gestion du cycle de vie d'un livre

  En tant que bibliothécaire,
  je veux pouvoir créer, lire, mettre à jour et supprimer des livres
  pour maintenir le catalogue à jour.

  Background:
    Given la base de données contient les livres suivants:
      | isbn           | title      | author         | publicationYear |
      | "978-0441013593" | "Dune"     | "Frank Herbert"| 1965            |
      | "978-0553803719" | "Foundation" | "Isaac Asimov" | 1951            |

  Scenario: Suppression réussie d'un livre existant
    When je demande à supprimer le livre avec l'ISBN "978-0441013593"
    Then l'opération doit réussir avec un statut "No Content"
    And le livre avec l'ISBN "978-0441013593" ne doit plus exister

  Scenario: Tentative de suppression d'un livre inexistant
    When je demande à supprimer le livre avec l'ISBN "999-9999999999"
    Then l'opération doit échouer avec un statut "Not Found"
```

### Phase 2 : Le Cycle Rouge-Vert-Refactor pour la Suppression

<procedure title="Implémentation guidée de la suppression">

1. **Lancez les tests.** Vous obtiendrez des "undefined steps". Créez une nouvelle classe
   `BookManagementStepDefinitions.java` et copiez-y les snippets. N'oubliez pas de la lier à Spring (`@SpringBootTest`,
   `@AutoConfigureMockMvc`, etc.).

2. **Étape ROUGE : Implémentez les steps de test.**
   ```java
   package fr.formation.spring.librarymanagement.bdd;
   // ... imports ...
   
   @SpringBootTest
   @AutoConfigureMockMvc
   // ...
   public class BookManagementStepDefinitions {
   
       @Autowired
       private MockMvc mockMvc;
       @Autowired
       private BookRepository bookRepository;
       // ... injectez le TestContext si vous en avez besoin ...
   
       private ResultActions resultActions;
   
       // Given et Background déjà gérés par notre DataInitializer
       // ou peuvent être implémentés ici pour plus de contrôle.
   
       @When("je demande à supprimer le livre avec l'ISBN {string}")
       public void je_demande_a_supprimer_le_livre_avec_l_isbn(String isbn) 
               throws Exception {
           resultActions = mockMvc.perform(delete("/api/books/" + isbn));
       }
   
       @Then("l'opération doit réussir avec un statut {string}")
       public void l_operation_doit_reussir_avec_un_statut(String status) 
               throws Exception {
           if ("No Content".equalsIgnoreCase(status)) {
               resultActions.andExpect(status().isNoContent()); // 204
           }
       }
   
       @Then("le livre avec l'ISBN {string} ne doit plus exister")
       public void le_livre_avec_l_isbn_ne_doit_plus_exister(String isbn) {
           assertThat(bookRepository.existsById(isbn)).isFalse();
       }
   
       @Then("l'opération doit échouer avec un statut {string}")
       public void l_operation_doit_echouer_avec_un_statut(String status) 
               throws Exception {
           if ("Not Found".equalsIgnoreCase(status)) {
               resultActions.andExpect(status().isNotFound()); // 404
           }
       }
   }
   ```
   **Lancez les tests :** Ils échoueront avec une erreur 404 (Not Found), car l'endpoint `DELETE /api/books/{isbn}`
   n'existe pas. C'est parfait, nous sommes dans le ROUGE.

3. **Étape VERTE : Écrivez le code de production.**
   Ajoutez la méthode de suppression dans `BookController.java`.
   ```java
   // Dans BookController.java
   
   @DeleteMapping("/{isbn}")
   public ResponseEntity<Void> deleteBook(@PathVariable String isbn) {
       if (!bookRepository.existsById(isbn)) {
           // Le livre n'existe pas, on ne peut pas le supprimer.
           return ResponseEntity.notFound().build();
       }
       bookRepository.deleteById(isbn);
       // Suppression réussie, on retourne 204 No Content.
       return ResponseEntity.noContent().build();
   }
   ```
   **Lancez les tests :** Tout passe au VERT ! Notre fonctionnalité est implémentée et validée par le comportement
   attendu.

4. **Étape REFACTOR :** Le code est simple et clair. Le test aussi. Pour l'instant, pas de refactoring majeur
   nécessaire. On a bien décomposé les `Then` en deux étapes distinctes (vérifier le statut, vérifier l'état en base),
   ce qui est une bonne pratique.

</procedure>

---

### Exercice 7 : Implémenter la liste des livres

Maintenant, c'est votre tour ! Appliquez la même démarche pour la User Story 2 : "En tant qu'utilisateur, je veux
pouvoir lister tous les livres disponibles pour parcourir le catalogue."

**Votre mission :**

1. **Ajoutez un scénario** à `book_management.feature` pour décrire ce comportement. Pensez à vérifier que la liste
   retournée n'est pas vide et qu'elle contient le bon nombre d'éléments définis dans le `Background`.
2. **Exécutez**, récupérez les snippets, et implémentez les steps de test dans la classe
   `BookManagementStepDefinitions`. L'appel sera un `GET /api/books`.
3. **Observez l'échec** (ROUGE).
4. **Écrivez le code** dans `BookController` pour faire passer le test (VERT).
5. Vérifiez que le corps de la réponse JSON contient bien un tableau de 2 livres. Utilisez
   `jsonPath("$.size()").value(2)`.

### Correction exercice 7 {collapsible='true'}

**1. Scénario ajouté à `book_management.feature`**

```gherkin
  Scenario: Lister tous les livres du catalogue
    When je demande la liste de tous les livres
    Then je dois recevoir une liste contenant 2 livres
    And la réponse est un succès
```

**2. Implémentation des Steps dans `BookManagementStepDefinitions.java`**

```java
// Dans BookManagementStepDefinitions.java

@When("je demande la liste de tous les livres")
public void je_demande_la_liste_de_tous_les_livres() throws Exception {
    resultActions = mockMvc.perform(get("/api/books")
            .accept(MediaType.APPLICATION_JSON));
}

@Then("je dois recevoir une liste contenant {int} livres")
public void je_dois_recevoir_une_liste_contenant_livres(int count)
        throws Exception {
    resultActions.andExpect(jsonPath("$.size()").value(count));
}

@Then("la réponse est un succès")
public void la_reponse_est_un_succes() throws Exception {
    resultActions.andExpect(status().isOk());
}
```

**3. Code de production dans `BookController.java`**

```java
// Dans BookController.java

@GetMapping
public ResponseEntity<List<Book>> getAllBooks() {
    List<Book> books = bookRepository.findAll();
    return ResponseEntity.ok(books);
}
```

En lançant les tests après ces ajouts, tous les scénarios devraient passer au vert.

**Appel API correspondant (format `HTTPClient`)**

```http
GET http://localhost:8080/api/books
Accept: application/json

###
```

La réponse attendue serait un JSON contenant un tableau avec les deux livres, "Dune" et "Foundation".

---

### Conclusion de cette partie

Vous l'avez fait ! Vous avez pris en charge des besoins métier et les avez transformés en fonctionnalités logicielles
solides, fiables et vérifiables. Ce n'est pas juste un exercice technique ; c'est le cœur du métier de Concepteur
Développeur d'Applications.

Vous avez prouvé que vous pouvez utiliser le BDD non pas comme une contrainte, mais comme un guide qui sécurise votre
développement et garantit la qualité.

Dans la dernière partie de ce module, nous allons prendre un peu de recul et discuter des grandes règles d'or du BDD et
d'un dernier aspect pratique très important : comment partager vos résultats de test avec le reste de l'équipe.