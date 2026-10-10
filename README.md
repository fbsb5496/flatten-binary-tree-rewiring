# flatten binary tree to linked list: reverse preorder, the O(1)-space rewiring trick, and the edge cases interviewers probe

Search this phrase and you'll hit two different problems wearing the same name. One is LeetCode 114 itself: a Medium that asks you to rewire a binary tree into a right-skewed chain in preorder. The other is the follow-up printed right under the constraints — *can you flatten the tree in place with O(1) extra space?* A good chunk of the traffic around this problem comes from people who solved it the easy way, submitted, and then noticed their solution quietly allocates an array, a queue, or a stack of size n.

This walks through three implementations, why each one works, what to say when someone asks for complexity, and the specific bugs that turn a correct-looking solution into a wrong answer on the last hidden test case. The problem is rated Medium with roughly a 71% acceptance rate, which tells you the basic idea isn't the hard part — the details are.

## What "flatten" actually means here

Given the root of a binary tree, you rewire it so that:

- the right pointer of each node points to the next node in preorder,
- the left pointer of every node is null,
- the node objects stay the same. Only their pointers move.

The result is still built out of `TreeNode`. It just behaves like a linked list because nothing has a left child anymore. That's why the judge prints the answer with interleaved nulls: `[1,null,2,null,3,null,4,null,5,null,6]`.

From the problem statement: node count is between 0 and 2000, values sit between -100 and 100, and the function returns nothing. You mutate `root` in place and the judge walks the right pointers afterward to check your work.

## The one insight that makes the problem click

Textbook preorder recursion on a binary tree looks like this:

python
def visit(node):
    do_something(node)
    visit(node.left)
    visit(node.right)


Applied naively here, that order is fatal. The instant you assign `node.right` to something else, you've lost your only handle on the original right subtree, and you haven't visited it yet. Preorder is Node → Left → Right, and flattening writes to the exact pointer you still need to read.

So you run it backwards. Recurse Right → Left → Node, which is preorder read in reverse. By the time you touch a node, everything below it has already been flattened, and the only work left is pointing this node at whatever you processed last.

That's the whole trick. `prev` holds the head of the chain built so far; each node you visit in reverse becomes the new head.

## Approach 1: recursion in reverse preorder

python
class Solution:
    def flatten(self, root):
        self.prev = None

        def dfs(node):
            if not node:
                return
            dfs(node.right)
            dfs(node.left)
            node.right = self.prev
            node.left = None
            self.prev = node

        dfs(root)


Trace it on `[1,2,5,3,4,null,6]`. Nodes get processed in the order 6, 5, 4, 3, 2, 1. Node 6 points its right at `None` because `prev` started empty. Node 5 points right at 6. Node 4 points right at 5, so the 4-5-6 chain is what node 3 receives a moment later. By the time the root runs, node 2 already sits at the head of the finished chain, so `root.right = 2` and `root.left = None` closes it out.

Time is O(n), stack depth is O(h) where h is the height. Balanced tree, that's O(log n). A 2000-node path, that's 2000 frames.

Which leads to a Python-specific trap worth knowing before it bites you: CPython's default recursion limit is 1000, and the constraint allows up to 2000 nodes. A maximally skewed input can blow the limit even though your algorithm is correct. The pointer-rewiring version below avoids the issue entirely, which is one reason it's worth learning rather than treating as a curiosity.

## Approach 2: explicit stack

If you'd rather drive the traversal yourself:

python
class Solution:
    def flatten(self, root):
        if not root:
            return
        stack = [root]
        while stack:
            node = stack.pop()
            if node.right:
                stack.append(node.right)
            if node.left:
                stack.append(node.left)
            node.left = None
            if stack:
                node.right = stack[-1]


Two details carry the whole thing.

Push right before left. A stack pops last-in-first-out, so right goes in first and left comes off first, which preserves preorder.

Then, after nulling the left pointer, `stack[-1]` is exactly the next node in preorder. That's what `node.right` gets pointed at. The final node in the traversal finds an empty stack and keeps its `right` as `None`.

Time is O(n), space is O(h) for the stack. On a skewed tree that's O(n) again, so this version buys you the recursion-limit escape, not a better space class.

## Approach 3: in-place rewiring with O(1) extra space

This is the follow-up answer, and it's usually what an interviewer wants once the recursive version is already on the board.

The idea: walk down the right spine. Whenever the current node has a left child, find the rightmost node of that left subtree, hang the node's original right subtree off it, then move the left subtree over to the right and clear the left pointer. Repeat.

python
class Solution:
    def flatten(self, root):
        cur = root
        while cur:
            if cur.left:
                last = cur.left
                while last.right:
                    last = last.right
                last.right = cur.right
                cur.right = cur.left
                cur.left = None
            cur = cur.right


Run it on `[1,2,5,3,4,null,6]`. At node 1, the left subtree is 2 with children 3 and 4. The rightmost node there is 4, so `4.right = 5` grafts the original right subtree onto the end. Node 1's right becomes 2, its left becomes null. `cur` then moves to 2, where the same move happens: rightmost of the left subtree is 3, so `3.right = 4` and 2's right becomes 3. From 3 onward there are no left children left to process, so the loop just follows right pointers down to 6.

No array, no stack, no recursion. Every pointer write reuses memory the tree already occupies.

Time stays O(n). The inner loop that hunts for the rightmost node looks like it should push you to O(n log n) or worse, but each edge in the tree is traversed a bounded number of times across the entire run, so the total work is linear. Space is O(1) beyond the two pointer variables.

## Complexity cheat sheet

| Approach | Time | Extra space | When to reach for it |
| --- | --- | --- | --- |
| Collect preorder into an array, relink | O(n) | O(n) | Fastest to write, fails the follow-up |
| Recursive reverse preorder | O(n) | O(h) frames | Clean interview answer if you explain the reversal |
| Explicit stack | O(n) | O(h) | Same space class, immune to recursion limits |
| Pointer rewiring | O(n) | O(1) | The follow-up. Also the most bug-prone |

## Mistakes that show up again and again

**Leaving left pointers stale.** The judge follows right pointers, but the statement is explicit that left must be null. A chain of correct right pointers with leftover left children is not a flattened tree.

**Forward preorder with recursion and a stashed right child.** It can work, but it's easy to get wrong. The usual failure is code that reassigns `node.right` and then recurses into `node.left`, silently dropping an entire subtree. If your output is *missing* nodes rather than misordered, this is nearly always the cause.

**Assuming "flattened" means sorted.** You're building the list in preorder, not ascending value order. This problem sits under tree and DFS tags, not BST tags.

**Returning a value.** The signature is `void`. Mutating the structure *is* the answer. Return `root` in your own driver code if it's convenient, but on the judge nothing is expected back.

**Reordering the stack push and the pointer assignment.** Popping a node, pushing both children, and then assigning `node.right` is what keeps the chain intact. Swap those lines and you'll point at a node you've already consumed.

## Edge cases worth running before you call it done

- `root = None`. Every version needs this as its first check.
- A single node. No rewiring should happen at all.
- An entirely left-skewed tree. The output is still a single chain hanging off right pointers, and the rightmost-node search is doing real work on every node.
- An entirely right-skewed tree. Nothing to move, and nothing should be touched.
- A full tree where every node has two children. This is where the "find the rightmost node" logic either holds or falls apart.

## Turning this into a pattern, not a memory

What makes this problem worth more than its own solution is that it's the same mechanic as Morris traversal, threaded trees, and a family of linked-list pointer puzzles: reuse existing links instead of allocating a helper structure.

A sequence that actually builds the skill:

1. Write the reverse preorder recursion from memory, no notes.
2. Convert it to the explicit stack version and explain out loud why right goes in first.
3. Rewrite it as the O(1)-space loop and state the amortized-time argument in one sentence.
4. Take the same three passes at a related problem, like a Morris inorder traversal or reversing a subtree in place.

Step three is the one that moves you from "I've seen this before" to "I can derive it under pressure." AlgoMonster's write-up of this exact problem leans on the same reverse-traversal reasoning, and their version changes the shape slightly: the signature is `flatten_tree(tree)`, it returns the flattened root rather than mutating a void function, and the tree is serialized with `x` marking an empty node. There's also a forum thread there where someone argues the stack ordering in their illustration looks wrong, and the answer is the FILO explanation above.

👉 [Practice flatten binary tree to linked list on AlgoMonster](https://algo.monster/problems/flatten_binary_tree?fpr=li69)

## Where AlgoMonster fits, and what it costs

If tree DFS is one of several patterns you're working through and you want a defined next step instead of picking problems at random, this is the pitch AlgoMonster sells: content organized around reusable patterns rather than a flat problem bank. Tree traversal is one of the categories, and the flatten problem is filed under DFS and binary tree work alongside preorder, inorder, and postorder variants.

The official subscribe page currently lists three tiers.

| Plan | Price | Billing | What you get |
| --- | --- | --- | --- |
| Free | $0 | — | Limited access to a subset of patterns and problems, enough to test the text-and-diagram format |
| Pro Yearly | $99/year (shown as $8.25/mo) | Billed annually after a 3-day free trial | All patterns, lessons and problems; unlimited AI assistant; unlimited Speedrun; real questions from top tech companies updated monthly; 325 lessons and problems, 678 illustrations, 3000+ editorials; member discount on live bootcamps; 12 months of access |
| Pro Lifetime | $189 | One-time | Everything in Pro Yearly, plus lifetime updates and an internal referral to a top tech company for qualified candidates |

Three things to know before you click buy.

The prices above are promotional and move around. Third-party review sites list different numbers — several quote a $459 lifetime tier, and one puts annual somewhere between $119 and $299 — because they snapshot older sales and don't update. The official subscribe page is the only figure that matters at checkout.

There's no monthly plan. It's annual or lifetime, nothing in between, and the annual option runs through a 3-day free trial before the yearly charge lands.

There's no team or enterprise tier. This is built for individual candidates.

Which one makes sense: if your interview loop is inside the next couple of months, annual is the lower-risk spend. Lifetime earns its premium if you expect to interview again in a few years, which is common enough among engineers who change jobs every two or three cycles. At $99/year, the $189 one-time cost breaks even after the second renewal.

👉 [Compare AlgoMonster's current plans and start the free trial](https://algo.monster/subscribe?fpr=li69)

## FAQ

**Is O(1) space required to pass?** No. The recursive and explicit-stack versions are both accepted. The O(1) constraint appears only in the follow-up, and it's what separates a solved problem from a strong interview answer.

**Why does my output have nulls between every value?** Because the "linked list" is still made of `TreeNode` objects. The serialized form includes a left slot for each node, and every one of those is null. `[1,null,2,null,3,null,4,null,5,null,6]` is correct output, not a bug.

**Does this count as a binary search tree problem?** No. The tree is unsorted and ordering never enters the picture. It's filed under DFS, binary tree, stack, and linked list.

**How long should the O(1) version take to write?** With the pattern internalized, under five minutes, and most of that is the inner rightmost-node loop. If you're reconstructing it from scratch each time, that's the signal to drill it a few more times rather than move on.

**Can the recursive version fail even when it's logically correct?** In Python, yes. A 2000-node skewed tree exceeds the default recursion limit. The iterative versions don't care.

If you only take one thing from this: the recursion is easy once you accept that you have to traverse backwards, and the O(1) version is just that same backwards logic expressed as pointer surgery. Get both into your fingers, and this problem stops being a LeetCode 114 question and becomes a general answer to "how do I restructure a tree without allocating anything new."
