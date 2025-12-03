# JPA ORM Mapping

## Testing

We use JUnit Jupiter for unit testing. Mind the following testing slices for Spring-based tests @DataJpaTest,
@WebMvcTest(UserController.class) or @JsonTest - all in solution: @SpringBootTest

All repositories, richtypes and converters must be tested exhaustively.

### Single Tests
```java
import static org.assertj.core.api.Assertions.*;
import org.junit.jupiter.api.Test;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.orm.jpa.DataJpaTest;

@DataJpaTest
class UserRepositoryTest {

  private @Autowired UserRepository userRepository;

  @Test
  void can_save_and_reread() {
    // assign all mandatory attributes, for relationships: assign both sides to be safe
    var user = User.builder().build();

    // use saveAndFlush to test db constraints (not needed for exam)
    var saved = userRepository.save(user);

    // test all attributes (isSameAs, isEqualTo, isNotNull)
    assertThat(saved).isSameAs(user).extracting(User::getId).isNotNull();

    // more complext queries must be defined in the Repository.
    var reread = userRepository.getById(saved.getId());

    // test multiple attributes
    assertThat(reread).isSameAs(user).extracting(User::getId).isNotNull();    
  }
}
```

### Parameterized Tests

User for enums and other cases where it is easy to test multiple values. Null-case must all be tested!

```java

import static org.assertj.core.api.Assertions.*;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.orm.jpa.DataJpaTest;

import org.junit.jupiter.params.ParameterizedTest;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.params.provider.Arguments;
import org.junit.jupiter.params.provider.MethodSource;

@DataJpaTest
public class GamePlatformConverterTest {

  @ParameterizedTest
  @MethodSource("validMappings") // method name that supplies inputs
  void can_convert_valid(Character value, GamePlatform expectedPlatform) {
    var converter = new GamePlatformConverter();

    var converted = converter.convertToEntityAttribute(value);

    assertThat(converted).isNotNull().isEqualTo(expectedPlatform);

    var convertBack = converter.convertToDatabaseColumn(converted);

    assertThat(convertBack).isNotNull().isEqualTo(value);
  }
  
  private static Stream<Arguments> validMappings() {
    return Stream.of(
      Arguments.of('P', GamePlatform.PC),
      Arguments.of('Y', GamePlatform.PLAYSTATION),
      Arguments.of('X', GamePlatform.XBOX),
      Arguments.of('S', GamePlatform.SWITCH)
    );
  }

  @Test
  void cannot_convert_invalid() {
    var value = 'L';
    var converter = new GamePlatformConverter();
    assertThatThrownBy(() -> converter.convertToEntityAttribute(value)).isInstanceOf(DataConstraintViolationException.class);
  }
```

## Validation
Use Jakarta Validation at model and data transfer objects.
Annotations like:
- `@Email`
- `@FutureOrPresent / @PastOrPresent`
- `@Max`
- `@Min`
- `@NotBlank`: also enforces `@NotNull` and `@NotEmpty`
- `@NotEmpty`
- `@NotNull`
- `@Size`

In addition, constraints can be enforced on the database-level:
```java
@Column(unique=true)
private String email;
```
OR (for composite uniques):
```java
@Table(
	uniqueConstraints = { @UniqueConstraint(columnNames = 
	{ "personNumber", "isActive" }) // field-names
	})
class MyCoolTable { ... }
```

### Exceptions

Custom exception type with static instance methods for full points:
```java
public class DataConstraintViolationException extends RuntimeException {

  private DataConstraintViolationException(String message) {
    super(message);
  } 
  
  private DataConstraintViolationException(String message, Throwable cause) {
    super(message, cause);
  } 

  public static <T> DataConstraintViolationException forInvalidValue(Class<?> clazz, T value)  {
    return new DataConstraintViolationException("Invalid value provided for: %s, provided value: %s.".formatted(clazz.getSimpleName(), value.toString()));
  }
  
  public static <T> DataConstraintViolationException forMissingValue(Class<?> clazz) {
    return new DataConstraintViolationException("Missing value for: %s".formatted(clazz.getSimpleName()));
  }
  
  public static <E extends Enum<E>> DataConstraintViolationException forInvalidEnumVariant(Class<E> clazz) {
    return new DataConstraintViolationException("Expected valid enum variant for: %s".formatted(clazz.getSimpleName()));
  }
  
  public static <T> DataConstraintViolationException forInvalidTimePeriod(Class<?> clazz, T start, T end) {
    return new DataConstraintViolationException("Invalid time period for %s, with start: %s and end: %s.".formatted(clazz.getSimpleName(), start.toString(), end.toString()));
  }
}
```
## Persistence
### Repository
Responsible for persisting entities to the database. This is where queries are defined as well. Often times, it is more efficient to use the generated queries.

```java
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.stereotype.Repository;

@Repository
public interface CustomerRepository extends JpaRepository<Customer, Customer.CustomerId> {
  public Customer findByFirstnameAndLastname(String firstname, String lastname);
  
  // for more complex queries: (named parameters as :<name>)
  @Query("""
       SELECT c 
       FROM Customer c
       JOIN c.rentals r
       WHERE r.price > :price
       """)
List<Customer> findCustomersWithRentalsAbove( BigDecimal price);

	// for projections (where you select a specific field from the return value):
	@Query("""
		select new spengergasse.Projection(s.firstname, s.grade)
		from Student s
		""")
		public Projection projectionFromStudent()
	}
```

### Projections
Used to select specific fields as the return-values of queries.
```java
package spengergasse;

public record Projection(String firstname, int grade) {}
```

### Entity
Needs @Entity and @Id or @EmbeddedId; @Table is optional but recommended.
Technical attributes like id are most likely not represented in the UML class diagram!

```java
@Entity
@Table(name="users")
public class User {
	private @EmbeddedId UserId id;

	private String firstName;

	@NotNull private String lastName;
	private record UserId(@GeneratedValue Long id) implements Serializable {}
}
```

Important Annotations:
- `@NoArgsConstructor(access = AccessLevel.PROTECTED)` ... for JPA
- `@AllArgsConstructor`
- `@Builder` or `@SuperBuilder` for inhertance-entities
- `@Getter`
- `@Setter`
- `@EqualsAndHashcode(of = "id")`

### Value Objects
Must **not** define an @Id or @EmbeddedId.

```java
@Embeddable
public record Address() {}
```

If one type of value object occurs in an owner-entity multiple times, the attributes and associations must be overriden, to prevent duplicate columns:
```java
/** An Entity to which the Embeddable has a ManyToOne relation */
@Entity
public class Location {
  @Id
  private java.lang.Integer id; 

  ...
}

/** Embeddable that will be embed twice in the same table */
@Embeddable 
public class User {
  // Basic mapping
  @Column(name="USER_NAME")
  private java.lang.String userName;

  // Basic mapping
  @Column(name="USER_ACCOUNT")
  private java.lang.Integer userAccount;

  // Relationship
  @ManyToOne()
  @JoinColumn(name = "LOCATION_ID", referencedColumnName = "id")
  public Location location;

  ...
}

/** Entity holding two instances of the same Embeddable type */
@Entity
public class Employee {
  @Id
  private long id;
  ...
  
  // Embed an instance of User
  @Embedded
  // rename the basic mappings
  @AttributeOverrides({
    @AttributeOverride(name="userName", column=@Column(name="EMPLOYEE_NAME")),
    @AttributeOverride(name="userAccount", column=@Column(name="EMPLOYEE_ACCOUNT"))
  })
  // rename the relationship
  @AssociationOverrides({
    @AssociationOverride(name = "location",
                joinColumns = @JoinColumn(name = "EMPLOYEE_LOCATION_ID"))
  })
  private User employee;

  // Embed a second instance of User
  @Embedded
  // rename the basic mappings
  @AttributeOverrides({
    @AttributeOverride(name="userName", column=@Column(name="SUPERVISOR_NAME")),
    @AttributeOverride(name="userAccount", column=@Column(name="SUPERVISOR_ACCOUNT"))
  })
  // rename the relationship
  @AssociationOverrides({
    @AssociationOverride(name = "location",
                joinColumns = @JoinColumn(name = "SUPERVISOR_LOCATION_ID"))
  })
  private User supervisor;

  ...
}
```

### Rich Types
Represents a type, that should be mapped to a single database column. Implemented via @Embeddable or by providing a @Converter / implements AttributeConverter.

```java
@Embeddable
public record Email(@Email String address) {}

public record GeoCoordinate(double longitude, double latitude) {}

@Converter(autoapply=true)
public GeoCoordinateConverter implements AttributeConverter<GeoCoordinate, String> {
...
}

```

Converters **MUST NOT** check for business logic/null. This is the responsibility of the rich-type constructor. In case of null as an input, null is returned.
### Enumerations
Faulty default behavior when mapped, we use a converter to ensure consistency.
```java
public enum GamePlatform {
  PC,
  PLAYSTATION,
  XBOX,
  SWITCH
}
```

```java
package at.spengergasse.ivehif.december_probetest.persistance.converters;

import at.spengergasse.ivehif.december_probetest.foundation.DataConstraintViolationException;
import at.spengergasse.ivehif.december_probetest.richtypes.GamePlatform;
import jakarta.persistence.AttributeConverter;
import jakarta.persistence.Converter;

@Converter(autoApply = true)
public class GamePlatformConverter implements AttributeConverter<GamePlatform, Character>{

	@Override
	public Character convertToDatabaseColumn(GamePlatform value) {
	  return switch(value) {
	    case null -> null;
	    case GamePlatform.PC -> 'P';
	    case GamePlatform.PLAYSTATION -> 'Y';
	    case GamePlatform.XBOX -> 'X';
	    case GamePlatform.SWITCH -> 'S';
	  };
	}

	@Override
	public GamePlatform convertToEntityAttribute(Character dbData) {
	  return switch(dbData) {
	    case null -> null;
	    case 'P' -> GamePlatform.PC;
	    case 'Y' -> GamePlatform.PLAYSTATION;
	    case 'X' -> GamePlatform.XBOX;
	    case 'S' -> GamePlatform.SWITCH;
	    default -> throw DataConstraintViolationException.forInvalidEnumVariant(GamePlatform.class);
	  };
	}
}
```
### Basic Attributes
like do them I guess. If you have a special name that might be reserved, overwrite it with `@Column(name = "...")`

### Inheritance
#### Single-Table
Single table inheritance is the simplest and typically the best performing and best solution. It is like a roll-up in database design – all child-table fields are put into the parent-entity. There is just one actual table in the database.

```java
@Entity
@Inheritance
@DiscriminatorColumn(name="PROJ_TYPE") // column used to differentiate
@Table(name="PROJECT")
public  class Project implements Serializable{
  @EmbeddedId
  private ProjectId id;
  ...
}

@Entity
@DiscriminatorValue("L")
public class LargeProject extends Project {
  private BigDecimal budget;
}

@Entity
@DiscriminatorValue("S")
public class SmallProject extends Project {
}
```

Result:

| ID  | PROJ_TYPE | NAME        | BUDGET |
| --- | --------- | ----------- | ------ |
| 1   | L         | Accountings | 5000   |
| 2   | S         | Legal       | null   |

#### Joined

Joined inheritance is the most logical inheritance solution because it mirrors the object model in the data model. In joined inheritance a table is defined for _each_ class in the inheritance hierarchy to store _only_ the local attributes of that class. The child-tables ID column is a FK to the parent. Note the `@PrimaryKeyJoinColumn(foreignKey = ...)` annotation.
```java
@Entity
@Table(name="media")
@Inheritance(strategy = InheritanceType.JOINED)
public class Medium {
  private @EmbeddedId MediumId id;
  private @NotNull String title;
  private @Min(1888) Integer releaseYear;
  private String genre;
  private @Version Integer version;

  public record MediumId(@GeneratedValue Long id) implements Serializable {}
  
}

@Table(name="movies")
@PrimaryKeyJoinColumn(foreignKey = @ForeignKey(name = "FK_movie_media")) // for joined inheritance
public class Movie extends Medium {
  private String director;
  private Integer durationInMinutes;
  private AgeRating ageRating; // converted auto-applied
}
```

#### Table-Per-Class
Table per class inheritance allows inheritance to be used in the object model, when it does not exist in the data model. Nevertheless, the parent-class CAN still exist as an entity (if non-abstract). In table per class inheritance a table is defined for _each_ concrete class in the inheritance hierarchy to store _all_ the attributes of that class and _all_ of its superclasses.
```java
@Entity
@Inheritance(strategy=InheritanceType.TABLE_PER_CLASS)
public abstract class Project {
  @Id
  private long id;
  ...
}
@Entity
@Table(name="LARGEPROJECT")
public class LargeProject extends Project {
  private BigDecimal budget;
}
@Entity
@Table(name="SMALLPROJECT")
public class SmallProject extends Project {
}
```
Result:
SMALLPROJECT:

| ID  | NAME  |
| --- | ----- |
| 2   | Legal |

LARGEPROJECT:

| ID  | NAME       | BUDGET |
| --- | ---------- | ------ |
| 1   | Accounting | 5000   |
#### Mapped Superclass
Mapped superclass inheritance also allows inheritance to be used in the object model, when it does not exist in the data model. It is similar to table per class inheritance, but does not allow querying, persisting, or relationships to the superclass. This is like a roll-down.
```java
@MappedSuperclass
public abstract class Project {
  @EmbeddedId
  private ProjectId id;
  @Column(name="NAME")
  private String name;
  ...
}
// shared columns wowww that can be changed hoooh
@Entity
@Table(name="LARGEPROJECT")
@AttributeOverride(name="NAME", column=@Column(name="PROJECT_NAME"))
public class LargeProject extends Project {
  private BigDecimal budget;
  ...
}
```
I don't see why you would ever make use to this for our use-case.

### ForeignKeys
Mind the necessary configuration of CascadeType s and the optional, but recommended config of the physical @JoinColumn (FK-Constraint). `@ForeignKey(name = "FK_a_b")`

### Collections

#### of Entities
Usually mapped via reverse foreign key or an intermediate table. Mind the declaration of necessary foreign key constraints. Overall, a relationship is always uni-direction, to map the other side, the `mappedBy` attribute must be used.
##### OneToOne
```java
@Entity
public class Employee {
	@EmbeddedId
	private EmplyoeeId id;
	...
	@OneToOne(fetch=FetchType.LAZY) // or eager is fetched a lot
	@JoinColumn(foreignKey = @ForeignKey(name = "ermwatthesigma" )) // nullable = true/false
	private Address address;
  
}

@Entity
public class Address {
	@EmbeddedId
	private AddressId id;
	...
	@OneToOne(fetch=FetchType.LAZY, mappedBy="address") // other side of relationship
	private Employee owner;
	...
}
```
##### OneToMany

```java
@Entity
@Table(name="rentals")
public class Rental {
	public record RentalId(@GeneratedValue Long id) implements Serializable {}
	private @EmbeddedId RentalId id;
	...
	@ManyToOne(cascade = CascadeType.PERSIST) // mind the cascade type
	@JoinColumn(nullable = false, foreignKey = @ForeignKey(name = "FK_rental_customer"))
	private @NotNull Customer customer;
	...
}

@Entity
@Table(name="customers")
public class Customer {
	private @EmbeddedId CustomerId id;
	...
	@OneToMany(mappedBy = "customer", cascade = CascadeType.PERSIST) // other side of relationship
	private List<Rental> rentals;

```
##### ManyToMany
```java
@Entity
@Table(name="affiliations")
public class Affiliation {
    @Builder.Default // default value of builder: new HashSet<>(3)
    @ManyToMany(cascade = CascadeType.PERSIST)
    @JoinTable( // "Kreuztabelle"
        name = "affiliation_countries",
        joinColumns = @JoinColumn(
            name = "affiliation_id",
            foreignKey = @ForeignKey(name = "FK_affiliation_countries_2_affiliation")
        ),
        inverseJoinColumns = @JoinColumn(
            name = "country_id",
            foreignKey = @ForeignKey(name = "FK_affiliation_countries_2_country")
        )
    )
    private Set<Country> countries = new HashSet<>(3);
    ...
}

@Entity
@Table(name="countries")
public class Country {
	@Builder.Default
    @ManyToMany(mappedBy = "countries") // other side of relationship
    private Set<Affiliation> affiliations = new HashSet<>(20);
	...
}
```
#### of ValueObjects
Because valueobjects do not have an ID, we create an extra table for the collection with a FK to the owner of the value object.
```java
    @Builder.Default
    @ElementCollection
    @CollectionTable(
        name = "challenge_attachements",
        foreignKey = @ForeignKey(name = "FK_challenge_attachements_2_challenge")
    )
    private Set<ChallengeAttachment> attachements = new HashSet<>(3);

```
##### _Primary keys in CollectionTable_

The JPA 2.0 specification does not provide a way to define the `Id` in the `Embeddable`. However, to delete or update an element of the `ElementCollection` mapping, some unique _key_ is normally required. Otherwise, on every update the JPA provider would need to delete everything from the `CollectionTable` for the `Entity`, and then insert the values back. So, the JPA provider will most likely assume that the combination of all of the fields in the `Embeddable` are unique, in combination with the foreign key (`JoinColumn`(s)). This however could be inefficient, or just not feasible if the `Embeddable` is big, or complex.

Some JPA providers may allow the `Id` to be specified in the `Embeddable`, to resolve this issue. Note in this case the `Id` only needs to be unique for the collection, not the table, as the foreign key is included. Some may also allow the `unique` option on the `CollectionTable` to be used for this. Otherwise, if your `Embeddable` is complex, you may consider making it an `Entity` and use a `OneToMany` instead.

## Optimistic Locking
Every entity that also has a repository and where it generally makes sense (does not just inherit all attributes from parent etc.) must have a version field for optimistic locking.
```java
private @Version Integer version;
```
