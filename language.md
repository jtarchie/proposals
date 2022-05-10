# Inspiration

The design of this language is inspired by my appreciation of the Ruby syntax.

The motiviation is not to have static analyzer of Ruby code. This is following
more in the steps of [MacRuby](http://macruby.org/).

The intention the a subset of Ruby that can be transpiled to related Golang.
Some syntax still needs to be work out, so in the interim is losely copying from
[Crystal](https://crystal-lang.org/).

**WARNING: This design and syntax has not been validated. Some portions my be
conflicting, therefore impossible to implement.**

# Types

All values must have type definitions. At this time there will be no gradual
typing or heavy type inference.

## Variable assigment

On variable assigment, the types can be infered on the function return value.

```ruby
a = String.new # a would have type String
a = ""         # a would have type String
```

A variable in Ruby is a pointer to value, not the actual assigment of the value.
This has the side effect of a variable having more than one assignble value,
therefore type.

To minimize scope, a variable cannot be assigned a different type in scoping.
The following would be invalid and raise a type mismatch.

```ruby
a = ""  # a would have type String
# some more code
a = 123 # type mismatch would occur as `a` is previously String
```

### Multi assigment

Multi assignment allows variables to be assigned in a single line. The left
handside of assignment is the variable definition and the right handside is the
value assignment.

In this example,

```ruby
a, b = 1, "one"
```

The assigned value of `a` would be of type `Integer`. The assigned value of `b`
would be of type `String`.

## Functions

Function declaration would be handle via the reserved keyword `def`. Ruby has
other conventions (some dynamic) for defining functions and those are beyond
this scope.

Example declaration of function:

```ruby
def adult?(age: Integer) : Boolean
  age > 18
end
```

**NOTE: Ruby classes (like `Integer`) are a value. When the class name is used
soley in an argument or return type, it is assumed that type is meant to be an
instance (`Integer.new`) of class.**

### Arguments

Arguments are always declared as
[keyword arguments](https://robots.thoughtbot.com/ruby-2-keyword-arguments).
Their declaration is used to defined the required type of the argument.

In this example,

```ruby
def adult?(age: Integer) : Boolean
  age > 18
end
```

The `age` argument is assigned to the type `Integer`. With this declaration,
there is no _zero_ value assumed. When the `adult?` method is invoked the `age`
keyword is required to passed in and be of type `Integer`.

#### Default values

A default value can be assigned by applying an instance of a value.

In this example,

```ruby
def adult?(age: 0) : Boolean
  age > 18
end
```

The `age` arugment is assigned the type `Integer`. With this declaraction, the
default value will be `0`. When the `adult?` method is invoked the `age` keyword
is not required to be passed in. If on invocation of `adult?` the `age` keyword
is passed it will be required to be of type `Integer`.

### Return Values

**NOTE: Ruby does not have a syntax for extending a function defintion that
could resemble a _return type_. An extension of the Ruby language is being made
to support this feature.**

A return type is specified at the end of `def` declaration. It validates the
`return` (implicit or explicit) is the correct type.

In this example,

```ruby
def adult?(age: 0) : Boolean
  age > 18
end

a = adult?(21)
```

The return value is `Boolean`. The final evaulated statement (an implicit
`return`) must of value `Boolean`. The assigned value of `a` would be of type
`Boolean`.

#### Return with multiple values (not types)

Multiple return values can be handle by encapsulating the types (similar to Go)
in parathesis `()`.

In this example,

```ruby
def adult(age: 0) : (Boolean, Integer)
  age > 18, 18 - age
end

a, b = adult(55)
```

The return values are `Boolean` then `Integer`. The final evaulated statement
(an implicit `return`) must of value `Boolean` and `Integer`. The assigned value
of `a` would be of type `Boolean` and `b` would be of type `Integer`.

## Blocks

A block is defined by the `{}` or `do...end` keywords. The conventions of
argument types and return types are similar to functions.

```ruby
a = some_func do |age: Integer| : Boolean
# some code
end
```

```ruby
a = some_func { |age: Integer| : Boolean code... }
end
```

## Native Types

- `String`, with an instance being declared with `""` or `''`.
- `Integer`, with an instance being declared as unsigned/signed integer -- ie
  `-1`, `0`, `+100`.
- `Float`, with an instance being declared as unsigned/singed floating point --
  ie `-1.1`, `0.111`, `+100.9`.
- `Boolean`, wiht an instance being declared as `true` or `false`.

## Array

Defining an array as type will always be an explicit declaration.

```ruby
def dollars(bills: Array(Integer)) : Integer
  bills.sum
end
```

The syntax of `Array(T)` where `T` is the type of values that can be in the
array.

To initialize a value of an array and have the compiler infer its type.

```ruby
a = [1, 2, 3]
```

The type of `a` is `Array(Integer)`.

This means that `[]` array cannot be declared as the type cannot be inferred.

## Hash

TODO

## Classes

TODO

## Modules

TODO

## Null/Void

TODO

# Macros

Ruby supports runtime evaluation of code. For example, being able to add
accessor methods using `attr_reader`/`attr_writer`.

The reserved keyword `macro` can be used for compile time definition and AST
expansion. The `quote` command can parse code and return the associated AST
nodes (from
[Elixir](https://elixirschool.com/en/lessons/advanced/metaprogramming/#quote)).
This may require a dynamic version of Ruby to be used for compile time
evaluation.

```ruby
macro :attr_accessor do |pointer: AST| : AST 
  attr_name = pointer.children[0].value.to_s
  attr_type = pointer.children[1].value.to_s
  return quote(%{
    def #{attr_name}() : #{attr_type}
      return @#{attr_name}
    end
    
    def #{attr_name}=(v: #{attr_type}) : #{attr_type}
      @#{attr_name} = v
    end 
  })
end

# ...some place else...

class Person
  attr_accessor :name, String
end
```

The following declaration in a class will be expanded into:

```ruby
class Person
  def name() : String
    return @name
  end
  
  def name=(v: String) : String
    @name = v
  end
end
```

A macro can be used anywhere inline with your code to expand the AST. They have
no context awareness, so `attr_accessor` can be used outside of a `class`
statement, which may cause invalid syntax to be generated.

Macros are scoped to the file they are declared in. This to limit macro
collision across files. A mechanism for sharing macros is being thought through.

# Golang Interoperability

Interacting with underlying golang features can be done at the standard library
level. This behaviour will be similar to
[Elixir to Erlang](https://elixirschool.com/en/lessons/advanced/erlang).

## Method invocation

The standard library can be referenced using a symbol and the appropriate
method. If the arguments being passed in are mapped to similar types in the
language, it should Just Work™. For example, a Ruby `String` should map to the
golang `string`.

```ruby
:fmt.Println("Hello, World!")
:"net/http".Get("http://google.com")
```

Hopefully this enables light calling to golang functions, so some of the
compiler can be self hosted.

## Handling Types

Golang standard types should have a 1-to-1 mapping to the Ruby types.

- `string` to `String`
- `int` to `Integer`
- `float` to `Float`
- `bool` to `Boolean`

**NOTE: There is no proposal for handling a golang `struct` or custom type at
the moment. It is recognized as something that will be useful though when
handling things such as "net/http" responses.**

# Known Missing Features

- Splats
- May not be fully self hosted
- there is no `Any` type
