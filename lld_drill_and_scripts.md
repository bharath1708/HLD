# LLD Schema Drill — Checklist + Reference

The gap that capped every mock at ~0.55 LLD is now closed. This is the checklist to
run in your head on EVERY table in the interview, plus worked examples.

---

## THE CHECKLIST (run on every table, out loud)

For each table, ask "how is this queried?" and answer:

1. **Every table gets addressed** — even boring ones. Either an index, or explicitly
   "PK-only, no secondary index needed." Never silently skip a table.
2. **Derive the index from the killer query** — name the query, then the index.
   ("the query is X → so index on Y")
3. **The index's columns must EXIST** — and the query's filter columns must exist as
   table columns (don't reference a column you forgot to add — e.g. the missing room_id).
4. **Composite order: equality columns FIRST, then range/sort columns.**
   (room_id before dates; user_id before created_at)
5. **Sort direction matches the query's MEANING:**
   - "recent first" → created_at DESC
   - "upcoming first" → start_date ASC
   - "in order" → a sequence column (lesson_number), not created_at
6. **EXACT match — no missing columns (under-index), no extra columns (over-index).**
   The index covers exactly what the query filters/sorts on. Padding an index with
   extra columns "just in case" is wasteful (bigger index, slower writes).
7. **Many-to-many / join tables queried BOTH directions need BOTH indexes.**
   (follows: follower_id AND followee_id; enrollment: student_id AND course_id)

---

## Worked examples (from the drill)

### Product catalog + reviews
```
categories   → PK-only (accessed by id)
products     → index (category_id[, sort col])   "products in a category"
users        → unique index (phone)               "auth by phone"
reviews      → index (product_id, created_at DESC) "reviews for a product, recent first"
```

### Support tickets (two-filter query)
```
customers    → PK-only
agents       → unique (email) if login by email, else PK-only
tickets      → index (assigned_to, status)         "my tickets by status"
             → index (created_by, created_at DESC)  "a customer's tickets, recent"
             → index (status, priority)             "open high-priority (triage)" — BOTH filters
```

### Learning platform (many-to-many)
```
students     → unique (email)
courses      → PK-only
lessons      → index (course_id, lesson_number)    "lessons in a course, IN ORDER" (sequence, not created_at)
enrollment   → index (student_id)                  "courses a student is in"      ← direction 1
             → index (course_id)                   "students in a course"          ← direction 2
             → unique (student_id, course_id)       prevent duplicate enrollment
```

### Food delivery (don't over-index)
```
restaurants  → PK-only
menu         → index (restaurant_id)               "a restaurant's menu"
customers    → unique (email)
drivers      → index (phone)
orders       → index (customer_id, created_at DESC) "customer's orders, recent"
             → index (assigned_to, status)          "driver's active orders" — BOTH filters
order_items  → index (order_id)                     "items in an order" — order_id ONLY, don't pad
```

### Hotel booking (date-range availability)
```
hotels       → PK-only
rooms        → index (hotel_id)                    "rooms in a hotel"
guests       → unique (email)
bookings     → index (guest_id, start_date ASC)    "a guest's bookings, upcoming first" (ASC!)
             → index (room_id, start_date, end_date) "availability: overlap for a room"
               (booking MUST have room_id — a booking is for a room; equality room_id LEADS)
```
Overlap query: `WHERE room_id=X AND start_date < :req_end AND end_date > :req_start`
Overlap rule: two ranges overlap ⇔ A.start < B.end AND A.end > B.start

---

## The trajectory (proof the reflex closed)
- Rep 1: skipped 2 tables (old gap)
- Rep 2: all tables addressed, 3 indexes derived
- Rep 3: PK-only stated, clean; down to sort + bidirectional join
- Rep 4: both-filter query nailed; one OVER-index (opposite of the old problem)
- Rep 5: sorts derived, availability index; down to leading-column order + missing FK

Gap closed. What remains is the precision checklist above — run it per table.

===============================================================================

# Scripted Answers (the two "soft" sections)

## Intro (~90s — career overview, technical OWNERSHIP)

"Hi, I'm Bharath Kumar Kandasamy — 9+ years in software engineering, with real depth
in backend, distributed systems, and payments.

I started at Prodapt on AT&T's platforms, beginning as a backend developer and growing
into a full-stack role. I then moved to Verizon, where I worked on payments and
revenue-assurance systems — that's where I built my foundation in transaction processing
and financial-grade reliability.

Now I'm a Product Development Engineer at Comcast on the Center of Excellence team,
building internal platforms that engineering teams rely on across their SDLC. My flagship
project is the Performance Portal — an in-house replacement for LoadRunner, the commercial
licensed tool, built on Angular, Spring Boot, and Kubernetes with entirely open-source
components. **I architected it end to end — the UI, the backend services, and the
Kubernetes deployment — and it saves the company around $10 million a year in licensing.**

My core strength is backend and distributed systems — Java, Spring Boot, Kubernetes —
with production depth in payments. I'm looking to take on more senior, end-to-end ownership
of large-scale systems, which is what drew me to this role."

Delivery: ~90s, out loud until it flows. Beat after "end to end" and after "$10 million."
Slow down on the payments sentence (your BFSI differentiator). End on the intent line.

## AI-assisted development (~60-90s)

"I use AI as a productivity multiplier for the repetitive parts of development, so I can
focus my time on design and problem-solving.

Day to day, that means boilerplate — try-catch blocks, logging, repetitive CRUD — plus
API documentation, generating test cases, and as a first-pass code reviewer to catch
obvious issues before human review.

But I'm deliberate about where I trust it. I always review generated code before it goes
in — especially anything touching business logic, security, or payments, where a subtle
mistake is expensive. AI is great at the mechanical 80%; the architecture, the edge cases,
and the judgment calls stay with me. I treat it like a fast junior pair-programmer —
useful, but everything it produces gets verified."

Keep it short. The balanced list + the "where I DON'T trust it" line (payments/security)
is the whole answer — it shows productivity AND judgment.
