# java-builder-pattern-tricks
The humble java builder pattern has been described frequently but is nearly always described in a very basic form without revealing its true potential! If you push the pattern a bit harder then you get can less verbosity in your API and more compile-time safety.

So what are these extra tricks?

 * [Format the code better (long method chained lines are yuk!)](#trick-1-formatting)
 * Shortcut the `builder()` method
 * Enforce mandatory parameters at compile time with *builder chaining*
 * Remove final `build()` call when all fields mandatory
 * Build generic signatures with builder chaining
 
 The open-source library [rxjava2-jdbc](https://github.com/davidmoten/rxjava2-jdbc) uses all these tricks to make the API easier to use.

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
The first thing I'll note is that when you're chaining methods you end up with something much more readable when you put one method per line. You can force an IDE to do honour this when it formats code by **adding an empty comment at the end of each line**:

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
        
        Builder2 title(String title) {
            this.title = title;
            return new Builder2(this);
        }
    }
    
    public static final class Builder2 {
        final Builder b;
        
        Builder2(Builder b) {
            this.b = b;
        }
        
        Builder2 category(String category) {
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
To demonstrate the building of generic signatures, we'll show an example with *Tuples*.

I want a class now that builds typed *Tuples* but I want a standard build method that honours generic types:

```java
Type2<Integer, String> t2 = Tuples.value(12).value("thing").build();
Type3<Integer, String, Date> t3 = Tuples.value(12).value("thing").value(new Date()).build(); 
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

## Conclusion
With luck you've seen that there is more power to the builder pattern than you realized and users of your APIs will benefit!




```



