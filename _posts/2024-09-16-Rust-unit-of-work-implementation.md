---
title: WIP - Rust unit of work implementation
date: 2024-09-16
---

In my previous post, i wrote that i needed to refactor my current unit of work implementation. 


To recap, how do the desired implementation look like

```rust 
async fn test() {
  let uow = generate_uow();

  let text_id = TextId::new();
  let text = Text::new(text_id, "Hello World, This is a change".to_owned());

  uow.text.save(text).await?;

  uow.commit().await?;

  let text = uow.text.get_by_id(text_id).await?;

  uow.text.delete_by_id(text_id).await?;

  uow.commit().await?;
}
```

What are the requirements:
- The persisting of data should happen through events.
  - By having all data changes happen through events, we have a garantee that all modification on the entity will raise an event.
  - This allows us to easaly switch to Event Sourcing if this is deamed nessasary. But, as of now, we currently have no real need for it, and therefore i would like to avoid the overhead that comes with it.
- Events are emitted after we have commited/saved the unit of work.
  - While inside a unit of work, we dont want to commit any events, as this could potentially cause data inconsistencies. 
- The api should be clean, and have minimal boilerplate.
- An event may have multiple handlers


Lets setup some basic infrastructure, to give us something to work with:
```rust
pub trait EventQuery {
    fn get_query(self) -> Query<'static, Sqlite, SqliteArguments<'static>>;
}

#[async_trait]
pub trait Aggregate {
    type Id;
    type Event: EventQuery + Clone;
    type Error;

    fn get_events(&self) -> Vec<Self::Event>;
}

#[async_trait]
pub trait Repository<Entity: Aggregate> {
    async fn generate_id(&self) -> Entity::Id;
    async fn get_by_id(&self, id: Entity::Id) -> Result<Option<Entity>, Entity::Error>;

    async fn save(&self, entity: Entity) -> Result<(), Entity::Error>;
    async fn delete(&self, entity: Entity) -> Result<(), Entity::Error>;
}
```


The idea, is that all entities, will be responsable for their own events. In practice, it would look something like this:

```
struct Text {
    events: Vec<TextEvent>,

    id: TextId,
    text: String,
}

impl Aggregate for Text {
    type Id = TextId;
    type Event = TextEvent;
    type Error = String;

    fn get_events(&self) -> Vec<Self::Event> {
        self.events
    }
}

impl Text {
    pub fn new(id: TextId, text: String) -> Self {
        Self {
            id,
            text,
            events: vec![TextEvent::Created { id, text }],
        }
    }

    pub fn update(&mut self, text: String) {
        self.text = text;
        self.events.push(TextEvent::Updated { id: self.id, text });
    }
}
```

What i am unsure about however is, the repository should be implemented.
Since we cant emit the events we need to store them until the transaction is committed.
