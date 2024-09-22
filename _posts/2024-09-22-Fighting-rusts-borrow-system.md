---
date: 2024-09-22
title: Fighting rusts type system
---

As I refactor my codebase, the borrow checker started throwing errors like: "cannot borrow self as mutable because it is also borrowed as immutable" or "cannot borrow self.events as mutable more than once at a time."

After several hours of trial and error, I found a solution that seems to work quite well.

In my code, I frequently use helper methods to retrieve linked list items stored in a Vec. These methods originally had a signature like `get_children(&mut self, parent_id: Id) -> Vec<&mut Item>`, which caused a lot of headaches. Eventually, I realized that by returning Ids instead of mutable references, I could then retrieve the mutable items I needed by querying with their IDs: `get_child_ids(&mut self, parent_id: Id) -> Vec<Id>`

After adopting this approach, I was able to work around the issue. The new implementation requires two function calls to retrieve my items, but for now, it works. I'm not entirely sure if this is the best solution long-term, but it has solved my immediate problem.
