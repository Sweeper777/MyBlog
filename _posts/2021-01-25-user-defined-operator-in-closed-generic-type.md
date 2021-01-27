---
tags: c# generics
op_link: https://stackoverflow.com/q/65880133/5133585
op_profile_link: https://stackoverflow.com/users/66490/trayman
op_name: "TrayMan"
---

### Premise

OP defined an operator on any two `Value<T>`, that produces a `Result<T, T>`:

{% highlight c# %}
class Result<T1, T2> {}

class Value<T> {
   public static Result<T,T> operator+(Value<T> a, Value<T> b) { ... }
}
{% endhighlight %}

However, they are not satisified and wanted to use this operator on any two `Value<T>` and `Value<U>`, where `T` and `U` can be different types.

I thought, ok, you need another type parameter. Let's add that:

{% highlight c# %}
class Value<T> {
   public static Result<T,U> operator+<U>(Value<T> a, Value<U> b) { ... }
}
{% endhighlight %}

I quickly tried that on dotnetfiddle, and nope, it didn't work. I got quite a lot of compiler errors telling me "`<some token>` expected". It seems like I have upset the parser with that, so I guess generic operators are not a thing then.

Rather than just showing what doesn't compile, I thought I should show the syntax of user defined operators, which I think is more convincing. I navigated to the [Classes -> Operators](https://docs.microsoft.com/en-us/dotnet/csharp/language-reference/language-specification/classes#operators) section of the language spec, and saw that the syntax of operator declarations are actually specified in a very different way from method declarations. Specifically, it's very restrictive, and almost has no optional parts, e.g.

```
unary_operator_declarator
    : type 'operator' overloadable_unary_operator '(' type identifier ')'
    ;
binary_operator_declarator
    : type 'operator' overloadable_binary_operator '(' type identifier ',' type identifier ')'
    ;
```

I couldn't believe that the parameter list is literally just hardcoded in like that. Compare that to the [parameter list of a method](https://docs.microsoft.com/en-us/dotnet/csharp/language-reference/language-specification/classes#method-parameters):

```
formal_parameter_list
    : fixed_parameters
    | fixed_parameters ',' parameter_array
    | parameter_array
    ;

fixed_parameters
    : fixed_parameter (',' fixed_parameter)*
    ;

fixed_parameter
    : attributes? parameter_modifier? type identifier default_argument?
    ;

default_argument
    : '=' expression
    ;

parameter_modifier
    : 'ref'
    | 'out'
    | 'this'
    ;

parameter_array
    : attributes? 'params' array_type identifier
    ;
```

I had always thought their syntax would be a superset of that of method declarations, and the restrictions that applies to operator declarations would be specified not in the syntax, but through the prose in the spec.

With such a limited syntax, operators certainly can't be generic.

Just when I was about to write my answer, I started to feel that "operators can't be generic" isn't that strong of an argument, because making the operator generic might not be the only way to achieve what the OP wants. To show that this is impossible, I need to find some feature that a solution _must_ make use of, and show that feature is not available for operators.

I brainstormed for a bit, and thought, type inference! In an expression such as `a + b`, where `a` and `b` are of type `Value<string>` and `Value<int>` respectively, in nowhere are we specifying the result type's type parameters, so type inference must be applied. Searching the [Expressions](https://docs.microsoft.com/en-us/dotnet/csharp/language-reference/language-specification/expressions) chapter, there are only mentions of type inference with regards to methods, not operators, which shows that type inference doesn't work on operators.

Indeed, if operators can't be generic, why would they need type inference? :D