# Groot
[![Carthage compatible](https://img.shields.io/badge/Carthage-compatible-4BC51D.svg?style=flat)](https://github.com/Carthage/Carthage) ![CocoaPods compatible](https://img.shields.io/cocoapods/v/Groot.svg)

Groot provides a simple way of serializing Core Data object graphs from or into JSON.

Groot uses [annotations](Documentation/Annotations.md) in the Core Data model to perform the serialization and provides the following features:

1. Attribute and relationship mapping to JSON key paths.
2. Value transformation using named `NSValueTransformer` objects.
3. Object graph preservation.
4. Support for entity inheritance

## Installing Groot

##### Using CocoaPods

Add the following to your `Podfile`:

``` ruby
pod ‘Groot’
```

Or, if you need to support iOS 6 / OS X 10.8:

``` ruby
pod ‘Groot/ObjC’
```

Then run `$ pod install`.

If you don’t have CocoaPods installed or integrated into your project, you can learn how to do so [here](http://cocoapods.org).

##### Using Carthage

Add the following to your `Cartfile`:

```
github “gonzalezreal/Groot”
```

Then run `$ carthage update`.

Follow the instructions in [Carthage’s README](https://github.com/Carthage/Carthage#adding-frameworks-to-an-application]) to add the framework to your project.

You may need to set **Embedded Content Contains Swift Code** to **YES** in the build settings for targets that only contain Objective-C code.

## Getting started

Consider the following JSON describing a well-known comic book character:

```json
{
    "id": "1699",
    "name": "Batman",
    "real_name": "Bruce Wayne",
    "powers": [
        {
            "id": "4",
            "name": "Agility"
        },
        {
            "id": "9",
            "name": "Insanely Rich"
        }
    ],
    "publisher": {
        "id": "10",
        "name": "DC Comics"
    }
}
```

We could translate this into a Core Data model using three entities: `Character`, `Power` and `Publisher`.

<img src="https://cloud.githubusercontent.com/assets/373190/6988401/5346423a-da51-11e4-8bf1-a41da3a7372f.png" alt="Model" width=600 height=334/>


### Mapping attributes and relationships

Groot relies on the presence of certain key-value pairs in the user info dictionary associated with entities, attributes and relationships to serialize managed objects from or into JSON. These key-value pairs are often referred in the documentation as [annotations](Documentation/Annotations.md).

In our example, we should add a `JSONKeyPath` in the user info dictionary of each attribute and relationship specifying the corresponding key path in the JSON:

* `id` for the `identifier` attribute,
* `name` for the `name` attribute,
* `real_name` for the `realName` attribute,
* `powers` for the `powers` relationship,
* `publisher` for the `publisher` relationship,
* etc.

Attributes and relationships that don't have a `JSONKeyPath` entry are **not considered** for JSON serialization or deserialization.

### Value transformers

When we created the model we decided to use `Integer 64` for our `identifier` attributes. The problem is that, for compatibility reasons, the JSON uses strings for `id` values.

We can add a `JSONTransformerName` entry to each `identifier` attribute's user info dictionary specifying the name of a value transformer that converts strings to numbers.

Groot provides a simple way for creating and registering named value transformers:

```swift
// Swift

func toString(value: Int) -> String? {
    return "\(value)"
}

func toInt(value: String) -> Int? {
    return value.toInt()
}

NSValueTransformer.setValueTransformerWithName("StringToInteger", transform: toString, reverseTransform: toInt)
```

```objc
// Objective-C

[NSValueTransformer grt_setValueTransformerWithName:@"StringToInteger" transformBlock:^id(NSString *value) {
    return @([value integerValue]);
} reverseTransformBlock:^id(NSNumber *value) {
    return [value stringValue];
}];
```

### Object graph preservation

To preserve the object graph and avoid duplicating information when serializing managed objects from JSON, Groot needs to know how to uniquely identify your model objects.

In our example, we should add an `identityAttribute` entry to the `Character`, `Power` and `Publisher` entities user dictionaries with the value `identifier`.

For more information about annotating your model have a look at [Annotations](Documentation/Annotations.md).

### Serializing from JSON

Now that we have our Core Data model ready we can start adding some data.

```swift
// Swift

let batmanJSON: JSONObject = [
    "name": "Batman",
    "id": "1699",
    "powers": [
        [
            "id": "4",
            "name": "Agility"
        ],
        [
            "id": "9",
            "name": "Insanely Rich"
        ]
    ],
    "publisher": [
        "id": "10",
        "name": "DC Comics"
    ]
]

let batman: Character = objectFromJSONDictionary(batmanJSON,
        inContext: context, mergeChanges: false, error: &error)
```

```objc
// Objective-C

Character *batman = [GRTJSONSerialization objectWithEntityName:@"Character" fromJSONDictionary:batmanJSON inContext:self.context error:&error];
```

If we want to update the object we just created, Groot can merge the changes for us:

```objc
// Objective-C

NSDictionary *updateJSON = @{
    @"id": @"1699",
    @"real_name": @"Bruce Wayne",
};

// This will return the previously created managed object
Character *batman = [GRTJSONSerialization objectWithEntityName:@"Character" fromJSONDictionary:updateJSON inContext:self.context error:&error];
```

#### Serializing relationships from identifiers

Suppose that our API does not return full objects for the relationships but only the identifiers.

We don't need to change our model to support this situation:

```objc
// Objective-C

NSDictionary *batmanJSON = @{
    @"name": @"Batman",
    @"real_name": @"Bruce Wayne",
    @"id": @"1699",
    @"powers": @[@"4", @"9"],
    @"publisher": @"10"
};

Character *batman = [GRTJSONSerialization objectWithEntityName:@"Character" fromJSONDictionary:batmanJSON inContext:self.context error:&error];
```

The above code creates a full `Character` object and the corresponding relationships pointing to `Power` and `Publisher` objects that just have the identifier attribute populated.

We can import powers and publisher from different JSON objects and Groot will merge them nicely:

```objc
// Objective-C

NSArray *powersJSON = @[
    @{
        @"id": @"4",
        @"name": @"Agility"
    },
    @{
        @"id": @"9",
        @"name": @"Insanely Rich"
    }
];

[GRTJSONSerialization objectsWithEntityName:@"Power" fromJSONArray:powersJSON inContext:self.context error:&error];

NSDictionary *publisherJSON = @{
    @"id": @"10",
    @"name": @"DC Comics"
};

[GRTJSONSerialization objectWithEntityName:@"Publisher" fromJSONDictionary:publisherJSON inContext:self.context error:&error];
```

For more serialization methods check [GRTJSONSerialization.h](Groot/GRTJSONSerialization.h) and [Groot.swift](Groot/Groot.swift).

### Entity inheritance

Groot supports entity inheritance via the [entityMapperName](Documentation/Annotations.md#entitymappername) annotation.

If you are using SQLite as your persistent store, Core Data implements entity inheritance by creating one table for the parent entity and all child entities, with a superset of all their attributes. This can obviously have unintended performance consequences if you have a lot of data in the entities, so use this feature wisely.

### Serializing to JSON

Groot provides methods to serialize managed objects back to JSON:

```swift
// Swift

let JSONDictionary = JSONDictionaryFromObject(batman)
```

```objc
// Objective-C

NSDictionary *JSONDictionary = [GRTJSONSerialization JSONDictionaryFromObject:batman];
```

For more serialization methods check [GRTJSONSerialization.h](Groot/GRTJSONSerialization.h) and [Groot.swift](Groot/Groot.swift).

## Contact

[Guillermo Gonzalez](http://github.com/gonzalezreal)  
[@gonzalezreal](https://twitter.com/gonzalezreal)

## License

Groot is available under the [MIT license](LICENSE.md).