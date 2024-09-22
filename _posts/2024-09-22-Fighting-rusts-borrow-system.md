---
date: 2024-09-22
title: Fighting rusts type system
---

As I refactor my codebase, the borrow checker started throwing errors like: "cannot borrow self as mutable because it is also borrowed as immutable" or "cannot borrow self.events as mutable more than once at a time."

After several hours of trial and error, I found a solution that seems to work quite well.

In my code, I frequently use helper methods to retrieve linked list items stored in a Vec. These methods originally had a signature like `get_children(&mut self, parent_id: Id) -> Vec<&mut Item>`, which caused a lot of headaches. Eventually, I realized that by returning Ids instead of mutable references, I could then retrieve the mutable items I needed by querying with their IDs: `get_child_ids(&mut self, parent_id: Id) -> Vec<Id>`

After adopting this approach, I realized that in many method calls, I only needed the item’s ID, which made the code cleaner and easier to work with. In cases where I need the full item, I now make two function calls to retrieve it. To me, this is an acceptable trade-off. While I’m not entirely sure if it's the best long-term solution, it effectively solves my immediate problem.

Happy Coding
