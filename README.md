# Token Object V2: A Comprehensive Analysis

The term "Token" usually refers to the fungible or non-fungible assets on-chain. As blockchain evolves, the standards and models that define tokens evolve too. In Aptos, `Token V2` refers to the upgraded non-fungible token standards. The original model, `Token V1`, uses resources as a way to represent tokens on-chain while the newer model uses objects. This change pushed the initial version beyond the limits allowing it to benefit from the object model. 

## Token V1: An Overview

The original token model in the Aptos blockchain, known as `Token V1`, uses resources as a way to represent tokens on-chain. This model has pros and cons.

### Pros

`Token V1` is storable in a resource on-chain, which can minimize the number of manual interactions with the module itself, and ensure immutability of the token's attributes. 

It also uses a `PropertyMap` to represent the token's traits on-chain. This allows for more extensibility, meaning that you can dynamically store properties of various data types to add more traits to the token down the road after its initial creation. This also provides serialization and deserialization of nested data fields, using the Binary Canonical Serialization (BCS) which allows for storing a data of different types.

### Cons

In this token model, data is stored within the resource account, meaning the token could be nested within multiple layers of data, leading to various issues, like querying the owner. 

The `PropertyMap` provides a lot of flexibility and robustness, but this comes at the cost of increased memory usage and computational overhead. 

## Token V2: A New Era

Token V2 refers to the upgraded non-fungible token standards within Aptos blockchain. The newer model uses objects.

### Token V2 components

Aptos token object model consists of the following modules:
- `collection.move`: Defines an object-based collection.
- `token.move`: Defines the core logic of an object-based token (token V2).
- `royalty.move`: Defines an object based royalty system. The royalty can be used for either a collection or an individual token. 
- `aptos_token.move`: Defines a basic token for no-code solutions, similar to the original token in the 0x3::token module. It has the fundamental token and collection functions,
allowing creators to control how tokens can change
and whether their freezable, burnable, and mintable. It also has
standard method to transfer objects and related events
as well as property type for metadata.
    - `property_map.move`: Offers universal metadata assistance for "AptosToken." It's a specific version of "SimpleMap" that ensures precise data types and conserves storage space by utilizing a constant u64 to signify types and storing values in the bcs format.

### Features of Token V2

Token V2 is built on top of the object model, meaning that the token is an object that is part of a collection (which is also an object). This offers several new capabilities while costs remain cheap. 

The object model separates ownership from data, meaning all object resources are top-level data, making them globally addressable and queryable. 

In terms of extensibility, you can add resources to an object even after it is initially been created, this allows for more extensibility and flexibility in the future. 

Composability wise, objects are so similar to accounts that they can even own other objects, this enables the creation of true on-chain object composability, the ability to store native, typed objects within one another. 

[Click to read more about the object model](https://noncegeek.medium.com/aptos-object-model-learning-move-0x01-550708f0fd33) 

### Token V2 Lifecycle

- An entity calls create on a top-level object type.
- The top-level object type calls create on its direct ancestor.
- This is repeated until the top-most ancestor is the `Object` struct, which defines the `create_object` function.
- The `create_object` generates a new address, stores an `Object` struct at that address within the `ObjectGroup` resource group, and returns back a `ConstructorRef`.
- The create functions called earlier can use the `ConstructorRef` to obtain a signer, store an appropriate struct within the `ObjectGroup` resource, make any additional modifications, and return the `ConstructorRef` back up the stack.
- In the object creation stack, any of the modules can define properties such as ownership, deletion, transferability, and mutability.


## Token V1 vs Token V2

| Token V1 | Token V2 |
| --- | --- |
| Based on the resource model | Based on the object model |
| Expensive to store its traits on-chain | Cheap to store its traits on-chain |
| Cannot own tokens | Can own tokens |
| Extensible | More Extensible |
| Data in some cases is not addressable neither queryable | Data is always addressable and queryable globally |

In conclusion, the evolution from Token V1 to Token V2 represents a significant leap forward in the Aptos blockchain's non-fungible token standards. The new model offers more flexibility, extensibility, and composability, paving the way for more complex and dynamic on-chain interactions.

## Token V2 in action: hero.move

there is no better way to learn a language from its pioneers. Aptos labs provides exemples on how to use the new token model in the [aptos-core repository](). 

As of this writing, `hero.move` is the simplest yet complete example of token V2. 

The smart contract allows to create a hero, add properties to it, and transfer to another account not only the hero itself, but also its attributes:

``` Rust
struct Hero has key {
        armor: Option<Object<Armor>>,
        gender: String,
        race: String,
        shield: Option<Object<Shield>>,
        weapon: Option<Object<Weapon>>,
        mutator_ref: token::MutatorRef,
    }
```

In terms of tokens, `hero` is a token and `armor`, `gender`, `race`, `shield`, `weapon` are the tokens within `hero` that represent its attributes;
- `Hero` can own `armor`, `shield`, and `weapon`.
- `Armor`, `shield`, and `weapon` can have `gems`.
- `Gems` can change `attack` and `defense` values of `armor`, `shield`, and `weapon`. They also have a `magic` attribute.

The lifecycle is as follows:

### 1. Initialing the module

Before creating a hero, we need to initialize the module.

```Rust
fun init_module(account: &signer) {
    let collection = string::utf8(b"Hero Quest!");
    collection::create_unlimited_collection(
        account,
        string::utf8(b"collection description"),
        collection,
        option::none(),
        string::utf8(b"collection uri"),
    );

    let on_chain_config = OnChainConfig {
        collection,
    };
    move_to(account, on_chain_config);
    }
```

This is done by creating a collection and storing it in the `OnChainConfig` resource that has the `key` keyword to store it globally. This will an easy query for the collection later on.

```Rust
struct OnChainConfig has key {
        collection: String,
    }
```

### 2. Creating a hero

``` Rust
public fun create_hero(
    creator: &signer,
    description: String,
    gender: String,
    name: String,
    race: String,
    uri: String,
): Object<Hero> acquires OnChainConfig {
    let constructor_ref = create(creator, description, name, uri);
    let token_signer = object::generate_signer(&constructor_ref);

    let hero = Hero {
        armor: option::none(),
        gender,
        race,
        shield: option::none(),
        weapon: option::none(),
        mutator_ref: token::generate_mutator_ref(&constructor_ref),
    };
    move_to(&token_signer, hero);

    object::address_to_object(signer::address_of(&token_signer))
}
```

`hero` in essence is an object. the `create_hero` function creates a hero and returns itbefore closing the function.

```Rust
public fun create_hero(
    
): Object<Hero>

...

object::address_to_object()
```

the freshly created `hero` data is stored in the resource account.

```Rust
let hero = Hero {
armor: option::none(),
gender,
race,
shield: option::none(),
weapon: option::none(),
mutator_ref: token::generate_mutator_ref(&constructor_ref),
};
move_to(&token_signer, hero);
```
`constructor_ref` is a reference to the `hero` object. It generates a signer that is used to complete the store operation.

`create_hero` will finally be wrapped up in a entry function `mint_hero`.

```Rust
entry fun mint_hero(
    account: &signer,
    description: String,
    gender: String,
    name: String,
    race: String,
    uri: String,
) acquires OnChainConfig {
    create_hero(account, description, gender, name, race, uri);
}
```

### 3. Creating shield, armor, weapon, and gem

Creating these objects follow the same pattern as creating a hero.

```Rust
    public fun create_weapon(
        creator: &signer,
        attack: u64,
        description: String,
        name: String,
        uri: String,
        weapon_type: String,
        weight: u64,
    ): Object<Weapon> acquires OnChainConfig {
        let constructor_ref = create(creator, description, name, uri);
        let token_signer = object::generate_signer(&constructor_ref);

        let weapon = Weapon {
            attack,
            gem: option::none(),
            weapon_type,
            weight,
        };
        move_to(&token_signer, weapon);

        object::address_to_object(signer::address_of(&token_signer))
    }
```

### 4. Adding/removing attributes to hero

The steps for adding an attribute to a hero (or a gem to an armor/weapon/shield) are straightforward: 
1. get the object.
2. fill the object in the hero object.
3. transfer the object to the new owner.

```Rust
public fun weapon_equip_gem(owner: &signer, weapon: Object<Weapon>, gem: Object<Gem>) acquires Weapon {
    let weapon_obj = borrow_global_mut<Weapon>(object::object_address(&weapon));
    option::fill(&mut weapon_obj.gem, gem);
    object::transfer_to_object(owner, gem, weapon);
}
```

Removing an attribute is also the same, except that we need to check if the attribute is present in the hero object.

```Rust
public fun weapon_unequip_gem(owner: &signer, weapon: Object<Weapon>, gem: Object<Gem>) acquires Weapon {
    let weapon_obj = borrow_global_mut<Weapon>(object::object_address(&weapon));
    let stored_gem = option::extract(&mut weapon_obj.gem);
    assert!(stored_gem == gem, error::not_found(EINVALID_GEM_UNEQUIP));
    object::transfer(owner, gem, signer::address_of(owner));
}
```

## References

- [Token v1 codebase](https://github.com/aptos-labs/aptos-core/tree/main/aptos-move/framework/aptos-token)
- [Token v2 codebase](https://github.com/aptos-labs/aptos-core/tree/main/aptos-move/framework/aptos-token-objects)
- [Object model Article](https://noncegeek.medium.com/aptos-object-model-learning-move-0x01-550708f0fd33)
- [Computing Gas](https://aptos.dev/concepts/base-gas/#instruction-gas)
- [hero.move](https://github.com/aptos-labs/aptos-core/tree/main/aptos-move/move-examples/token_objects/hero)
