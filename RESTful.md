# RESTful Layer

Controllers translate HTTP requests into service calls and service results into HTTP responses. Business logic should not be handled here directly.

## Controller Structure

```java
@RestController
@RequestMapping("/api/artists")
@RequiredArgsConstructor
public class ArtistController {
    private final ArtistService artistService;

    @GetMapping
    public ResponseEntity<List<ArtistDto>> getArtists() {
        return ResponseEntity.ok(artistService.getArtists());
    }

    @GetMapping("/{artistName}")
    public ResponseEntity<ArtistDto> getArtist(@PathVariable String artistName) {
        return artistService.getArtist(artistName)
            .map(ResponseEntity::ok)
            .orElse(ResponseEntity.notFound().build());
    }

    @PostMapping
    public ResponseEntity<ArtistDto> createArtist(
        @Valid @RequestBody CreateArtistCommand command
    ) {
        var created = artistService.create(command);
        var location = URI.create("/api/artists/" + created.artistName());

        return ResponseEntity.created(location).body(created);
    }

    @PutMapping("/{artistName}")
    public ResponseEntity<ArtistDto> replaceArtist(
        @PathVariable String artistName,
        @Valid @RequestBody UpdateArtistCommand command
    ) {
        return artistService.replace(artistName, command)
            .map(ResponseEntity::ok)
            .orElse(ResponseEntity.notFound().build());
    }
}
```

## HTTP Methods

Use the method that matches the operation:

```text
GET     read resource
POST    create resource or execute command/action
PUT     replace complete resource
PATCH   partial update
DELETE  delete resource
```

## Path Variables

Path variables are part of the resource identity and are a required part of the request.

```text
GET /api/performances/{performanceId}
```

```java
@GetMapping("/{performanceId}")
public ResponseEntity<PerformanceDto> getPerformance(
    @PathVariable UUID performanceId
) {
    return performanceService.getPerformance(performanceId)
        .map(ResponseEntity::ok)
        .orElse(ResponseEntity.notFound().build());
}
```

## Query Parameters

Query parameters are optional filters or options.

```text
GET /api/performances?date=2026-07-15
```

```java
@GetMapping
public ResponseEntity<List<PerformanceDto>> getPerformances(
    @RequestParam(required = false)
    @DateTimeFormat(iso = DateTimeFormat.ISO.DATE)
    LocalDate date
) {
    return ResponseEntity.ok(performanceService.getPerformances(date));
}
```

Use query parameters for:

- filters
- search terms
- sorting
- paging
- optional date filters

## Paging

Paging parameters are query parameters. Spring can bind them directly into a `Pageable` argument.

```text
GET /api/challenges?page=1&size=5
GET /api/users?page=0&size=20
GET /api/affiliations/search/cfe?page=1&size=5
```

Example:

```java
@GetMapping
public ResponseEntity<SlicedModel<EntityModel<ChallengeDto>>> getChallenges(
    Pageable pageable
) {
    var challenges = challengeService.getChallenges(pageable);
    var model = slicedAssembler.toModel(challenges, challengeAssembler::dtoToModel);

    return challenges.isEmpty()
        ? ResponseEntity.noContent().build()
        : ResponseEntity.ok(model);
}
```

Service and repository methods should accept the same `Pageable` and return a `Slice<T>` when the API only needs the current page plus information about whether another page exists.

```java
public Slice<ChallengeDto> getChallenges(Pageable pageable) {
    return challengeRepository.findAllProjectedBy(ChallengeDto.class, pageable);
}
```

```java
<T> Slice<T> findAllProjectedBy(Class<T> projection, Pageable pageable);
```

Use `Slice<T>` is more lightweight. `Page<T>` is heavier because it also calculates total elements and total pages.


For HATEOAS endpoints, tests should assert `page.size` and `page.number`:

```java
mockMvc.perform(get("/api/challenges")
        .param("page", "1")
        .param("size", "5"))
    .andExpect(status().isOk())
    .andExpect(jsonPath("$._embedded.challengeDtoList[0].name").value("Web Warmup"))
    .andExpect(jsonPath("$.page.size").value(5))
    .andExpect(jsonPath("$.page.number").value(1));
```

For non-HATEOAS `Slice<T>` responses like `CountryController`, the JSON uses `content`, `size`, and `number` directly:

```java
mockMvc.perform(get("/api/countries")
        .param("page", "1")
        .param("size", "5"))
    .andExpect(status().isOk())
    .andExpect(jsonPath("$.content[0].name").value("Austria"))
    .andExpect(jsonPath("$.size").value(5))
    .andExpect(jsonPath("$.number").value(1));
```

## Dates

For ISO dates like:

```text
2026-07-15
```

use `LocalDate`.

In a request body:

```java
public record CreatePerformanceCommand(
    @NotNull LocalDate day
) {}
```

As a query parameter:

```java
@RequestParam
@DateTimeFormat(iso = DateTimeFormat.ISO.DATE)
LocalDate date
```

## Command Objects

Commands are input objects for write operations and are the input for a controller.

Example create command:

```java
@Builder
public record CreateArtistCommand(
    @NotBlank String artistName,
    @NotBlank String genre,
    String biography,
    String socialMediaHandle
) {}
```

Example update command:

```java
@Builder
public record UpdateArtistCommand(
    @NotBlank String genre,
    String biography,
    String socialMediaHandle
) {}
```

Commands should be validated with annotations like `@Valid` and `@RequestBody` in the controller.

## DTOs

DTOs are output objects of the controller.

```java
public record ArtistDto(
    String artistName,
    String genre,
    String biography,
    String socialMediaHandle
) {}
```

Exam note: if mappers are excluded, simple manual construction is fine.

## Validation

Use validation annotations on command fields:

```java
@NotBlank
@NotNull
@Min(1)
@Size(max = 64)
@Pattern(regexp = "\\d{2}:\\d{2}")
@Valid
```

`BindingResult` can be used to catch errors that happen during validation and should always be used in combination with `@Valid`, it must come directly after the validated argument:

```java
@PostMapping
public ResponseEntity<?> create(
    @Valid @RequestBody CreateArtistCommand command,
    BindingResult bindingResult
) {
    if (bindingResult.hasErrors()) {
        return ResponseEntity.badRequest().body(RequestErrorDto.from(bindingResult));
    }

    var created = artistService.create(command);
    return ResponseEntity.created(...).body(created);
}
```

## Top-Level Arrays

Collections should be abstracted into custom records that wrap the raw collection.

Instead of:

```java
ResponseEntity<List<ArtistDto>>
```

Wrapped version:

```java
public record ArtistListDto(List<ArtistDto> artists) {}
```

Response:

```json
{
  "artists": [
    {
      "artistName": "DJ Electric",
      "genre": "Electronic"
    }
  ]
}
```

## Global Controller Advice

Use global advice for cross-controller exception handling.

```java
package at.spengergasse.sj2526pos1javaseed.presentation.api;

import org.springframework.http.HttpStatus;
import org.springframework.http.ProblemDetail;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.ControllerAdvice;
import org.springframework.web.bind.annotation.ExceptionHandler;

import at.spengergasse.sj2526pos1javaseed.foundation.BadRequestException;
import lombok.extern.slf4j.Slf4j;

@ControllerAdvice
@Slf4j
public class GlobalControllerAdvice {
    @ExceptionHandler(BadRequestException.class)
    public ResponseEntity<ProblemDetail> handleBadRequestException(BadRequestException e) {
        log.warn("Bad request: {}", e.getMessage(), e);
        ProblemDetail problemDetail = ProblemDetail.forStatus(HttpStatus.BAD_REQUEST);
        problemDetail.setDetail(e.getMessage() != null ? e.getMessage() : "The request was invalid.");

        return ResponseEntity.badRequest().body(problemDetail);
    }

    @ExceptionHandler
    public ResponseEntity<ProblemDetail> handleException(Throwable t) {
        log.error("Unhandled throwable: {}", t.getMessage(), t);
        ProblemDetail problemDetail = ProblemDetail.forStatus(HttpStatus.INTERNAL_SERVER_ERROR);
        problemDetail.setDetail(t.getMessage() != null ? t.getMessage() : "An internal error occurred. Sorry :(");

        return ResponseEntity.internalServerError().body(problemDetail);
    }
}
```


## Local Exception Handler

An `@ExceptionHandler` inside a controller applies only to that controller.

```java
@RestController
@RequestMapping("/api/bookings")
public class BookingController {

    @ExceptionHandler(BookingException.class)
    public ResponseEntity<ProblemDetail> handleBookingException(
        BookingException ex
    ) {
        var problem = ProblemDetail.forStatus(HttpStatus.BAD_REQUEST);
        problem.setDetail(ex.getMessage());
        return ResponseEntity.badRequest().body(problem);
    }
}
```

Use local handlers for controller-specific business exceptions.

Use global advice for general exceptions.

## Questions

These points should be clarified before implementing a new exam-style REST endpoint in this project.

### Empty Collection Behavior

The example exam expects collection endpoints to return `200 OK` with an empty list when no resources exist.

```java
mockMvc.perform(get("/api/artists"))
    .andExpect(status().isOk())
    .andExpect(content().json("[]"));
```

Current project controllers often return `204 No Content` for empty slices.

```java
return challenges.isEmpty()
    ? ResponseEntity.noContent().build()
    : ResponseEntity.ok(model);
```

Clarify which behavior is expected before writing tests:

```text
Exam/simple REST style: 200 OK + []
Current project style:   204 No Content
```

### Location Header Builder

The example exam uses `ServletUriComponentsBuilder` to build the `Location` header after creating a resource.

```java
var location = ServletUriComponentsBuilder.fromCurrentRequest()
    .path("/{artistName}")
    .buildAndExpand(artist.artistName())
    .toUri();

return ResponseEntity.created(location).body(artist);
```

The current project often uses the HATEOAS `self` link from the response model.

```java
var model = challengeAssembler.toModel(challenge);
var self = model.getRequiredLink("self");

return ResponseEntity.created(self.toUri()).body(model);
```

Clarify whether a new endpoint should follow plain exam style with `ServletUriComponentsBuilder` or project style with HATEOAS links.

## HATEOAS

Use HATEOAS when a resource response should tell the client what can be done next:

- `self`: link to the current resource
- `collection`: link back to the resource collection
- `search`: link to search/filter endpoint
- domain-specific links (e.g. for ctf challenge: `solves`)
- affordances for allowed state-changing operations like `DELETE`, `PATCH`, `PUT`, or action-style `POST`

### Single Resource Model

Single resources should return `EntityModel<Dto>`.

Example:

```java
@GetMapping("/{key}")
public ResponseEntity<EntityModel<ChallengeDto>> getChallenge(@PathVariable Long key) {
    return challengeService
        .getChallenge(new Challenge.ChallengeId(key))
        .map(challengeAssembler::dtoToModel)
        .map(ResponseEntity::ok)
        .orElse(ResponseEntity.notFound().build());
}
```

The assembler owns link creation. Controllers should not duplicate link-building code for every response.
Mappers are not part of the exam, so the actions performed by `toModel` can probably be inlined (see [open questions](#questions)).

```java
@Component
@RequiredArgsConstructor
public class ChallengeAssembler
    implements RepresentationModelAssembler<Challenge, EntityModel<ChallengeDto>> {

    private final ChallengeMapper mapper;

    @Override
    public EntityModel<ChallengeDto> toModel(Challenge challenge) {
        return dtoToModel(mapper.toDto(challenge));
    }

    public EntityModel<ChallengeDto> dtoToModel(ChallengeDto dto) {
        var id = dto.id().id();

        var self = linkTo(methodOn(ChallengeController.class).getChallenge(id))
            .withSelfRel()
            .andAffordance(afford(methodOn(ChallengeController.class).deleteChallenge(id)))
            .andAffordance(afford(methodOn(ChallengeController.class).patchChallenge(id, null)))
            .andAffordance(afford(methodOn(ChallengeController.class).solveChallenge(id, null, null)));

        var collection = linkTo(methodOn(ChallengeController.class).getChallenges(null))
            .withRel("collection");
        var solves = linkTo(methodOn(ChallengeController.class).getSolves(id, null))
            .withRel("solves");
        var attachments = linkTo(methodOn(ChallengeController.class).getAttachments(id))
            .withRel("attachments");

        return EntityModel.of(dto, self, collection, solves, attachments);
    }
}
```

Expected response shape:

```json
{
  "id": { "id": 1 },
  "name": "Web Warmup",
  "description": "Find the flag",
  "category": "WEB",
  "difficulty": "EASY",
  "_links": {
    "self": { "href": "http://localhost/api/challenges/1" },
    "collection": { "href": "http://localhost/api/challenges" },
    "solves": { "href": "http://localhost/api/challenges/1/solves" },
    "attachments": { "href": "http://localhost/api/challenges/1/attachments" }
  }
}
```

### Collection Model With Paging

For paged or sliced endpoints, a `SlicedResourcesAssembler` can be used.

```java
private final SlicedResourcesAssembler<ChallengeDto> slicedAssembler;

@GetMapping
public ResponseEntity<SlicedModel<EntityModel<ChallengeDto>>> getChallenges(Pageable pageable) {
    var challenges = challengeService.getChallenges(pageable);
    var model = slicedAssembler.toModel(challenges, challengeAssembler::dtoToModel);

    return challenges.isEmpty()
        ? ResponseEntity.noContent().build()
        : ResponseEntity.ok(model);
}
```

The returned JSON is not a top-level array. It is a HAL document with `_embedded`, `_links`, and `page`.

```json
{
  "_embedded": {
    "challengeDtoList": [
      {
        "id": { "id": 1 },
        "name": "Web Warmup",
        "_links": {
          "self": { "href": "http://localhost/api/challenges/1" },
          "collection": { "href": "http://localhost/api/challenges" }
        }
      }
    ]
  },
  "_links": {
    "self": { "href": "http://localhost/api/challenges?page=0&size=20" }
  },
  "page": {
    "size": 20,
    "number": 0
  }
}
```

Use this JSON path style in tests:

```java
.andExpect(jsonPath("$._embedded.challengeDtoList[0].name").value("Web Warmup"))
.andExpect(jsonPath("$._embedded.challengeDtoList[0]._links.self.href").exists())
.andExpect(jsonPath("$.page.size").value(20))
.andExpect(jsonPath("$.page.number").value(0))
```

### Create Responses

For `POST`, create the model first and use its `self` link for the `Location` header. This avoids manually rebuilding the URI.

```java
@PostMapping
public ResponseEntity<?> createChallenge(
    @Valid @RequestBody CreateChallengeCommand cmd,
    BindingResult bindingResult
) {
    if (bindingResult.hasErrors()) {
        return ResponseEntity.badRequest().body(RequestErrorDto.from(bindingResult));
    }

    var challenge = challengeService.create(cmd);
    var model = challengeAssembler.toModel(challenge);
    var self = model.getRequiredLink("self");

    return ResponseEntity.created(self.toUri()).body(model);
}
```

Test the status, `Location`, DTO fields, and links:

```java
mockMvc.perform(post("/api/challenges")
        .contentType(MediaType.APPLICATION_JSON)
        .content(objectMapper.writeValueAsString(command)))
    .andExpect(status().isCreated())
    .andExpect(header().string("Location", endsWith("/api/challenges/1")))
    .andExpect(jsonPath("$.name").value("Web Warmup"))
    .andExpect(jsonPath("$._links.self.href").exists())
    .andExpect(jsonPath("$._links.collection.href").exists());
```

## Controller Tests

Use `@WebMvcTest`.

```java
@WebMvcTest(ArtistController.class)
class ArtistControllerTest {
    @Autowired MockMvc mockMvc;
    @Autowired ObjectMapper objectMapper;

    @MockitoBean ArtistService artistService;
}
```

The service is mocked. The controller is tested through HTTP.

### MockMvc Configuration

```java
package at.spengergasse.sj2526pos1javaseed.presentation.api;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.extension.ExtendWith;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.restdocs.RestDocumentationContextProvider;
import org.springframework.restdocs.RestDocumentationExtension;
import org.springframework.test.web.servlet.MockMvc;
import org.springframework.test.web.servlet.setup.MockMvcBuilders;
import org.springframework.web.context.WebApplicationContext;

import com.fasterxml.jackson.databind.ObjectMapper;

import static org.springframework.restdocs.mockmvc.MockMvcRestDocumentation.documentationConfiguration;
import static org.springframework.restdocs.operation.preprocess.Preprocessors.prettyPrint;

@ExtendWith({ RestDocumentationExtension.class })
abstract class AbstractRestDocsControllerTest {
    protected @Autowired MockMvc mockMvc;

    protected @Autowired ObjectMapper objectMapper;

    protected @Autowired WebApplicationContext context;

    protected RawResponseBodySnippet rawResponseBodySnippet = new RawResponseBodySnippet();

    @BeforeEach
    public void setUp(RestDocumentationContextProvider restDocumentation) {
        this.mockMvc = MockMvcBuilders
            .webAppContextSetup(context)
            .apply(
                documentationConfiguration(restDocumentation)
                    .snippets()
                    .withAdditionalDefaults(rawResponseBodySnippet)
                    .and()
                    .operationPreprocessors()
                    .withRequestDefaults(prettyPrint())
                    .withResponseDefaults(prettyPrint())
            )
            .build();
    }
}

```

## GET Collection Test

```java
@Test
void can_fetch_artists() throws Exception {
    var artist = new ArtistDto(
        "DJ Electric",
        "Electronic",
        "Award-winning producer",
        "@djelectric"
    );

    when(artistService.getArtists()).thenReturn(List.of(artist));

    mockMvc.perform(get("/api/artists"))
        .andExpect(status().isOk())
        .andExpect(jsonPath("$[0].artistName").value("DJ Electric"))
        .andExpect(jsonPath("$[0].genre").value("Electronic"));

    verify(artistService).getArtists();
}
```

For wrapped collections:

```java
.andExpect(jsonPath("$.artists[0].artistName").value("DJ Electric"))
```

## GET One Test

```java
@Test
void can_fetch_existing_artist() throws Exception {
    var artist = new ArtistDto("DJ Electric", "Electronic", null, "@djelectric");

    when(artistService.getArtist("DJ Electric")).thenReturn(Optional.of(artist));

    mockMvc.perform(get("/api/artists/DJ Electric"))
        .andExpect(status().isOk())
        .andExpect(jsonPath("$.artistName").value("DJ Electric"));

    verify(artistService).getArtist("DJ Electric");
}
```

```java
@Test
void cant_fetch_missing_artist() throws Exception {
    when(artistService.getArtist("Missing")).thenReturn(Optional.empty());

    mockMvc.perform(get("/api/artists/Missing"))
        .andExpect(status().isNotFound());

    verify(artistService).getArtist("Missing");
}
```

## POST Test

```java
@Test
void can_create_artist() throws Exception {
    var command = CreateArtistCommand.builder()
        .artistName("DJ Electric")
        .genre("Electronic")
        .biography("Award-winning producer")
        .socialMediaHandle("@djelectric")
        .build();

    var created = new ArtistDto(
        "DJ Electric",
        "Electronic",
        "Award-winning producer",
        "@djelectric"
    );

    when(artistService.create(any())).thenReturn(created);

    mockMvc.perform(post("/api/artists")
            .contentType(MediaType.APPLICATION_JSON)
            .content(objectMapper.writeValueAsString(command)))
        .andExpect(status().isCreated())
        .andExpect(header().string("Location", "/api/artists/DJ Electric"))
        .andExpect(jsonPath("$.artistName").value("DJ Electric"));

    verify(artistService).create(any());
}
```

## Invalid Request Test

```java
@Test
void cant_create_invalid_artist() throws Exception {
    var command = CreateArtistCommand.builder().build();

    mockMvc.perform(post("/api/artists")
            .contentType(MediaType.APPLICATION_JSON)
            .content(objectMapper.writeValueAsString(command)))
        .andExpect(status().isBadRequest());

    verify(artistService, never()).create(any());
}
```

## Duplicate Resource Test

```java
@Test
void cant_create_duplicate_artist() throws Exception {
    var command = CreateArtistCommand.builder()
        .artistName("DJ Electric")
        .genre("Electronic")
        .build();

    when(artistService.create(any()))
        .thenThrow(new ConflictException("Artist already exists"));

    mockMvc.perform(post("/api/artists")
            .contentType(MediaType.APPLICATION_JSON)
            .content(objectMapper.writeValueAsString(command)))
        .andExpect(status().isConflict());

    verify(artistService).create(any());
}
```
