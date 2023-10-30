# java-builder-pattern-tricks
The humble java builder pattern has been described frequently but is nearly always described in a very basic form without revealing its true potential! If you push the pattern a bit harder then you get can less verbosity in your API and more compile-time safety.

So what are these extra tricks?

 * [Format the code better (long method chained lines are yuk!)](#trick-1-formatting)
 * [Shortcut the `builder()` method](#shortcut-the-builder-method)
 * [Enforce mandatory parameters at compile time with *builder chaining*](#trick-3-enforce-mandatory-fields-at-compile-time-with-builder-chaining)
 * [Remove final `build()` call when all fields mandatory](#trick-4-remove-final-build-call-when-all-fields-mandatory)
 * [Build generic signatures with builder chaining](#trick-5-build-generic-signatures)
 * [Improve discoverability](#trick-6-improve-discoverability)
 * [Build lists succinctly](#trick-7-build-lists-succinctly)
 
 The open-source library [rxjava2-jdbc](https://github.com/davidmoten/rxjava2-jdbc) uses all these tricks to make the API easier to use.
 
[java-builder2](https://github.com/davidmoten/java-builder2) is a library to generate builders.

## What's the basic builder pattern for?
* constructor field assignments can be mixed up
* constructor calls are not very readable because Java doesn't support named parameters

## The basic builder pattern
Let's start with a basic builder pattern and then we'll improve it. We'll consider how to build a `Book` object.

Our `Book` object has:
* a mandatory `author` field
* a mandatory `title` field
* an optional `category` field

```java
public final class Book {
    // Make fields final so we always know we've missed assigning one in the constructor!
    private final String author;
    private final String title;
    private final Optional<String> category;
    
    //should not be public
    private Book(Builder builder) {
        //Be a bit defensive
        Preconditions.checkNotNull(builder.author);
        Preconditions.checkArgument(builder.author.trim().length() > 0);
        Preconditions.checkNotNull(builder.title);
        Preconditions.checkArgument(builder.title.trim().length() > 0);
        Preconditions.checkNotNull(category);
        this.author = builder.author;
        this.title = builder.title;
        this.category = builder.category;
    }
    
    public String author() {
        return author;
    }
    
    public String title() {
        return title;
    }
    
    public Optional<Category> category() {
       return category;
    }
    
    public static Builder builder() {
        return new Builder();
    }
    
    public static final class Builder {
        String author;
        String title;
        Optional<String> category = Optional.empty();
        
        // should not be public to force use of the static builder() method
        private Builder() {
        }
        
        public Builder author(String author) {
            this.author = author;
            return this;
        }
        
        public Builder title(String title) {
            this.title = title;
            return this;
        }
        
        public Builder category(String category) {
            this.category = Optional.of(category);
            return this;
        }
        
        public Book build() {
            return new Book(this);
        }   
    }
}
```

To use this basic builder:

```java
Book book = Book.builder().author("Charles Dickens").title("Great Expectations")
    .category("Novel").build();
```
Note that I haven't prefixed the methods with `set` or `with`. Seems like pointless noise to me but go with whatever style you like.

So that example looks ok but we can do better. 

## Making the basic builder better
### Trick 1: Formatting
The first thing I'll note is that when you're chaining methods you end up with something much more readable when you put one method per line. You can force an IDE to honour this when it formats code by **adding an empty comment at the end of each line**:

```java
Book book = Book //
  .builder() //
  .author("Charles Dickens") //
  .title("Great Expectations") //
  .category("Novel") //
  .build();
```

### Trick 2: Shortcut the builder() method

The `builder()` method call is unnecessary if you have a mandatory field. We can replace that call with a call to one or many of the mandatory fields like `author`:

So instead of the declaration:

```java
public Builder builder() {
    return new Builder();
}
```
we put:
```java
public Builder author(String author) {
     return new Builder().author(author);
}
```
and we reduce the visibility of the `Builder.author(String)` method (because it has already been set).

Now we can call:

```java
Book book = Book //
  .author("Charles Dickens") //
  .title("Great Expectations") //
  .category("Novel") //
  .build();
```
Saved one line, nice!

By incorporating `author` as a final field in the builder and passing it in the constructor we can clean up the internals a bit more:

```java
public final class Book {
    // Make fields final so we always know we've missed assigning one in the constructor!
    private final String author;
    private final String title;
    private final Optional<String> category;
    
    private Book(Builder builder) {
        //Be a bit defensive
        Preconditions.checkNotNull(builder.author);
        Preconditions.checkArgument(builder.author.trim().length() > 0);
        Preconditions.checkNotNull(builder.title);
        Preconditions.checkArgument(builder.title.trim().length() > 0);
        Preconditions.checkNotNull(category);
        this.author = builder.author;
        this.title = builder.title;
        this.category = builder.category;
    }
    
    public String author() {
        return author;
    }
    
    public String title() {
        return title();
    }
    
    public Optional<Category> category() {
       return category;
    }
    
    public static Builder author(String author) {
        return new Builder(author);
    }
    
    public static final class Builder {
        final String author;
        String title;
        Optional<String> category = Optional.empty();
        
        private Builder(String author) {
            this.author = author;
        }
       
        public Builder title(String title) {
            this.title = title;
            return this;
        }
        
        public Builder category(String category) {
            this.category = Optional.of(category);
            return this;
        }
        
        public Book build() {
            return new Book(this);
        }     
    }
}
```

Note that you may prefer to continue using the `builder()` method particularly if the class already has a number of public static methods and distinguishing your builder entry method is difficult.

### Trick 3: Enforce mandatory fields at compile time with builder chaining
Now let's improve the builder some more. We have to consider the `Book` object itself. It has two mandatory fields `author` and `title` and one optional field `category`. 

As the builder stands so far we have a runtime check on the mandatory fields:

```java
// will throw NullPointerException because title missing!
Book book = Book
  .author("Charles Dickens")
  .build();
```
It would be even better to get compile-time indication and this is something we can achieve with the *builder chaining* trick:

```java
public final class Book {
    // Make fields final!
    private final String author;
    private final String title;
    private final Optional<String> category;
    
    private Book(Builder builder) {
        //Be a bit defensive
        Preconditions.checkNotNull(builder.author);
        Preconditions.checkArgument(builder.author.trim().length() > 0);
        Preconditions.checkNotNull(builder.title);
        Preconditions.checkArgument(builder.title.trim().length() > 0);
        Preconditions.checkNotNull(category);
        this.author = builder.author;
        this.title = title;
        this.category = category;
    }
    
    public String author() {
        return author;
    }
    
    public String title() {
        return title();
    }
    
    public Optional<Category> category() {
       return category;
    }
    
    public static BuilderHasAuthor author(String author) {
        return new BuilderHasAuthor(author);
    }
    
    public static final class BuilderHasAuthor {
        final String author;
        String title;
        Optional<String> category = Optional.empty();
        
        private BuilderHasAuthor(String author) {
            this.author = author;
        }
        
        BuilderHasTitle title(String title) {
            this.title = title;
            return new Builder2(this);
        }
    }
    
    public static final class BuilderHasTitle {
        final BuilderHasAuthor b;
        
        BuilderHasTitle(BuilderHasAuthor b) {
            this.b = b;
        }
        
        BuilderHasTitle category(String category) {
            b.category = Optional.of(category);
            return this;
        }
                
        public Book build() {
            return new Book(b);
        }     
    }
}
```
Now if we try the previous example it won't compile. That's really good because finding bugs at compile time is MUCH better than finding out at runtime!

```java
// will not compile!
Book book = Book
  .author("Charles Dickens")
  .build();
```

In fact, the new builder forces us to follow an exact method order in that you can't swap the order of `author`,`title` and `category` methods.

```java
Book book = Book
  .author("Charles Dickens")
  .title("Great Expectations")
  .category("Novel")
  .build();
```

Hey `category` is an optional parameter isn't it? Let's test that:

```java
Book book = Book
  .author("Charles Dickens")
  .title("Great Expectations")
  .build();
```
Yep that compiles, no problem.

Note also the naming standard `BuilderHas*` that helps readability.

### Trick 4: Remove final build() call when all fields mandatory
See that final `.build()` method? Sometimes we can get rid of that too. If every field was mandatory and they were all builder chained then you could do this:

```java
Book book = Book
  .author("Charles Dickens")
  .title("Great Expectations")
  .category("Novel");
```

That's one less line of code again which is nice but can only be achieved when all fields are mandatory.

To achieve this just replace this code:

```java
        Builder2 category(String category) {
            b.category = Optional.of(category);
            return this;
        }
                
        public Book build() {
            return new Book(b);
        }   
```
with
```java
        Book category(String category) {
            b.category = Optional.of(category);
            return new Book(b);
        }
```

### Trick 5: Build generic signatures
To demonstrate the building of generic signatures, we'll show a contrived example with *Tuples*.

I want a class now that builds typed *Tuples* but I want a standard build method that honours generic types:

```java
Type2<Integer, String> t2 = Tuples
  .value(12)
  .value("thing")
  .build();
Type3<Integer, String, Date> t3 = Tuples
  .value(12)
  .value("thing")
  .value(new Date())
  .build(); 
```
This api is achieved using builder chaining where each chained builder adds another generic signature:

```java
public final class Tuple2<A,B> {
    public final A value1;
    public final B value2;
    public Tuple2(A value1, B value2) {
      this.value1 = value1;
      this.value2 = value2;
    }
}

public final class Tuple3<A,B,C> {
    public final A value1;
    public final B value2;
    public final C value3;
    public Tuple2(A value1, B value2, C value3) {
      this.value1 = value1;
      this.value2 = value2;
      this.value3 = value3;
    }
}

public final class Tuples {

    public static <T> Builder1<T> value(T t){
        return new Builder1<T>(t);
    }
    
    public static final class Builder1<A> {
        private final A value1;
        private Builder1(A value1) {
            this.value1 = value1;
        }
        public <B> value(A value2) {
            return new Builder2<A, B>(value1, value2);
        }
    }
    
    public static final class Builder2<A, B> {
        private final A value1;
        private final B value2;
        private Builder2(A value1, B value2) {
            this.value1 = value1;
            this.value2 = value2;
        }
        public <C> Builder3<A, B, C> value(C value3) {
            return new Builder3<A, B, C>(value1, value2);
        }
        
        public Tuple2<A,B> build() {
            return new Tuple2<A,B>(value1, value2);
        }
    }
    
    public static final class Builder3<A, B, C> {
        private final A value1;
        private final B value2;
        private final C value3;
        private Builder3(A value1, B value2, C value3) {
            this.value1 = value1;
            this.value2 = value2;
            this.value3 = value3;
        }
        
        //could add a Builder4 to keep chaining!
        
        public Tuple3<A, B, C> build() {
            return new Tuple3<A, B, C>(value1, value2, value3);
        }
    }
}
```
### Trick 6: Improve discoverability
If your builder is to create a `Thing` then ideally they should be able to start the builder via a static method on `Thing`:

```java
Thing thing = Thing.name("FRED").sizeMetres(1.5).create();
```
If you can't add a method to `Thing` (you might not own the api) then favour use of `Things`:

```java
Thing thing = Things.name("FRED").sizeMetres(1.5).create();
```
For clarity reasons (for example if the class already has a lot of public static methods) if you want to use a `builder` method then go for it:

```java
Thing thing = Thing.builder().name("FRED").sizeMetres(1.5).create();
```
### Trick 7: Build lists succinctly
Using a special chaining trick we can support the building of lists like this:

```java
Group
  .name("friends")
  .firstName("John")
  .lastName("Smith")
  .yearOfBirth(1965)
  .firstName("Anne")
  .lastName("Jones")
  .build();
```
In the example above `firstName` and `lastName` is mandatory and `yearOfBirth` is optional.

You see that we didn't need to return back to the top level builder to add the next person. This is achieved by adding methods pointing back to the top level builder from the person builder both for creating the next person but also for building the group if the list has finished. Here's all the code:

```java
public final class Group {

    private final String name;
    private final List<Person> persons;

    public Group(String name, List<Person> persons) {
        this.name = name;
        this.persons = persons;
    }

    public String name() {
        return name;
    }

    public List<Person> persons() {
        return persons;
    }

    public static Builder name(String name) {
        Preconditions.checkNotNull(name);
        return new Builder(name);
    }

    public static final class Builder {
        final String name;
        List<Person> persons = new ArrayList<>();

        Builder(String name) {
            this.name = name;
        }

        public Builder2 firstName(String firstName) {
            Preconditions.checkNotNull(firstName);
            return new Builder2(this, firstName);
        }

        public Group build() {
            return new Group(name, persons);
        }
    }

    public static final class Builder2 {

        private final Builder b;
        private final String firstName;
        private String lastName;
        private Optional<Integer> yearOfBirth = Optional.empty();

        Builder2(Builder b, String firstName) {
            this.b = b;
            this.firstName = firstName;trick-7-build-lists-succintly
        }

        public Builder3 lastName(String lastName) {
            Preconditions.checkNotNull(firstName);
            this.lastName = lastName;
            return new Builder3(this);
        }
    }

    public static final class Builder3 {

        private final Builder2 person;

        Builder3(Builder2 person) {
            this.person = person;
        }

        public Builder3 yearOfBirth(int yearOfBirth) {
            person.yearOfBirth = Optional.of(yearOfBirth);
            return this;
        }

        public Builder2 firstName(String firstName) {
            Person p = new Person(person.firstName, person.lastName, person.yearOfBirth);
            person.b.persons.add(p);
            return person.b.firstName(firstName);
        }

        public Group build() {
            return person.b.build();
        }
    }

    public static final class Person {
        private final String firstName;
        private final String lastName;

        private final Optional<Integer> yearOfBirth;

        public Person(String firstName, String lastName, Optional<Integer> yearOfBirth) {
            this.firstName = firstName;
            this.lastName = lastName;
            this.yearOfBirth = yearOfBirth;
        }

        public String firstName() {
            return firstName;
        }

        public String lastName() {
            return lastName;
        }

        public Optional<Integer> yearOfBirth() {
            return yearOfBirth;
        }
    }
}
```

## Conclusion
With luck you've seen that there is more power to the builder pattern than you realized and users of your APIs will benefit!




