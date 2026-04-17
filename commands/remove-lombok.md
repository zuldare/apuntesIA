Analyze Lombok usage in the specified class or package and remove Lombok annotations, following the project's established conventions.

**Target:** $ARGUMENTS (a fully qualified class name or package path)

## Analysis steps

For each class with Lombok annotations:

1. **Identify Lombok annotations used:** `@Data`, `@Builder`, `@NoArgsConstructor`, `@AllArgsConstructor`, `@Getter`, `@Setter`, `@ToString`, `@EqualsAndHashCode`, `@Value`, `@RequiredArgsConstructor`, etc.

2. **Classify the class type:**
   - **JPA Entity** (`@Entity`): MUST remain a class. JPA requires mutable fields, no-args constructor, and records are NOT supported.
   - **JPA Embeddable** (`@Embeddable`): Same as entity — must remain a class.
   - **Immutable View** (`@Entity` + `@Immutable`): Keep as class with `public` no-args constructor and getters only (no setters). Use `public` visibility on no-args constructor because tests may need cross-package access.
   - **DTO / Message / Event** (no JPA annotations): Candidate for Java `record` IF it doesn't extend a class. Records can implement interfaces.
   - **`@ConfigurationProperties`** class: Must remain a class with getters+setters for Spring setter-based binding (Spring Boot 3.x). In Spring Boot 4.x, constructor binding is default and records would work — but for now, keep as class.
   - **Abstract class or class with inheritance**: Cannot be a record. Keep as class.

3. **Check Builder necessity:**
   - Search for `ClassName.builder()` usage in both `src/main/` and `src/test/`.
   - If Builder is used → implement hand-written Builder (static inner class pattern).
   - If Builder is NOT used anywhere → do NOT add a Builder.
   - Search for `toBuilder()` usage. If used, implement `toBuilder()` method. If not used, omit it.

4. **Check constructor usage:**
   - For classes with few non-confusable fields (<=3 fields of different types), consider if Builder is overkill — a simple constructor may be enough.
   - For classes with many fields or multiple fields of the same type (e.g., multiple Strings), a Builder prevents argument-ordering bugs.

5. **Check `@Data` risks on entities:**
   - `@Data` generates `equals()`, `hashCode()`, `toString()` — these are dangerous on JPA entities with lazy-loaded `@ManyToOne`/`@OneToMany` relations (can trigger lazy loading or `LazyInitializationException`).
   - When removing `@Data` from entities, do NOT generate custom `equals()`/`hashCode()`/`toString()` — just omit them (JPA uses identity equality by default, which is correct).

## Implementation conventions

- **Never use Lombok in the refactored code.** Implement everything by hand.
- **Records:** Use for DTOs/messages/events. Jackson 2.12+ deserializes records via canonical constructor. `@JsonFormat` and other Jackson annotations work on record components.
- **Entity Builder pattern:**
  ```java
  public static Builder builder() { return new Builder(); }
  public static final class Builder {
      private Builder() {}
      // fluent setters returning `this`
      public ClassName build() {
          ClassName entity = new ClassName();
          entity.field = field; // direct field access from inner class
          return entity;
      }
  }
  ```
- **Record Builder pattern:**
  ```java
  public record MyDto(String name, int value) {
      public static Builder builder() { return new Builder(); }
      public static final class Builder {
          // fields, fluent setters
          public MyDto build() { return new MyDto(name, value); }
      }
  }
  ```
- **Entity no-args constructor:** Use `public` visibility (not `protected`), especially for `@Immutable` views tested from different packages.
- **All code in English:** Comments, Javadoc, identifiers, log messages.

## After making changes

1. **Find and update ALL affected tests:**
   - Search for `ClassName.builder()` calls in test files — they should still work with the hand-written Builder.
   - Search for `.getXxx()` getter calls — if class was converted to record, these become `.xxx()` (record accessor style).
   - Search for `.setXxx()` calls — if entity kept setters, these still work. If converted to record, these must be replaced.
   - Search for `new ClassName()` calls — ensure no-args constructor is still available where needed.
   - For `@Immutable` views in tests, if setters were removed, use `ReflectionTestUtils.setField()` to set private fields.

2. **Compile and verify:**
   - Run `./mvnw compile -q` to check for compilation errors.
   - If errors are found, fix them before proceeding to the next class.

3. **Report summary** for each class:
   - Class name and type (entity/record/class)
   - Annotations removed
   - Whether Builder was kept/added/removed and why
   - Whether converted to record and why/why not
   - Tests updated (if any)
