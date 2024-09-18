---
title: Rust domain events (proc macro)
date: 2024-09-17
---

How can we use serde to create clean domain events?

What we want
```json
{
    "Entity": "<EntityName>",
    "EntityId": <EntityId>,
    "EventType": <EventType>,
    "Data": <SomeData>
}
```

Requirements:
- The payload is serialized into the above format
- We can use rusts pattern matching to handle the events in process.


How can we accomplish this?


Given that we have the following enum structure:

```rust 

#[derive(Serialize)]
struct NoteId(u32);

#[derive(Serialize)]
enum NoteEvents {
    Created { id: NoteId, name: String },
    NameChanged { id: NoteId, name: String },
    Deleted { id: NoteId },
}

impl NoteEvents {
    pub fn get_id(&self) -> &NoteId {
        match self {
            NoteEvents::Created { id, .. } => id,
            NoteEvents::NameChanged { id, .. } => id,
            NoteEvents::Deleted { id } => id,
        }
    }
}


#[derive(Serialize)]
struct BlockId(String);

#[derive(Serialize)]
enum BlockEvents {
    Created { id: BlockId, name: String },
    Moved { id: BlockId, name: String },
    Deleted { id: BlockId },
}
impl BlockEvents {
    pub fn get_id(&self) -> &BlockId {
        match self {
            BlockEvents::Created { id, .. } => id,
            BlockEvents::Moved { id, .. } => id,
            BlockEvents::Deleted { id } => id,
        }
    }
}


#[derive(Serialize)]
enum InlineType {
    Bold,
    Italic,
    Underline,
}

#[derive(Serialize)]
struct TextInline {
    offset: u32,
    length: u32,
    r#type: InlineType,
}

#[derive(Serialize)]
struct TextId(i64);

#[derive(Serialize)]
enum TextEvents {
    Created { id: TextId, text: String },
    Updated { id: TextId, text: String },
    Deleted { id: TextId },
}
impl TextEvents {
    pub fn get_id(&self) -> &TextId {
        match self {
            TextEvents::Created { id, .. } => id,
            TextEvents::Updated { id, .. } => id,
            TextEvents::Deleted { id } => id,
        }
    }
}

#[derive(SerializeDomainEvent)]
enum DomainEntityEvent {
    Note(NoteEvents),
    Block(BlockEvents),
    Text(TextEvents),
}
```


We can accomplish the desired serialization by creating a proc macro `SerializeDomainEvent`, that manually implements the serde ´Serialize´ trait. 

___This code is not pretty, but after pulling my hair out for 2 days, I am just happy to have something that works!___

```rust 
#[proc_macro_derive(SerializeDomainEvent)]
pub fn dynamic_serialize_derive(input: TokenStream) -> TokenStream {
    let input = parse_macro_input!(input as DeriveInput);
    let name = input.ident;

    let data = match input.data {
        Data::Enum(data) => data,
        _ => panic!("SerializeDomainEntity only works with enums"),
    };

    let variants = data.variants.into_iter().map(|variant| {
        let variant_ident = variant.ident;

        quote! {
            #name::#variant_ident(inner) => {
                let mut map = serde_json::Map::new();
                map.insert("Entity".to_string(), serde_json::Value::String(stringify!(#variant_ident).to_string()));

                let inner_json = serde_json::to_value(inner)
                    .map_err(serde::ser::Error::custom)?;

                let inner_json = match inner_json {
                    serde_json::Value::Object(value) => value,
                    _ => panic!("Failed to parse the inner json element")
                };

                let event_type = inner_json.keys().next().unwrap();
                let event_data = inner_json.values().next().unwrap();

                // map.insert("EntityId", serde_json::to_value(inner.get_id()));

                map.insert("EventType".to_string(), serde_json::Value::String(event_type.to_string()));
                map.insert("Data".to_string(), event_data.clone());

                serde_json::Value::Object(map)
            }
        }
    });

    let expanded = quote! {
        impl serde::Serialize for #name {
            fn serialize<S>(&self, serializer: S) -> Result<S::Ok, S::Error>
            where
                S: serde::Serializer,
            {
                let json_value = match self {
                    #(#variants),*
                };
                json_value.serialize(serializer)
            }
        }
    };

    TokenStream::from(expanded)
}
```


Now, if we run our test, we can verify that our data is in the correct format:
```rust 
#[test]
fn serialize_test() {
    let data = DomainEntityEvent::Text(TextEvents::Updated {
        id: TextId(2222),
        text: "Hello World".to_owned(),
    });

    let serialized = serde_json::to_string(&data);

    println!("{:#?}", serialized);
    // Prints: "{\"Data\":{\"id\":2222,\"text\":\"Hello World\"},\"Entity\":\"Text\",\"EntityId\":2222,\"EventType\":\"Updated\"}"
}
```
