---
title: Domain Driven Design in rust
date: 2024-09-18
---


I am deeply passionate about continuous learning, knowledge creation and finding good solutions to complex solutions. Over the past few weeks, I have been experimenting with different ways of implementing Domain Driven Design in rust. Here are the alternatives I have found, tested and implemented. 

It all started a few weeks ago, when a colleague introduced me to the concept of Event Sourcing. After talking with him, and reading up on the subject, I started looking at ways people have implemented it in rust. That's when I came across the [cqrs-es](https://docs.rs/cqrs-es/latest/cqrs_es/) create, which I later used for inspiration to implement my first attempt at implementing DDD aggregates: 

```rust
#[async_trait]
trait Aggregate {
    type Id;
    type Command;
    type Event: EventQuery + Clone;
    type Error;

    async fn handle(&self, command: Self::Command) -> Result<Vec<Self::Event>, Self::Error>;

    fn apply(&mut self, event: Self::Event);
}

struct TextId(u32);

enum TextCommand {
  Create { id: TextId, text: String },
  Update { id: TextId, text: String },
}

enum TextEvent {
  Created { id: TextId, text: String },
  Update { id: TextId, text: String },
}

struct Text {
  id: TextId,
  text: String
}

#[async_trait]
impl Aggregate for Text {
  type Id = TextId;
  type Command = TextCommand;
  type Event = TextEvent;
  type Error = String;

  
  async fn handle(&self, command: TextCommand) -> Result<Vec<TextEvent>, String> {
    match self {
      TextCommand::Create { id, text } => {
        // In pactice, we should not need to have validation here.
        // The error returned should ideally be business logic, and not request validation
        if text.len() == 0 {
          return Err("Text must have a length".to_owned())
        }

        return Ok(vec![
          TextEvent::Created { id, text }
        ]);
      },
      TextCommand::Update { text } => {
        return Ok(vec![
          TextEvent::Updated { id: self.id, text }
        ]);
      }
    }
  }

  fn apply(&mut self, event: TextEvent) {
    match self {
      TextEvent::Created { id, text } => {
        self.id = id;
        self.text = text;
      },
      TextEvent::Updated { id, text } => {
        self.text = text;
      }
    }
  }
}
```

Disadvantage: 
- The creation of the entity is handled differently than the update / deletion
- Due to the way the commands worked, I ended up creating helper methods on the entities themselves, meaning I had a create and update fn, which was kind of annoying...
- To handle and persist changes, we need a AggregateContext - overcomplicating transactions.

Advantage:
- Enum command:
  - Simplifies the pipeline, allowing us to autogenerate code using macros
  - Easy testing: we have a simple setup method that accepts all of our commands, removing a lot of the boilerplate in our tests
 


After recognizing that the commands simply call helper methods, I decided it was worth experimenting with. This is the code that I ended up with:

```rust
pub trait Aggregate {
    type Id;
    type Event;

    fn get_events(self) -> Vec<Self::Event>;
}

enum TextEvent {
  Created { id: TextId, text: String },
  Update { id: TextId, text: String },
}

struct TextId(u32);

struct Text {
  events: Vec<TextEvent>,

  id: TextId,
  text: String
}

impl Aggregate for Text {
  type Id = TextId;
  type Event = TextEvent;

  fn get_events(self) -> Vec<TextEvent>{
    self.events
  }
}

impl Text {
  pub(crate) fn new(id: TextId, text: &str) -> Self {
    Self {
      id,
      text: text.to_owned(),
      events: Vec::new()
    }
  }

  pub fn create(id: TextId, text: &str) -> Self {
    let mut entity = Text::new(id, text);
    entity.events.push(TextEvent::Created { id, text: text.to_owned() });

    entity
  }

  pub fn update(self, text: &str) -> Self {
    self.text = text.to_owned();
    self.events.push(TextEvent::Updated { id: self.id, text: text.to_owned() });

    self
  }
}

```

The beauty of this approach is in its simplicity. It allows us to really streamline our entities, and makes the code a whole more readable.


Disadvantages:
- None that I have found yet

Advantages:
- The create "command" is only accessible when we want to create the entity, and commands requiring an existing entity forces us to either get it from the repository, or to instantiate it.
- The events are stored on the entity itself. I have in a previous explained how I implemented a unit of work around this concept
- Less, and simpler code that is easier to maintain
