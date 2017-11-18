# java-builder-pattern-tricks
The humble java builder pattern has been described frequently but is nearly always described in a very basic form without revealing its true potential! If you push the pattern a bit harder then you get can less verbosity and more compile-time safety.

So what are these extra tricks?

 * Shortcut the `builder()` method
 * Enforce mandatory parameters at compile time with *builder chaining*
 * Build generic signatures with builder chaining
 * Force formatting in IDEs of method chaining (avoid long lines of code!)

## What's the builder pattern for?

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
Book book = Book
  .builder()
  .author("Charles Dickens")
  .title("Great Expectations")
  .category("Novel")
  .build();
```
Note that I haven't prefixed the methods with `set` or `with`. Seems like pointless noise to me but go with whatever style you like.

So that example looks ok but we can do better. 

## Making the basic builder better

The first thing I'll note is that when you're chaining methods you end up with something much more readable when you put one method per line as above. You can force an IDE to do honour this when it formats code by **adding an empty comment at the end of each line**:

```java
Book book = Book //
  .builder() //
  .author("Charles Dickens") //
  .title("Great Expectations") //
  .category("Novel") //
  .build();
```

Now let's improve the builder. We have to consider the `Book` object itself. It has two mandatory fields `author` and `title` and one optional field `category`. Let's make a little change to the builder. 


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

The *builder()* method has been replaced with a static `author` method and the `author` method is no longer public in the builder. We've shortened the build process now by a line:

```java
Book book = Book
  .author("Charles Dickens")
  .title("Great Expectations")
  .category("Novel")
  .build();
```

However, we haven't captured in a type-safe way that `title` is a mandatory field. This call below compiles but will throw a `NullPointerException` at runtime only:

```java
// will throw NPE!
Book book = Book
  .author("Charles Dickens")
  .build();
```
We can fix this problem with a *builder chaining* trick:

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

See that final `.build()` method? Sometimes we can get rid of that too. If every field was mandatory and they were all builder chained then you could do this:

```java
Book book = Book
  .author("Charles Dickens")
  .title("Great Expectations")
  .category("Novel");
```

That's one less line of code again which is nice but can only be achieved when all fields are mandatory.




