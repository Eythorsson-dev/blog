---
title: Data consistency in a Note-taking application
date: 2024-09-15
---

I recently started a rewrite of my Note-taking application, as I wanted to experiment, and implement the DDD principles. I have currently figured out that I should create an aggregate that represents a Note, and others that represent the different block types: 
- Paragraph (aka. text)
- List (aka. text with a list type; Ordered, unordered)
- Table
- Image 
- Video
- Canvas

The note stores the different types of blocks as a linked list - where each block can be any number of different types; Image, Text, Table, Video, etc. This gives us a good separation of concern, as the logic for handling moving, deleting and inserting blocks are quite complex. Especially, since indentation, a requirement part of any good note-taking application (in my opinion).

What I am struggling with is figuring out how I should handle creating new blocks, as it requires changes in both the Note, and the corresponding block aggregate (ex. Text).

In my current implementation, aggregates, may only reference each other using Ids. Meaning that a block, may only reference a Text by the TextId. Resulting in me having to create the text before I create the block. 

Example: 
```rust

struct Context{
  uow: UnitOfWork
}

async fn create_block_command(ctx: Context, note_id: NoteId, id: BlockId, parent_id: Option<BlockId>, previous_id: Option<BlockId>, r#type: BlockTypeDto) -> Result<(), Error> {
    let r#type: BlockType = match r#type {
      BlockTypeDto::Text { id, text } => {
        let text = Text::new(id, text);
        ctx.uow.text.save(text).await?;
        BlockType::Text(text_id)
      }
    };k
  
    let note = ctx.uow.text.get_by_id(note_id).await?;
  
    note.create_block(id, parent_id, previous_id, r#type);
    
    ctx.uow.note.save(note).await?;
    
    ctx.uow.commit();

    Ok(())
}
```

The above POC, requires me to refactor my current command handling, and unit of work implementation - but it seems way better than what I currently have (a cqrs implementation heavily inspired by the [cqrs-es](https://docs.rs/cqrs-es/latest/cqrs_es/) crate).
