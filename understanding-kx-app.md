To determine how `AverageDailyVolume` is calculated in a KDB+/KX application, you’ll want to **trace the lineage of that field** through the codebase. Here’s a step-by-step method to help you reverse-engineer or discover how it's derived:

---

### **1. Search the Codebase for `AverageDailyVolume`**

Start with a full-text search across your `.q` scripts or any configuration/DSL files in your KX application.

```bash
grep -r "AverageDailyVolume" .
```

If you’re using an IDE like VS Code or an editor like Sublime/IntelliJ, use **"Find in Files"** for `AverageDailyVolume`.

Look for patterns like:

* `AverageDailyVolume:`
* `AverageDailyVolume =`
* `update AverageDailyVolume`
* `select ..., AverageDailyVolume: ...`
* `insert` or `upsert` with a column named `AverageDailyVolume`

---

### **2. Identify the Context**

Once you find it, examine the context:

* Is it in a **query**, a **function**, or a **pipeline transformation**?
* Is it calculated in a `select`/`update`?
* Is it a result of an **aggregation** (like `sum volume` or `avg` over a grouped table)?

Example:

```q
update AverageDailyVolume: volume sum each groupByDate % count each groupByDate from trade
```

This might be computing the sum of volume per day divided by count of entries per day (i.e., average per day).

---

### **3. Look for Function Definitions**

If it’s calculated inside a function, you’ll see something like:

```q
calcAdv:{[t] ... update AverageDailyVolume: ... from t }
```

Then, search for where `calcAdv` is called and how the input table is built.

---

### **4. Check Intermediate Tables or Streams**

In many KX applications (especially real-time ones), the final field is built from a **chain of transformations** on:

* real-time streams (e.g., `trade`, `quote`)
* historical data
* daily aggregates

Find where those tables are:

```q
tables[]
```

And inspect their schemas:

```q
meta trade
```

Then check where those tables feed into the logic that calculates `AverageDailyVolume`.

---

### **5. Analyze Time-based Aggregation**

Since the field is called **AverageDailyVolume**, it's very likely to be:

* `sum volume by sym, date` divided by the number of trading days, or
* `avg volume` grouped by sym/date.

Check for code like:

```q
select AverageDailyVolume: avg volume by sym, date from trade
```

Or if the app uses sliding windows, maybe something like:

```q
select AverageDailyVolume: sum volume % count i by sym from trade where date within (start;end)
```

---

### **6. Ask the Running Process**

If the system is running and `AverageDailyVolume` is in memory (say, in a table), do:

```q
select from <tablename> where not null AverageDailyVolume
```

Then use `\d` to inspect namespaces and functions that may relate:

```q
\d .
```

Look for any function definitions like `.analytics.calcADV` or `.daily.adv`.

---

### **7. Bonus – If All Else Fails:**

If it’s calculated in a UI (like KX Dashboards or a gateway query), check:

* the **schema definitions**
* **dashboard queries**
* **APIs** defined in the gateway

---

If you want, you can paste a code snippet or some output from your search here and I can help interpret it.
