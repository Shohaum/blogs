# Why RESP Exists — The Protocol Behind Redis, Explained From First Principles

Before diving into knowing what RESP is, first we should know how RESP came into play.
I hope you guys have heard of Redis, worry not even if you have not. I will give you a brief on that.

RESP is a protocol, heavily used by Redis.

---

## But what is Redis?

Imagine an e-commerce website.

When you open the homepage, the site may need:

- your login status
- your cart items
- recently viewed products
- your search history

Instead of asking the main database every time, the app can ask Redis first. Redis gives the answer very quickly.

So **Redis helps applications store and fetch data much faster.**

---

## Why do we need Redis in the first place?

We need Redis because normal databases are not always the fastest choice for every task.

A traditional database (a system that stores data permanently on disk) is great for keeping records safely, but it can be slower when the same data is requested again and again.

Redis is useful when:

- data is needed very often
- speed matters more than long-term storage
- the same information is read repeatedly
- temporary data needs to be stored for a short time

Suppose a blog website has 1 million visitors.

If every visitor requests the same homepage data from the main database, the database gets overloaded.

Instead, the app can store that homepage data in Redis for a short time. Then all visitors get the data quickly without putting too much pressure on the main database.

In a login system, Redis can store:

- OTP attempts
- session data (temporary login information)
- rate limits (limits on how many times something can happen)

This makes the system faster and easier to control.

So, we need Redis when we want:

- speed
- less load on the main database
- temporary storage
- real-time responses

---

## How is Redis so fast?

Redis is fast mainly because **it keeps data in RAM.**

A normal database usually stores data on a disk, and disk access is slower than RAM access. Redis avoids that delay for most operations.

Redis is also fast because:

- it uses simple data structures (like strings, lists, sets, and hashes)
- it does not spend much time on complex processing
- it is designed for quick read and write operations
- many operations happen in memory instead of waiting for disk access

Think of two ways to find a book:

- In a library storage room, you may need time to search shelves, open boxes, and check records.
- On your study table, the book is already in front of you.

Redis works more like the study table. The data is already close and ready to use.

If an app needs to check a user's session 100 times, Redis can do it quickly because the session data is already in memory. The app does not need to wait for the disk every time.

That is why Redis feels extremely fast compared to many other storage systems.

---

## How did RESP come into play?

Let's think like building Redis from scratch.

### We built a Redis server… now how do clients talk to it?

We now have a Redis server — a program running somewhere that stores data in memory.

But users don't directly interact with memory. They use applications like:

- a backend server (Node.js, Python, etc.)
- a CLI (command line interface, a tool to type commands)

So the problem is:

> How does a client send commands like `SET name "shohaum"` to Redis?

### Can we just send plain text?

At first, it feels simple.

Client sends:

```
SET name shohaum
```

Redis reads it, splits by spaces, and executes.

Sounds fine… but breaks very quickly.

---

## Problems with plain text

### 1. What if data contains spaces?

Suppose client sends:

```
SET message Hello World
```

Now Redis server sees:

```
["SET", "message", "Hello", "World"]
```

- Is the value `"Hello"`?
- Or `"Hello World"`?

There is no way to know.

Even worse with spaces:

```
SET bio "I love system design"
```

If we split by spaces, Redis sees:

```
["SET", "bio", "I", "love", "system", "design"]
```

Wrong — a value should be one single string. Redis expects:

```
SET key value
```

### 2. What about binary data?

> Binary data = raw bytes, not just readable text — like images, files, etc.

Redis supports binary-safe strings like:

- Images
- Serialized objects
- Encoded data
- Even `\0`, `\r`, `\n`

```
SET file <binary_data>
```

Now:

- binary data may contain spaces
- may contain special characters
- may even contain invisible bytes

Plain text parsing completely breaks.

### 3. No clear boundaries between arguments

In plain text:

```
SET key value GET key
```

How does the server know:

- where one argument ends?
- where the next begins?

Here the arguments are `(SET, key, value)` and `(GET, key)` — but the server can't tell.

### 4. Hard to stream over TCP

TCP is a stream, not message-based. So we do not know how TCP will divide the data packets.

Suppose the client sends:

```
SET name Shohaum
```

Data may arrive like:

```
SE          → first packet
T name Shohau  → second packet
m           → third packet
```

If Redis server parses by splitting with spaces, the request arrives as:

```
["SE", "T", "name", "Shohau", "m"]
```

Now suppose Redis has a rule that the first token is the command, the second is the key, and the last is the value.

By that rule, `"SE"` is the command, `"T"` is the key, and `"Shohau"` is the value.

But Redis has no command called `SE`. It will throw an error:

```
(error) ERR unknown command `SE`, with args beginning with: `T`, `name`, `Shohau`, `m`
```

### 5. How does Redis know where data ends?

If data is large:

```
SET big_key <10MB data>
```

Data may arrive like:

```
Chunk 1 → SET big_key Hell
Chunk 2 → oooooooo....
Chunk 3 → ....(more data)
```

Server cannot know:

- "Is this complete?"
- "Should I wait for more?"

Plain text parser:

- Has to guess boundaries
- Must buffer everything *(a risky thing — if an attacker sends 10GB of data, it will eat up all the RAM)*
- You often need to read until delimiter
- Slow processing

There is no clear boundary.

### 6. Delimiter collision

Plain text depends on spaces, `\n`, and `\r`.

What if the command is:

```
SET file Hello\nWorld\nBinaryData...
```

- Is `\n` a delimiter?
- Or actual data?

Cannot say for sure.

### 7. Splitting becomes expensive

To parse:

```
SET key <huge_value>
```

- scan entire string
- split by spaces

For large data:

- O(n) scanning repeatedly
- CPU overhead increases

### 8. Risk of truncation / corruption

If connection drops mid-transfer:

```
SET key Hellooooooo...
```

Server doesn't know:

- incomplete data?
- or valid end?

---

## Need for a proper protocol

At this point, we realize:

> We don't just need text, we need a **protocol.**
> *(A protocol = a strict format for communication between systems)*

We already know:

- Plain text parsing breaks (spaces, binary, ambiguity)
- We need something predictable and machine-friendly

This protocol must:

- handle spaces safely
- support binary data
- clearly define boundaries
- address the TCP streaming issue
- be easy and fast to parse

---

## Let's solve this from first principles

So we have this command:

```
SET bio You know who I am
```

We have 8 major issues on parsing the command. Let's divide a command into 3 sections for simplicity:

- **command name**
- **key**
- **value**

So the above maps to: `SET`, `bio`, `You know who I am`

Both "key" and "value" are the arguments of the "command name."

---

## Approaches

### Approach 1: Use a separator

What about using a pipe `|`?

```
SET | bio | You know who I am
```

With this approach we can safely parse the command by splitting with `|` and we get:

```
["SET", "bio", "You know who I am"]
```

What if data itself contains `|`?

```
You know who | I am
```

**Any separator you choose can also appear inside data. Approach 1 is not a solution.**

---

### Approach 2: Using quotes

```
SET | bio | "You know who I am"
```

What if data contains quotes?

```
SET | bio | "You know "who" I am"
```

- Where does the string end?
- Do we escape quotes?
- How complex should parsing be?

**Approach 2 is not good.**

---

### Approach 3: Escape characters

```
SET | bio | "You know \"who\" I am"
```

Works well till now, but parsing in blazing fast O(1) time is the goal — and calculating escape sequences takes time.

- You need to handle escape rules
- More CPU work (processing overhead)
- Still tricky for binary data (`\x00\xFF\xA1…`)

**Approach 3 will not work either.**

---

## So where is the main problem? Let's look back and think again.

A user types a command into a CLI and hits "Enter." We (Redis devs) have to send the command over to a Redis server via TCP, and the server has to parse the entire command to know which one is the command name, key, and value.

We can use a separator so that the server can split the command by that separator. Seems good.

But here's the catch — TCP is not in our favor. Even with a simple command like:

```
SET | bio | "You know \"who\" I am"
```

Network packets can split anywhere. Redis might receive:

```
Chunk 1: SE
Chunk 2: T | bio | "You 
Chunk 3: know \"who\" I a
Chunk 4: m"
```

- TCP does **not preserve message boundaries**
- It only guarantees **order**, not completeness per read

Let's build a parsing strategy. On the server side:

- Start reading the incoming stream
- Wait until **two pipe (`|`) separators** are encountered
  - The first segment → command
  - The second segment → key
- Once the second pipe is found, switch to **quoted string parsing mode**
- Read everything inside double quotes, while handling escaped quotes

In a **simple case**, this logic can work. But in real systems, it breaks badly.

**Reality check:**

1. You still don't know when data is complete
2. Escaping becomes a nightmare
3. Binary data breaks everything
4. Hardcoded assumption of exactly 2 pipes doesn't work — `PING` has 0 args, `SET key value EX 10 NX` has many
5. Pipelining is impossible without clear boundaries

---

## Fixes

### Fix 1: Treat the command as a list

Stop thinking in "text" and think in **structured data:**

```
*3
SET
bio
You know "who" I am
```

`*3` means: total 3 elements.

We no longer depend on spaces to split data.

---

### Fix 2: Add length before every element

```
*3
$3
SET
$3
bio
$23
You know "who" I am
```

In `$<length>`, `$` represents a **bulk string** and `<length>` is the length of the string.

Now Redis reads:

- `$3` → read exactly 3 bytes: `SET`
- `$3` → read exactly 3 bytes: `bio`
- `$23` → read exactly 23 bytes: full value

---

### Fix 3: Ignores content completely

Earlier we had:

```
"You know \"who\" I am"
```

We needed escape rules, quote handling, and complex parsing.

With length-based reading:

```
$23
You know "who" I am
```

Redis doesn't care — quotes are just characters, no escaping logic needed. **It does not interpret data, it just counts bytes.**

---

### Fix 4: Length-based reading handles TCP fragmentation

Data may arrive like:

```
Chunk 1: *3 $3 SE
Chunk 2: T $3 bi
Chunk 3: o $23 You kn
Chunk 4: ow "who" I am
```

With length-based reading:

- `$3` → wait until 3 bytes are complete → `SET`
- `$3` → wait → `bio`
- `$23` → wait → full value

Even if split, Redis reconstructs perfectly. It turns a messy TCP stream into predictable chunks.

---

### Fix 5: Array count defines boundaries

Suppose client sends two commands back to back. TCP may combine them:

```
SET bio "hello"GET bio
```

But with count boundaries:

```
*3
$3
SET
$3
bio
$5
hello
*2
$3
GET
$3
bio
```

- `*3` → first command ends after 3 elements
- `*2` → next command starts

No confusion.

---

### Fix 6: `*<count>` handles variable argument counts

Commands vary:

```
PING           → *1
GET key        → *2
SET key value EX 10 NX  → *5
```

Now you know exactly how many arguments are in the command. Fully flexible.

---

### Fix 7: Length-based = binary safe

Value could be:

```
\x00\xFFhello\nworld
```

Separator / quotes fail — special chars break parsing.

With length-based reading:

```
$11
<raw bytes>
```

Redis reads exactly 11 bytes. Doesn't care what's inside.

---

We've reached something like:

```
*3
$3
SET
$3
bio
$23
You know "who" I am
```

Looks structured. Seems fine.

---

## But we still have some problems — let's solve them

### Problem: Where does a number end?

Look at this:

```
*3$3SET$3bio$23You know "who" I am
```

Now ask:

- Where does `3` end?
- Where does `$3` end?
- Where does metadata stop and data begin?

Character-by-character scanning, conditional logic, more CPU work.

**We need a clear delimiter.**

> Is there any delimiter that is historically reliable across platforms?

Yes — `\r\n`. It is used by many network protocols (HTTP, SMTP, etc.) and avoids ambiguity across systems.

---

### Understanding `\r\n`

`\r\n` is called **CRLF (Carriage Return + Line Feed).**

**What does `\r` mean?**

`\r` is a special character called **Carriage Return (CR)**. It moves the cursor to the start of the *same* line.

**What does `\n` mean?**

`\n` is called **Line Feed (LF)**. It moves the cursor down to the next line.

**Example — `\r`:**

```
Hello\rWorld
```

1. Write `"Hello"` → `Hello|`
2. `\r` → cursor jumps to beginning → `|Hello`
3. Write `"World"` → overwrites from start → `World`

> Note: `\r` does NOT delete text. It just moves the cursor. Overwriting happens because new text is written on top.

**Example — `\n`:**

```
Hello\nWorld
```

1. Write `"Hello"` → `Hello|`
2. `\n` → go to next line
3. Write `"World"`

Result:

```
Hello
World
```

**Combined — `\r\n`:**

```
Hello\r\nWorld
```

1. `\r` → go to start of line
2. `\n` → go to next line

Result:

```
Hello
World
```

---

### Fix: Add `\r\n` as a clear delimiter for metadata

```
*3\r\n
$3\r\n
SET\r\n
```

Now Redis can:

- read until `\r\n`
- parse the number cleanly

No guessing. No scanning logic.

---

### Fix: Separate metadata from data

Consider:

```
$23You know "who" I am
```

Without delimiter, the parser must scan digits and detect the boundary. Is it `$2` + `3You...`? Or `$23` + `You...`?

With `\r\n`:

```
$23\r\n
You know "who" I am
```

Now:

- read until `\r\n` → `$23`
- parse length = 23
- read next 23 bytes

**Clean and deterministic.**

---

### Fix: `\r\n` gives a strong boundary even across TCP chunks

Data may arrive like:

```
$23\r
\nYou know "who" I am
```

Without a clear delimiter, parsing becomes fragile. With `\r\n`, the server logic becomes:

> Keep reading until I hit `\r\n`, now I have a complete header.

Even if split across chunks — it waits, then processes safely.

**The final parser becomes:**

1. Read line until `\r\n`
2. Interpret line (`*3`, `$23`, etc.)
3. Act accordingly

Extremely simple.

---

## One question remains — who converts human commands into RESP?

Whatever is converting the commands into RESP, won't it face all the same problems?

That something is **the client** (not the user).

**The client does NOT figure this out from a continuous stream.** It already gets commands as **separate inputs.**

### Two real scenarios:

**1. Interactive client (like `redis-cli`)**

You type:

```
SET name shohaum
```

You press **Enter** — that Enter = end of one command.

Then:

```
GET name
```

Another Enter = new command.

So internally:

```
Command 1 → "SET name shohaum"
Command 2 → "GET name"
```

Client processes **one line at a time**, converts each to RESP, sends separately.

**2. Programmatic client (Node.js, Python, etc.)**

You don't send raw text like:

```
SET name shohaum GET name
```

Instead you call functions:

```python
client.set("name", "shohaum")
client.get("name")
```

Each function call = one command. Client directly builds RESP.

> If you're imagining: `SET name shohaum GET name INCR counter` — the client never allows this format at all. CLI splits by newline (Enter key). Libraries use function calls.

---

## Conclusion

This approach is known as **Redis Serialization Protocol (RESP)**, which Redis uses to transform simple human-readable commands into a structured, unambiguous, and TCP-friendly format.

Instead of relying on separators, quotes, or assumptions, RESP defines everything explicitly:

- how many arguments exist
- how long each piece of data is
- where each part ends

This removes all ambiguity from parsing, making communication between client and server reliable — even in the face of TCP streaming, partial reads, and binary data.
