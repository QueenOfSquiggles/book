<!--
  ~ Copyright (c) godot-rust; Bromeon and contributors.
  ~ This Source Code Form is subject to the terms of the Mozilla Public
  ~ License, v. 2.0. If a copy of the MPL was not distributed with this
  ~ file, You can obtain one at https://mozilla.org/MPL/2.0/.
-->

# Registering classes

Classes are the backbone of data modeling in Godot. If you want to build complex user-defined types in a type-safe way, you won't get around
classes. Arrays, dictionaries and simple types only get you so far, and overusing them defeats the purpose of using a statically typed language.

Rust makes class registration straightforward. As mentioned before, Rust syntax is used as a baseline, with gdext-specific additions.


See also [GDScript reference for classes][godot-gdscript-classes].


## Table of contents

<!-- toc -->


## Defining a Rust struct

In Rust, Godot classes are represented by structs. Structs are defined as usual and can contain any number of fields. To register them with
Godot, you need to derive the `GodotClass` trait.

```admonish info title="GodotClass trait"
The `GodotClass` trait marks all classes known in Godot. It is already implemented for engine classes, for example `Node` or `Resource`.
If you want to register your own classes, you need to implement `GodotClass` as well.

`#[derive(GodotClass)]` streamlines this process and takes care of all the boilerplate.  
See [API docs][api-derive-godotclass] for detailed information.
```

Let's define a simple class named `Monster`:

```rs
#[derive(GodotClass)]
struct Monster {
    name: String,
    hitpoints: i32,
}
```

That's it. Immediately after compiling, this class becomes available in Godot through hot reloading (before Godot 4.2, after restart).
It won't be very useful yet, but the above definition is enough to register `Monster` in the engine.

```admonish info title="Auto-registration"
`#[derive(GodotClass)]` _automatically_ registers the class -- you don't need an explicit `add_class()` registration call
or a central list mentioning all classes.

The proc-macro internally registers the class in such a list at startup time.
```


## Selecting a base class

By default, the base class of a Rust class is `RefCounted`. This is consistent with GDScript when you omit the `extends` keyword.

`RefCounted` is quite useful for data bundles. As implied by the name, it allows sharing instances tracked by a reference counter;
as such, you don't need to worry about memory management. `Resource` is a subclass of `RefCounted` and is useful for data that needs to be
serialized to the filesystem.

However, if you want your class to be part of the scene tree, you need to use `Node` (or one of its derived classes) as a base class.

Here, we use a more concrete node type, `Node3D`. This is done by specifying `#[class(base=Node3D)]` on the struct definition:

```rs
#[derive(GodotClass)]
#[class(base=Node3D)]
struct Monster {
    name: String,
    hitpoints: i32,
}
```


## The base field

Since Rust does not have inheritance, we need to use composition to achieve the same effect. gdext provides a `Base<T>` type, which lets us
store the instance of the Godot superclass (base class) as a field in our `Monster` class.

```rs
#[derive(GodotClass)]
#[class(base=Node3D)]
struct Monster {
    name: String,
    hitpoints: i32,
    
    #[base]
    base: Base<Node3D>,
}
```

The `#[base]` attribute is currently necessary (as of January 2024), but will likely disappear in the future.

The important part is the `Base<T>` type. `T` must match the base class you specified in the `#[class(base=...)]` attribute.
You can also use the associated type `Self::Base` for `T`.

When you declare a base field in your struct, you can access the `Node` API through provided methods `self.base()` and `self.base_mut()`, but
more on this later.


## Default constructor

The constructor of any `GodotClass` object is called `init`. It is necessary to instantiate the object in Godot -- which is used by the scene
tree or when you write `Monster.new()` in GDScript.

There are two options to define the constructor: let gdext generate it or define it manually.


### Library-generated

You can use `#[class(init)]` to generate a constructor for you. This is limited to simple cases, and it calls `Default::default()` for each field.

```rs
#[derive(GodotClass)]
#[class(init, base=Node3D)]
struct Monster {
    name: String,
    hitpoints: i32,
    
    #[base]
    base: Base<Node3D>,
}
```

The above definition enables `Monster.new()` in GDScript, but any instance created like this would have an empty string as `name` and
`0` as `hitpoints`.


### Manually defined

```admonish info title="Interface traits"
Each engine class comes with an associated trait, which has the same name but is prefixed with the letter `I`, for "Interface".
The trait has no required functions, but you can override any functions to customize the behavior towards Godot.

Any `impl` block for the trait must be annotated with the `#[godot_api]` attribute macro.
```

```admonish info title="godot_api macro"
The attribute proc-macro `#[godot_api]` is applied to `impl` blocks and marks their items for registration.
It takes no arguments.

See [API docs][api-godot-api] for detailed information.
```

In our case, the `Node3D` comes with the `INode3D` trait.

We can provide a manually-defined constructor by overriding the trait's associated function `init`:

```rs
#[derive(GodotClass)]
#[class(base=Node3D)] // No init here, since we define it ourselves.
struct Monster {
    name: String,
    hitpoints: i32,
    
    #[base]
    base: Base<Node3D>,
}

#[godot_api]
impl INode3D for Monster {
    fn init(base: Base<Node3D>) -> Self {
        Self {
            name: "Nomster".to_string(),
            hitpoints: 100,
            base,
        }
    }
}
```

As you can see, the `init` function takes a `Base<Node3D>` as its one and only parameter. This is the base class instance, which is typically
just forwarded to its corresponding field in the struct, here `base`.

The `init` method always returns `Self`. You may notice that this is currently the only way to construct a `Monster` instance. As soon as your
struct contains a base field, you can no longer provide your own constructor, as you can't provide a value for that field. This is by design and
ensures that _if_ you need access to the base, that base comes from Godot directly.

However, fear not: you can still provide all sorts of constructors, they just need to go through dedicated functions that internally call `init`.
More on this topic in the next chapter.


## Conclusion

You learned how to define a Rust class and register it with Godot. You now know how to select a base class and how to define a constructor.
The next chapter will allow you to implement logic by providing functions.

[api-derive-godotclass]: https://godot-rust.github.io/docs/gdext/master/godot/register/derive.GodotClass.html
[api-godot-api]: https://godot-rust.github.io/docs/gdext/master/godot/register/attr.godot_api.html
[godot-gdscript-classes]: https://docs.godotengine.org/en/stable/tutorials/scripting/gdscript/gdscript_basics.html#classes
