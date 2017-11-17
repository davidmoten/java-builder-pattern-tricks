# java-builder-pattern-tricks
The java builder pattern has been described frequently but is nearly always described in a very basic form without revealing its true potential! 

So what are these extra tricks?

 * More implicit typing
 * No unnecessary methods
 * Builder chaining (not just method chaining!)

Let's start with a basic builder pattern. We'll consider how to build a `Book` object:

```java
public final class Book {
    private final String author;
    private final String title;
    private final Optional<String> category;
    
    private Book(Builder builder) {
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
        final String author;
        final String title;
        final Optional<String> category = Optional.empty();
        
        Builder() {
        }
        
        public Builder author(String author) {
            this.author = author;
        }
        
        public Builder title(String title) {
            this.title = title;
        }
        
        public Book build() {
            return new Book(this);
        }      
}
```



