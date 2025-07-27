# Chapitre 5 : Tester les Différentes Couches de l'Application Spring

Jusqu'à présent, nos scénarios BDD agissaient comme un utilisateur final : ils envoyaient une requête HTTP et
vérifiaient la réponse. C'est excellent pour valider le comportement global. Mais que se passe-t-il si une logique
métier complexe se trouve dans une classe de `Service` ? Devons-nous systématiquement créer un `Controller` juste pour
pouvoir la tester ? Non.

Le BDD est une technique de spécification de comportement. Ce comportement peut se situer à n'importe quel niveau : API,
service, ou même un composant de calcul complexe. Nous allons apprendre à adapter nos tests BDD pour qu'ils se
concentrent sur la bonne couche, au bon moment.

### Objectifs pédagogiques

À la fin de ce chapitre, vous serez capable de :

* **Comprendre la différence** entre un test d'intégration complet et un test de couche de service.
* **Écrire un scénario BDD** qui cible directement la logique d'une classe de `@Service`.
* **Utiliser Mockito** et l'annotation `@MockBean` pour simuler (mocker) les dépendances (comme les repositories) et
  isoler le service testé.
* **Tester la couche de persistance** avec `@DataJpaTest` pour ne charger que le contexte JPA.
* **Organiser** vos fichiers `.feature` et vos "Step Definitions" par domaine fonctionnel pour une meilleure
  maintenabilité.

### Introduction : Le Mécanicien et le Pilote

Imaginez une voiture de course.
Le **pilote** fait un test "end-to-end" : il prend la voiture, fait un tour de circuit et vérifie le chrono. C'est ce
que nous avons fait jusqu'à présent avec `MockMvc`. On teste le système dans son ensemble.

Mais parfois, le **mécanicien** veut juste tester le moteur. Il ne va pas mettre la voiture sur le circuit. Il va la
mettre sur un banc d'essai, brancher des capteurs, simuler l'arrivée d'essence et la commande d'accélération, et mesurer
la puissance directement à la sortie du moteur.

Dans ce chapitre, nous allons devenir des mécaniciens. Nous allons mettre notre `@Service` sur un "banc d'essai",
simuler ses dépendances (le `@Repository`) et vérifier sa logique métier de manière isolée.

### 1. Tester la Couche Service avec Mockito

La couche de service contient souvent la logique métier la plus riche de votre application. C'est là que les règles de
gestion, les calculs et les orchestrations se produisent.

**Scénario :** Imaginons un service qui calcule le prix de location d'un livre en fonction de son année de publication.
Les livres récents sont plus chers.

**`src/test/resources/features/pricing/book_pricing.feature`**

```gherkin
Feature: Calcul du prix de location d'un livre

  Scenario Outline: Le prix de location dépend de l'année de publication
    Given un livre publié en <annee_publication>
    When je calcule le prix de location de ce livre
    Then le prix calculé doit être de <prix_attendu> euros

    Examples:
      | annee_publication | prix_attendu |
      | 2023              | 5.0          |
      | 2010              | 3.5          |
      | 1995              | 2.0          |
```

Ce scénario ne parle pas d'API, de HTTP ou de JSON. Il parle de "calculer un prix". C'est un pur test de logique métier.

Pour l'implémenter, nous n'avons pas besoin de `MockMvc`. À la place, nous allons injecter notre service directement et
**mocker** sa dépendance (le `BookRepository`).

**Le Service à tester (`BookPricingService.java`)**

```java
package fr.formation.spring.librarymanagement.service;

import fr.formation.spring.librarymanagement.domain.Book;
import fr.formation.spring.librarymanagement.repository.BookRepository;
import org.springframework.stereotype.Service;

import java.time.LocalDate;

@Service
public class BookPricingService {

    private final BookRepository bookRepository;

    public BookPricingService(BookRepository bookRepository) {
        this.bookRepository = bookRepository;
    }

    public double calculateRentalPrice(String isbn) {
        Book book = bookRepository.findById(isbn)
                .orElseThrow(() -> new IllegalArgumentException("Book not found"));

        int currentYear = LocalDate.now().getYear();
        if (book.getPublicationYear() > currentYear - 5) {
            return 5.0; // Moins de 5 ans
        } else if (book.getPublicationYear() > currentYear - 20) {
            return 3.5; // Moins de 20 ans
        } else {
            return 2.0; // Plus de 20 ans
        }
    }
}
```

**La classe de Step Definitions (`BookPricingStepDefinitions.java`)**

```java
package fr.formation.spring.librarymanagement.bdd;

import fr.formation.spring.librarymanagement.domain.Book;
import fr.formation.spring.librarymanagement.repository.BookRepository;
import fr.formation.spring.librarymanagement.service.BookPricingService;
import io.cucumber.java.en.Given;
import io.cucumber.java.en.Then;
import io.cucumber.java.en.When;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.boot.test.mock.mockito.MockBean;

import java.util.Optional;

import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.Mockito.when;

@SpringBootTest
public class BookPricingStepDefinitions {

    // Le service que nous voulons tester
    @Autowired
    private BookPricingService pricingService;

    // La dépendance du service, que nous allons MOCKER
    @MockBean
    private BookRepository bookRepository;

    private String currentIsbn;
    private double calculatedPrice;

    @Given("un livre publié en {int}")
    public void un_livre_publie_en(int publicationYear) {
        // 1. Créer un faux livre
        Book mockBook = new Book();
        this.currentIsbn = "isbn-" + publicationYear;
        mockBook.setIsbn(this.currentIsbn);
        mockBook.setPublicationYear(publicationYear);

        // 2. Dire à Mockito : "Quand le service appellera
        //    bookRepository.findById avec cet ISBN,
        //    retourne notre faux livre."
        when(bookRepository.findById(this.currentIsbn))
                .thenReturn(Optional.of(mockBook));
    }

    @When("je calcule le prix de location de ce livre")
    public void je_calcule_le_prix_de_location_de_ce_livre() {
        // 3. Appeler directement la méthode du service
        this.calculatedPrice = pricingService
                .calculateRentalPrice(this.currentIsbn);
    }

    @Then("le prix calculé doit être de {double} euros")
    public void le_prix_calcule_doit_etre_de_euros(Double expectedPrice) {
        // 4. Vérifier le résultat
        assertThat(this.calculatedPrice).isEqualTo(expectedPrice);
    }
}
```

<warning>
**La puissance de <code>@MockBean</code>**
<ul>
<li><code>@MockBean</code> remplace le vrai <code>BookRepository</code> dans le contexte Spring par un "mock" (un double) fourni par la bibliothèque Mockito.</li>
<li>Le mock ne fait rien par défaut. C'est à nous de lui dire comment se comporter avec <code>when(...).thenReturn(...)</code>.</li>
<li>Cela nous permet de <b>tester le <code>BookPricingService</code> en isolation totale</b> de la base de données. Le test est plus rapide et ne dépend pas de l'état de la base.</li>
</ul>
</warning>

### 2. Tester la Couche de Persistance (JPA)

Parfois, on veut s'assurer que nos requêtes JPA, nos entités et leurs relations fonctionnent correctement. Dans ce cas,
nous voulons une vraie base de données (en mémoire, comme H2), mais nous n'avons pas besoin de toute la couche Web (
Contrôleurs, Services...).

Spring Boot fournit une annotation parfaite pour cela : `@DataJpaTest`.

**Scénario :** S'assurer qu'un livre peut être sauvegardé et retrouvé.

**`src/test/resources/features/persistence/book_persistence.feature`**

```gherkin
Feature: Persistance de l'entité Livre

  Scenario: Sauvegarder et retrouver un livre
    Given je crée un nouvel objet Livre avec le titre "Le Petit Prince"
    When je sauvegarde ce livre dans la base de données
    Then je dois pouvoir retrouver le livre "Le Petit Prince" depuis la base
```

**La classe de Step Definitions (`BookPersistenceStepDefinitions.java`)**

```java
package fr.formation.spring.librarymanagement.bdd;

import fr.formation.spring.librarymanagement.domain.Book;
import fr.formation.spring.librarymanagement.repository.BookRepository;
import io.cucumber.java.en.Given;
import io.cucumber.java.en.Then;
import io.cucumber.java.en.When;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.orm.jpa.DataJpaTest;
import org.springframework.boot.test.autoconfigure.orm.jpa.TestEntityManager;

import static org.assertj.core.api.Assertions.assertThat;

// @DataJpaTest ne charge que le strict nécessaire pour JPA !
// C'est beaucoup plus léger et rapide que @SpringBootTest.
@DataJpaTest
public class BookPersistenceStepDefinitions {

    // Injecte un EntityManager spécial pour les tests, très pratique.
    @Autowired
    private TestEntityManager entityManager;

    @Autowired
    private BookRepository bookRepository;

    private Book newBook;

    @Given("je crée un nouvel objet Livre avec le titre {string}")
    public void je_cree_un_nouvel_objet_Livre_avec_le_titre(String title) {
        newBook = new Book();
        newBook.setIsbn("978-2070612758");
        newBook.setTitle(title);
        newBook.setAuthor("Antoine de Saint-Exupéry");
        newBook.setPublicationYear(1943);
    }

    @When("je sauvegarde ce livre dans la base de données")
    public void je_sauvegarde_ce_livre_dans_la_base_de_donnees() {
        // Utiliser le repository pour sauvegarder
        bookRepository.save(newBook);
    }

    @Then("je dois pouvoir retrouver le livre {string} depuis la base")
    public void je_dois_pouvoir_retrouver_le_livre(String title) {
        // entityManager.flush(); // Force la synchronisation avec la BDD
        Book foundBook = bookRepository.findById("978-2070612758")
                .orElse(null);

        assertThat(foundBook).isNotNull();
        assertThat(foundBook.getTitle()).isEqualTo(title);
    }
}
```

<warning>
**Quand utiliser <code>@DataJpaTest</code> ?**
Utilisez cette approche lorsque vous voulez tester :
<ul>
<li>Des relations complexes entre entités (<code>@OneToMany</code>, <code>@ManyToMany</code>...).</li>
<li>Des requêtes personnalisées écrites avec <code>@Query</code> dans vos repositories.</li>
<li>Le bon fonctionnement des cascades (<code>CascadeType</code>) ou du chargement (<code>FetchType</code>).</li>
</ul>
C'est un test d'intégration ciblé sur la couche de persistance.
</warning>

---

### Exercice 6 : Tester un service avec un mock

Maintenant, à vous de jouer ! Nous allons créer un service simple qui vérifie si un livre est un "classique" (publié il
y a plus de 50 ans).

**Votre mission :**

1. **Créez** la classe `BookQualificationService` avec une méthode `isClassic(String isbn)`. Cette méthode utilisera le
   `BookRepository` pour trouver le livre et vérifier son année.
2. **Créez** le fichier `book_qualification.feature` avec un `Scenario Outline` pour tester le cas d'un livre classique
   et celui d'un livre moderne.
3. **Créez** la classe de Step Definitions `BookQualificationStepDefinitions`.
4. Dans cette classe, **utilisez `@MockBean`** pour mocker le `BookRepository`.
5. Implémentez les étapes `Given`, `When`, `Then` pour appeler votre service et vérifier le résultat booléen.

**`BookQualificationService.java` (à créer)**

```java
package fr.formation.spring.librarymanagement.service;

// ...
@Service
public class BookQualificationService {
    private final BookRepository bookRepository;
    // ... constructeur ...

    public boolean isClassic(String isbn) {
        // Logique à implémenter :
        // 1. Chercher le livre par ISBN.
        // 2. Si non trouvé, lancer une exception.
        // 3. Comparer son année de publication à l'année actuelle - 50.
        // 4. Retourner true ou false.
    }
}
```

### Correction exercice 6 {collapsible='true'}

**1. `BookQualificationService.java` (implémenté)**

```java
package fr.formation.spring.librarymanagement.service;

import fr.formation.spring.librarymanagement.domain.Book;
import fr.formation.spring.librarymanagement.repository.BookRepository;
import org.springframework.stereotype.Service;

import java.time.LocalDate;
import java.util.NoSuchElementException;

@Service
public class BookQualificationService {

    private final BookRepository bookRepository;

    public BookQualificationService(BookRepository bookRepository) {
        this.bookRepository = bookRepository;
    }

    public boolean isClassic(String isbn) {
        Book book = bookRepository.findById(isbn)
                .orElseThrow(() -> new NoSuchElementException(
                        "Book with ISBN " + isbn + " not found.")
                );

        int classicThresholdYear = LocalDate.now().getYear() - 50;
        return book.getPublicationYear() <= classicThresholdYear;
    }
}
```

**2. `book_qualification.feature`**

```gherkin
Feature: Qualification d'un livre (classique ou moderne)

  Scenario Outline: Vérifier si un livre est un classique
    Given un livre publié en <annee_publication>
    When je qualifie ce livre
    Then le résultat doit être que le livre est un classique : <est_classique>

    Examples:
      | annee_publication | est_classique |
      | 1965              | true          |
      | 2020              | false         |
```

**3. `BookQualificationStepDefinitions.java`**

```java
package fr.formation.spring.librarymanagement.bdd;

import fr.formation.spring.librarymanagement.domain.Book;
import fr.formation.spring.librarymanagement.repository.BookRepository;
import fr.formation.spring.librarymanagement.service.BookQualificationService;
import io.cucumber.java.en.Given;
import io.cucumber.java.en.Then;
import io.cucumber.java.en.When;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.boot.test.mock.mockito.MockBean;

import java.util.Optional;

import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.Mockito.when;

@SpringBootTest
public class BookQualificationStepDefinitions {

    @Autowired
    private BookQualificationService qualificationService;

    @MockBean
    private BookRepository bookRepository;

    private String currentIsbn;
    private boolean isClassicResult;

    @Given("un livre publié en {int}")
    public void un_livre_publie_en(int publicationYear) {
        this.currentIsbn = "isbn-" + publicationYear;
        Book mockBook = new Book();
        mockBook.setIsbn(this.currentIsbn);
        mockBook.setPublicationYear(publicationYear);

        when(bookRepository.findById(this.currentIsbn))
                .thenReturn(Optional.of(mockBook));
    }

    @When("je qualifie ce livre")
    public void je_qualifie_ce_livre() {
        this.isClassicResult = qualificationService.isClassic(currentIsbn);
    }

    @Then("le résultat doit être que le livre est un classique : {string}")
    public void le_resultat_doit_etre_que_le_livre_est_un_classique(String expected) {
        boolean expectedBoolean = Boolean.parseBoolean(expected);
        assertThat(this.isClassicResult).isEqualTo(expectedBoolean);
    }
}
```

---

### Auto-évaluation

1. **Quand est-il préférable d'utiliser `@MockBean` dans un test BDD ?**
   a) Quand on veut tester une API REST de bout en bout.
   b) Quand on veut tester la logique d'un composant (ex: un Service) en l'isolant de ses dépendances (ex: un
   Repository).
   c) Quand on veut vérifier que des entités JPA sont correctement mappées.
   d) `@MockBean` n'est pas utilisé avec Cucumber.

2. **Quelle annotation Spring Boot est la plus adaptée pour tester la couche de persistance seule ?**
   a) `@SpringBootTest`
   b) `@WebMvcTest`
   c) `@DataJpaTest`
   d) `@ServiceTest`

3. **Dans un test de service utilisant un `@MockBean` pour le repository, où la donnée "livre" existe-t-elle
   réellement ?**
   a) Dans la base de données H2.
   b) En mémoire, dans un objet créé par Mockito pour la durée du test.
   c) Dans un fichier JSON chargé au démarrage.
   d) Elle n'existe pas, le service retourne toujours `null`.

4. **Décrivez une situation où il serait plus judicieux d'écrire un scénario BDD pour un service plutôt que pour un
   contrôleur.**

5. **Quelle est la principale différence, en termes de contexte Spring chargé, entre un test annoté `@SpringBootTest` et
   un test annoté `@DataJpaTest` ? Quel est l'impact sur la vitesse d'exécution des tests ?**

---

### Conclusion : Le bon outil pour le bon travail

Vous avez maintenant une boîte à outils de test BDD beaucoup plus complète. Vous n'êtes plus limité aux tests "
end-to-end", vous pouvez choisir avec précision la couche de l'application que vous souhaitez spécifier et valider.

Vous avez appris à :

* **Isoler la logique métier** d'un service en simulant ses dépendances avec `@MockBean`.
* **Valider la couche de persistance** de manière efficace et légère avec `@DataJpaTest`.
* **Adapter vos scénarios Gherkin** pour qu'ils décrivent le comportement de la couche ciblée, et non plus seulement
  celui d'une API.

Cette flexibilité est essentielle dans les projets professionnels. Elle vous permet d'avoir une pyramide de tests
équilibrée : de nombreux tests unitaires et de service rapides et ciblés, et des tests d'intégration "end-to-end" plus
complets mais moins nombreux pour vérifier que tout s'assemble correctement.

Dans le dernier module, nous allons assembler toutes ces connaissances dans un mini-projet de synthèse et discuter des
bonnes pratiques pour faire du BDD un véritable atout dans votre quotidien de développeur.