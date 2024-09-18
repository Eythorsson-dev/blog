---
title: Rust unit of work implementation
date: 2024-09-18
---

In a [previous post](https://eythorsson-dev.github.io/blog/2024/09/15/Journal-Data-consistancy.html), I wrote that I needed to refactor my current unit of work implementation. After being side tracked by [domain events](https://eythorsson-dev.github.io/blog/2024/09/17/Rust-Domain-Events.html), I have now gotten around to implementing unit of work. 


Before we jump into coding, let's specify how we want to use our implementation to work. As any good developer, we will do this in the form of tests:

```rust 
#[tokio::test]
async fn can_create() {
    let config = CoreConfig::load_from_env().unwrap();
    let pool = establish_connection(&config.DB_URL).await.unwrap();

    let uow = UnitOfWork::new(pool);

    let id = uow.text.generate_id().await;
    let text = Text::new(id, "Hello World");

    uow.text.save(text).await.unwrap();

    uow.commit().await.unwrap();

    let text = uow.text.get_by_id(id).await
        .unwrap() // Unwrap the error
        .unwrap() // Unwrap the option
    ;

    assert_eq!(text.text, "Hello World");
}

#[tokio::test]
async fn can_update() {
    let config = CoreConfig::load_from_env().unwrap();
    let pool = establish_connection(&config.DB_URL).await.unwrap();

    let uow = UnitOfWork::new(pool);

    let id = uow.text.generate_id().await;
    let text = Text::new(id, "Hello World");

    uow.text.save(text).await.unwrap();

    uow.commit().await.unwrap();

    let mut text = uow.text.get_by_id(id).await
        .unwrap() // Unwrap the error
        .unwrap() // Unwrap the option
    ;


    text.update("Foo bar baz");

    uow.text.save(text).await.unwrap();

    uow.commit().await.unwrap();

    let text = uow.text.get_by_id(id).await
        .unwrap() // Unwrap the error
        .unwrap() // Unwrap the option
    ;
    assert_eq!(text.text, "Foo bar baz");
}

#[tokio::test]
async fn can_delete() {
    let config = CoreConfig::load_from_env().unwrap();
    let pool = establish_connection(&config.DB_URL).await.unwrap();

    let uow = UnitOfWork::new(pool);

    let id = uow.text.generate_id().await;
    let text = Text::new(id, "Hello World");

    uow.text.save(text).await.unwrap();

    uow.commit().await.unwrap();

    let text = uow.text.get_by_id(id).await
        .unwrap() // Unwrap the error
        .unwrap() // Unwrap the option
    ;
    uow.text.delete(text).await.unwrap();

    uow.commit().await.unwrap();

    let text = uow.text.get_by_id(id).await
        .unwrap() // Unwrap the error
    ;

    assert_eq!(None, text);
}
```

What are the requirements:
- The persisting of data should happen through events.
  - By having all data changes happen through events, we have a guarantee that all modification on the entity will raise an event.
  - This allows us to easily switch to Event Sourcing if this is deemed necessary. But, as of now, we currently have no real need for it, and therefore I would like to avoid the overhead that comes with it.
- Events can be emitted to an event bus after the unit of work is committed/saved.
  - While inside a unit of work, we don't want to commit any events, as this could potentially cause data inconsistencies. 
- The API should be clean, and have minimal boilerplate.
- An event may have multiple handlers


Let's set up some basic infrastructure, to give us something to work with:
```rust
pub trait EventQuery {
    fn get_query(self) -> Query<'static, Sqlite, SqliteArguments<'static>>;
}

#[async_trait]
pub trait Aggregate {
    type Id;
    type Event: EventQuery + Clone;
    type Error;

    fn get_events(self) -> Vec<Self::Event>;
}

#[async_trait]
pub trait Repository<Entity: Aggregate> {
    async fn generate_id(&self) -> Entity::Id;
    async fn get_by_id(&self, id: Entity::Id) -> Result<Option<Entity>>;

    async fn save(&self, entity: Entity) -> Result<()>;
    async fn delete(&self, entity: Entity) -> Result<()>;
}
```


Once that is in place, it's time to create our aggregate:

```rust
// I dont like poluting our structs like this. Idealy, the domain should not know
// that we are using sqlx...
#[derive(Serialize, Clone, Copy, sqlx::Type, PartialEq, Eq, Debug)]
#[sqlx(transparent)]
pub struct TextId(Uuid);

#[derive(PartialEq, Eq, Debug)]
pub struct Text {
    events: Vec<TextEvent>,

    pub id: TextId,
    pub text: String,
}

impl Aggregate for Text {
    type Id = TextId;
    type Event = TextEvent;
    type Error = String;

    fn get_events(self) -> Vec<Self::Event> {
        self.events
    }
}

impl Text {
    pub fn new(id: TextId, text: &str) -> Self {
        Self {
            id,
            text: text.to_owned(),
            events: vec![TextEvent::Created {
                id,
                text: text.to_owned(),
            }],
        }
    }

    pub fn update(&mut self, text: &str) {
        self.text = text.to_owned();

        self.events.push(TextEvent::Updated {
            id: self.id,
            text: text.to_owned(),
        });
    }
}


#[derive(Serialize, Clone, PartialEq, Eq, Debug)]
pub enum TextEvent {
    Created { id: TextId, text: String },
    Updated { id: TextId, text: String },
    Deleted { id: TextId },
}
```

Now, we have come to the point where we will implement the repository. By basing our persistence (and unit of work) around events, we can effectively batch execute all our persistence changes - allowing us to reduce the length of time our transaction is open. The only disadvantage of this approach is that I force us to know the id of the entities we are inserting. But for our purposes, that's actually preferred, as it allows us to do some cool performance tricks on the frontend.

Let's implement the repository:
```rust
pub struct TextRepository {
    pool: Arc<Pool<Sqlite>>,
    queue: Arc<Mutex<EventQueue>>,
}

impl TextRepository {
    pub fn new(pool: Arc<Pool<Sqlite>>, queue: Arc<Mutex<EventQueue>>) -> Self {
        Self { pool, queue }
    }
}

#[derive(FromRow)]
struct TextEntity {
    id: TextId,
    text: String,
}

#[async_trait]
impl Repository<Text> for TextRepository {
    async fn generate_id(&self) -> TextId {
        TextId(Uuid::new_v4())
    }
    async fn get_by_id(&self, id: TextId) -> Result<Option<Text>> {
        let query = sqlx::query_as("SELECT * FROM Texts where id = ? LIMIT 1").bind(id.clone());
        let text: Option<TextEntity> = query.fetch_optional(self.pool.as_ref()).await?;

        if let Some(text) = text {
            return Ok(Some(Text {
                id: text.id,
                text: text.text,
                events: Vec::new(),
            }));
        }

        return Ok(None);
    }

    async fn save(&self, entity: Text) -> Result<()> {
        // Todo: This could probably be moved out in a helper function
        let mut events = entity
            .get_events()
            .iter()
            .map(|event| DomainEvents::Text(event.clone()))
            .collect();

        self.queue.lock().unwrap().append(&mut events);

        // Notice, we are not actually doing anything with the database when we save.
        // Instead, we simply queue the changes, and let our unit of work persist them

        Ok(())
    }
    async fn delete(&self, entity: Text) -> Result<()> {
        let event = TextEvent::Deleted { id: entity.id };

        self.queue.lock().unwrap().push(DomainEvents::Text(event));

        Ok(())
    }
}


impl EventQuery for TextEvent {
    fn get_query(self) -> Query<'static, Sqlite, SqliteArguments<'static>> {
        match self {
            TextEvent::Created { id, text } => sqlx::query("INSERT INTO Texts (id, text) VALUES (?,?)")
                .bind(id)
                .bind(text),
            TextEvent::Updated { id, text } => sqlx::query("UPDATE Texts SET text = ? WHERE id = ?")
                .bind(text)
                .bind(id),
            TextEvent::Deleted { id } => sqlx::query("DELETE FROM Texts WHERE id = ?").bind(id),
        }
    }
}
```


Now that we have the fundamentals in place, let's create the UnitOfWork:

```rust
#[derive(Debug)]
pub enum DomainEvents {
    Text(TextEvent),
}

impl EventQuery for DomainEvents {
    fn get_query(self) -> Query<'static, Sqlite, SqliteArguments<'static>> {
        match self {
            DomainEvents::Text(text) => text.get_query(),
        }
    }
}

 pub struct EventQueue {
    events: Vec<DomainEvents>,
}

impl EventQueue {
    pub fn new() -> Self {
        let events = Vec::new();

        Self { events }
    }
}

impl EventQueue {
    pub fn append(&mut self, events: &mut Vec<DomainEvents>) {
        self.events.append(events);
    }

    pub fn push(&mut self, events: DomainEvents) {
        self.events.push(events);
    }

    fn take_all_events(&mut self) -> Vec<DomainEvents> {
        std::mem::take(&mut self.events)
    }
}

struct UnitOfWork {
    queue: Arc<Mutex<EventQueue>>,
    pool: Arc<Pool<Sqlite>>,
    text: TextRepository,
}

impl UnitOfWork {
    pub fn new(pool: Pool<Sqlite>) -> Self {
        let queue = EventQueue::new();
        let queue = Arc::new(Mutex::new(queue));
        let pool = Arc::new(pool);
        Self {
            text: TextRepository::new(Arc::clone(&pool), Arc::clone(&queue)),
            queue,
            pool,
        }
    }

    pub async fn commit(&self) -> Result<()> {
        let events = self.queue.lock().unwrap().take_all_events();

        let mut txn = self.pool.begin().await?;
        for event in events {
            let query = event.get_query();
            query.execute(txn.as_mut()).await?;
        }

        txn.commit().await?;

        Ok(())
    }
}
```

Now that we have both the infrastructure and the implementation in place, our integration test goes green.

Happy coding.
