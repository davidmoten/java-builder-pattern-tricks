# java-builder-pattern-tricks
The java builder pattern has been described frequently but is nearly always described in a very basic form without revealing its true potential! 

So what are these extra tricks?

 * Shortcut the `builder()` method
 * Mandatory parameters with builder chaining C
 * Get compile time indications as the class evolves with field changes
 * Type safety with builder chaining (not just method chaining!)
 * Force formatting in IDEs of method chaining (avoid long lines of code!)

Let's start with a basic builder pattern and then we'll improve it. We'll consider how to build a `Book` object:

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
    
    public static Builder builder() {
        return new Builder();
    }
    
    public static final class Builder {
        String author;
        String title;
        Optional<String> category = Optional.empty();
        
        Builder() {
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

Looks ok doesn't it, but we can do better. 

The first thing I'll note is that when you're chaining methods you end up with something much more readable when you put one method per line as above. You can force an IDE to do this by **adding an empty comment at the end of each line**:

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
        return new Builder().author();
    }
    
    public static final class Builder {
        String author;
        String title;
        Optional<String> category = Optional.empty();
        
        Builder() {
        }
        
        Builder author(String author) {
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

The *builder()* method has been replaced with a static `author` method and the `author` method is no longer public in the builder. We've shortened the build process now by a line:

```java
Book book = Book
  .author("Charles Dickens")
  .title("Great Expectations")
  .category("Novel")
  .build();
```
