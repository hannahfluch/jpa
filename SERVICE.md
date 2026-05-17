# Service Layer

## What Belongs In The Service Layer

The service layer contains business logic. Controllers should call services, not implement rules themselves.

Domain-specific calculations and functionalities must be abstracted in the domain-layer itself and also tested there directly.

## Typical Service Structure

```java
@Service
@RequiredArgsConstructor
@Transactional(readOnly = true)
public class ArtistService {
    private final ArtistRepository artistRepository;

    public List<ArtistDto> getArtists() {
        return artistRepository.findAllProjectedBy();
    }

    public Optional<ArtistDto> getArtist(String artistName) {
        return artistRepository.findProjectedByArtistName(artistName);
    }

    @Transactional
    public ArtistDto create(@Valid CreateArtistCommand command) {
        if (artistRepository.existsByArtistName(command.artistName())) {
            throw new ConflictException("Artist already exists");
        }

        var artist = new Artist(
            command.artistName(),
            command.genre(),
            command.biography(),
            command.socialMediaHandle()
        );

        return ArtistDto.from(artistRepository.save(artist));
    }
}
```

## Transactions

Use:

```java
@Transactional(readOnly = true)
```

on the class for read-only default behavior.

Use:

```java
@Transactional
```

on write methods:

- create
- update
- replace
- patch
- delete
- business actions that mutate state

## Return Types

Good service return types:

```java
Optional<ArtistDto>       // may not exist
boolean                   // simple success/failure
ArtistDto                 // guaranteed result
TicketRefundResult        // custom business result
```

Avoid returning raw `Map` or complex nested collections when the result has business meaning. Create a custom record/class instead that abstracts the underlying collection.

## Money

Use `BigDecimal`, never `double` or `float`.

```java
private static final BigDecimal REGULAR_REFUND_RATE = new BigDecimal("0.75");

private BigDecimal refundAmount(TicketOrder order) {
    return switch (order.getTicketType()) {
        case REGULAR -> order.getPrice().multiply(REGULAR_REFUND_RATE);
        case VIP -> order.getPrice();
        case PROMO, FESTIVAL -> BigDecimal.ZERO;
    };
}
```

## Domain Logic Should Be Abstracted

If a calculation describes the domain, do not duplicate it everywhere.

Bad:

```java
int total = hotel.getFloors() * hotel.getRoomsPerFloor();
```

Better:

```java
int total = hotel.numberOfTotalRooms();
```

Business methods belong close to the domain when they describe the object itself. The domain-logic must also be tested directly.

## Assumptions

If the task is underspecified, make a reasonable assumption and document it in a short comment.

Example:

```java
// Assumption: if both performances start at the same minute, the second one
// in the input list is treated as the later performance.
```

## Exceptions

Use custom exceptions when the controller should handle cases differently.

Examples:

```java
public class ConflictException extends RuntimeException {
    public ConflictException(String message) {
        super(message);
    }
}
```

Common service exception cases:

- duplicate resource
- invalid business operation
- referenced entity does not exist
- operation not allowed

`IllegalArgumentException` is acceptable for simple invalid arguments, but custom exceptions are easier to map to HTTP status codes.

## Service Unit Tests

Use Mockito for dependencies.

```java
@ExtendWith(MockitoExtension.class)
class ArtistServiceTest {
    @Mock ArtistRepository artistRepository;
    @InjectMocks ArtistService artistService;
}
```

Example:

```java
@Test
void cant_create_duplicate_artist() {
    var command = CreateArtistCommand.builder()
        .artistName("DJ Electric")
        .genre("Electronic")
        .build();

    when(artistRepository.existsByArtistName("DJ Electric")).thenReturn(true);

    assertThatThrownBy(() -> artistService.create(command))
        .isInstanceOf(ConflictException.class);

    verify(artistRepository, never()).save(any());
}
```

## Performance

Use `.forEach` instead of normal for-loops.
