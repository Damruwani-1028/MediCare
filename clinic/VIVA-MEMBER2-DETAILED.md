# Member 2 — Appointments, Schedules & Queue
## Detailed viva notes

Your part is the middle of the system. The patient portal and the doctor portal
both talk to your tables, and reception runs the day out of your queue page.
If your module is wrong, everything downstream is wrong — which is also why the
examiner is likely to spend the most time here.

---

## 1. Your files and your tables

| File | What it holds |
|---|---|
| `models/Appointment.php` | every query that touches the `appointment` table |
| `models/Schedule.php` | working sessions, and turning them into bookable slots |
| `controllers/StaffController.php` | the `queue`, `appointments`, `book` and `status` methods |
| `controllers/HomeController.php` | the `slots` method - free slots as JSON |
| `views/staff/queue.php`, `views/staff/appointments.php`, `views/staff/book.php`, `views/patient/book.php` | the HTML |

Two tables: **`appointment`** and **`doctor_schedule`**.

You do not own `patient` or `doctor`, but you hold foreign keys to both.

---

## 2. The two tables, column by column

### `doctor_schedule` — when a doctor works

```sql
CREATE TABLE doctor_schedule (
    Schedule_ID    INT AUTO_INCREMENT PRIMARY KEY,
    Doctor_ID      INT NOT NULL,
    Available_Date DATE NOT NULL,
    Start_Time     TIME NOT NULL,
    End_Time       TIME NOT NULL,
    Is_Available   TINYINT(1) DEFAULT 1,
    CONSTRAINT fk_schedule_doctor FOREIGN KEY (Doctor_ID) REFERENCES doctor(Doctor_ID),
    CONSTRAINT uq_schedule UNIQUE (Doctor_ID, Available_Date, Start_Time)
);
```

One row is a **session**, not a slot: "Dr Perera works 09:00–12:00 on 15 March."
`Is_Available` lets a doctor close a session without deleting it, which matters
because appointments already point at that `Schedule_ID`.

The UNIQUE stops a doctor entering the same session twice by double-clicking.

### `appointment` — one booking

```sql
CREATE TABLE appointment (
    Appointment_ID   INT AUTO_INCREMENT PRIMARY KEY,
    Patient_ID       INT NOT NULL,
    Doctor_ID        INT NOT NULL,
    Schedule_ID      INT,
    Receptionist_ID  INT,
    Appointment_Date DATE NOT NULL,
    Time_Slot        TIME NOT NULL,
    Queue_No         INT,
    Status           ENUM('Pending','Confirmed','Completed','Cancelled','No-Show')
                     NOT NULL DEFAULT 'Pending',
    Reason           VARCHAR(255),
    Created_At       TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    Slot_Key TIME GENERATED ALWAYS AS (IF(Status = 'Cancelled', NULL, Time_Slot)) STORED,

    ... four foreign keys ...
    CONSTRAINT uq_doctor_slot UNIQUE (Doctor_ID, Appointment_Date, Slot_Key)
);
```

Points worth making about this table:

- **`Status` is an ENUM, not a VARCHAR.** The database itself refuses any value
  outside the five listed. A typo like 'Completed' cannot get in.
- **`Queue_No` is nullable** because a cancelled appointment has no place in the
  queue — cancelling sets it back to NULL.
- **`Schedule_ID` is nullable** so a reception walk-in booking can exist even
  when it did not come from a published session.
- **`Created_At`** records when the booking was made, which is different from
  when the visit happens.

---

## 3. Turning sessions into slots

This is the idea an examiner most often asks you to explain, because it is the
one design decision that is not obvious.

**Why not store every slot as a row?** A doctor working 9–12 and 4–6 every day
for a month would be 200+ rows to insert, and changing one session would mean
deleting and re-adding all of them. Instead we store the session once and
calculate the slots when someone asks.

`Schedule::freeSlots()` does it:

```php
foreach ($sessions as $session) {
    $start = strtotime($date . ' ' . $session['Start_Time']);
    $end   = strtotime($date . ' ' . $session['End_Time']);

    for ($t = $start; $t < $end; $t += self::SLOT_MINUTES * 60) {
        $time = date('H:i:s', $t);

        if (in_array($time, $bookedSlots, true)) { continue; }   // already taken
        if ($isToday && $t < time())             { continue; }   // already passed

        $slots[] = [
            'schedule_id' => $session['Schedule_ID'],
            'time'        => $time,
            'label'       => date('h:i A', $t)
        ];
    }
}
```

Read it out loud as: *start at the session's start time, step forward 30 minutes
at a time until the end time, and skip any step that is already booked or has
already passed today.*

`SLOT_MINUTES` is a class constant, so changing 30 to 20 changes the whole system
in one place.

`$bookedSlots` comes from `Appointment::bookedSlots()`:

```sql
SELECT Time_Slot FROM appointment
WHERE Doctor_ID = ? AND Appointment_Date = ? AND Status <> 'Cancelled'
```

Cancelled appointments are excluded, so a cancelled slot is offered again.

**Loose end to be honest about if asked:** the session length is not forced to
divide evenly by 30. A 09:00–10:20 session produces 09:00, 09:30 and 10:00, and
the last 20 minutes go unused. The loop condition `$t < $end` means we never
create a slot that starts after the doctor has finished, which is the safe
direction to be wrong in.

---

## 4. Stopping double booking — the strongest point in your module

**The naive approach and why it fails.** You might check in PHP:

```php
if (slot is free) { insert the appointment; }
```

Between the check and the insert, another user's request can run. Both see the
slot as free, both insert, and now two patients have 10:30 with the same doctor.
This is called a **race condition**, and no amount of PHP checking removes it.

**What we do instead.** The database enforces it:

```sql
UNIQUE (Doctor_ID, Appointment_Date, Slot_Key)
```

The second insert is rejected by MySQL itself, no matter how close together the
two requests are. PHP's job is only to turn that rejection into a friendly
message:

```php
} catch (PDOException $e) {
    if ($e->getCode() === '23000') {   // integrity constraint violation
        return false;
    }
    throw $e;
}
```

`23000` is the SQLSTATE code for a constraint violation. Any other error is
re-thrown, because hiding unknown errors would hide real bugs.

**Why `Slot_Key` and not `Time_Slot` directly?** If the UNIQUE were on
`Time_Slot`, a cancelled 10:30 appointment would keep blocking 10:30 forever,
because the row is still there. So:

```sql
Slot_Key TIME GENERATED ALWAYS AS (IF(Status = 'Cancelled', NULL, Time_Slot)) STORED
```

A generated column is calculated by MySQL from other columns in the same row —
we never insert into it. When the status becomes 'Cancelled', `Slot_Key` becomes
NULL automatically. And **a UNIQUE index allows many NULLs**, because in SQL one
NULL is not considered equal to another. So any number of cancelled appointments
can sit on the same slot, while at most one live appointment can.

That one sentence — *UNIQUE treats NULLs as distinct* — is the whole trick, and
it is worth saying it in exactly those words.

---

## 5. Queue numbers

```php
public function create($data)
{
    try {
        $this->pdo->beginTransaction();

        $queueNo = $this->nextQueueNo($data['doctor_id'], $data['date']);

        // INSERT ... VALUES (..., $queueNo, 'Confirmed', ...)

        $id = $this->pdo->lastInsertId();
        $this->pdo->commit();
        return $id;
    } catch (PDOException $e) {
        $this->pdo->rollBack();
        ...
    }
}
```

And `nextQueueNo()`:

```sql
SELECT IFNULL(MAX(Queue_No), 0) + 1
FROM appointment
WHERE Doctor_ID = ? AND Appointment_Date = ? AND Status <> 'Cancelled'
```

Three details to explain:

- **`IFNULL(MAX(...), 0) + 1`** — on the first booking of the day `MAX` returns
  NULL, and NULL + 1 is NULL in SQL, not 1. `IFNULL` turns that NULL into 0 so
  the first patient gets 1.
- **Why a transaction** — the SELECT and the INSERT must behave as one
  indivisible step. Without it, two simultaneous bookings could both read "last
  is 7" and both become 8.
- **Why per doctor per day** — the queue is the line outside one doctor's room on
  one day. A number shared across all doctors would tell the patient nothing.

**An honest limitation, if pushed:** a transaction alone does not fully prevent
two readers under MySQL's default isolation level; a fully watertight version
would use `SELECT ... FOR UPDATE` to lock the rows, or a separate counter table.
Saying this shows you understand what the transaction does and does not
guarantee — much stronger than pretending it is perfect.

---

## 6. Cancelling

```php
public function cancel($id)
{
    $stmt = $this->pdo->prepare(
        "UPDATE appointment
         SET Status = 'Cancelled', Queue_No = NULL
         WHERE Appointment_ID = ?"
    );
    return $stmt->execute([$id]);
}
```

We **update, never delete**. The clinic needs to know a booking existed and was
cancelled — for reports, for spotting patients who repeatedly cancel, and
because `payment` and `medical_record` may point at that appointment; deleting it
would break those foreign keys.

Setting `Queue_No = NULL` takes the patient out of the line. `Slot_Key` becomes
NULL by itself because the status changed, so the slot frees up. One UPDATE, three
effects.

---

## 7. How the pieces talk to each other

```
index.php?page=patient/book        index.php?page=staff/book
          |                                   |
          |  both call ...                    |
          v                                   v
              models/Appointment.php + models/Schedule.php
                              |
                              v
              appointment  /  doctor_schedule tables
                              ^
                              |
   index.php?page=doctor/schedule writes sessions here,
   index.php?page=doctor/dashboard reads today's queue from here
```

Say this plainly in the viva: **the booking rules are written once**. The patient
portal does not have its own copy of the SQL — `PatientController::book()` and
`StaffController::book()` both call the same `Appointment::create()`. If we changed the slot
length or the double-booking rule, both front ends would change together. That is
the reason for putting SQL in models instead of inside pages.

`HomeController::slots()` is the other connection point. The booking page asks:

```
index.php?page=api/slots&doctor_id=1&date=2026-03-15
```

and gets back JSON:

```json
{"slots":[{"schedule_id":7,"time":"09:00:00","label":"09:00 AM"}, ...]}
```

The JavaScript turns each entry into a clickable button. This is why choosing a
different doctor updates the slots without the page reloading.

The radio button's value is `schedule_id|time`, for example `7|09:30:00`, and the
controller splits it:

```php
list($scheduleId, $timeSlot) = array_pad(explode('|', $slot), 2, null);
```

Two values in one form field, so the appointment can record both which session it
came from and the exact time.

---

## 8. The queue page

`StaffController::queue()` loads the day's appointments in queue order and works
out who is being seen:

```php
foreach ($queue as $row) {
    if ($row['Status'] === 'Confirmed' || $row['Status'] === 'Pending') {
        $nowServing = $row;
        break;
    }
}
```

The first person in queue order who is not finished is the one currently being
seen. Simple, and it needs no extra column.

The query behind it:

```sql
SELECT a.*, CONCAT(p.First_Name,' ',p.Last_Name) AS Patient_Name, p.Phone
FROM appointment a
JOIN patient p ON a.Patient_ID = p.Patient_ID
WHERE a.Doctor_ID = ? AND a.Appointment_Date = ? AND a.Status <> 'Cancelled'
ORDER BY a.Queue_No
```

An INNER JOIN is right here, not a LEFT JOIN: an appointment must have a patient,
so there is nothing to preserve.

---

## 9. Questions you should be ready for

**"Why is `Time_Slot` a TIME and not part of a DATETIME?"**
We keep `Appointment_Date` and `Time_Slot` separate because almost every query
groups or filters by date alone — today's queue, the daily report, free slots for
a date. A combined DATETIME would need a function on the column in each of those,
which also makes indexes harder to use.

**"What if the doctor changes a session after people have booked?"**
Closing a session sets `Is_Available = 0`, so no new bookings appear, but existing
appointments stay. We do not automatically cancel them — a human should call
those patients. Worth naming as a deliberate choice, not an oversight.

**"How would you add rescheduling?"**
Update `Appointment_Date` and `Time_Slot` on the existing row inside a
transaction, re-running the queue number for the new day. The UNIQUE constraint
protects the new slot exactly as it protects a new booking. We did not build it —
right now a patient cancels and books again.

**"What does `Receptionist_ID` do?"**
It is there for recording who made a booking at the desk, but nothing fills it
yet. Say that straight; an unused column you can explain is far better than one
you cannot.

**"Show me where SQL injection is prevented."**
Open any model method. Every value is a `?` placeholder passed to `execute()`,
never concatenated into the string. Point at one and read it out.

**"What is the weakest part of your module?"**
Good answers: the queue number needs row locking to be fully safe under heavy
load; sessions that do not divide evenly by 30 minutes waste the remainder; there
is no rescheduling. Naming a real weakness you understand scores better than
claiming there is none.

---

## 10. Two-minute summary if you are asked to just explain your part

"I handle appointments and the queue. A doctor stores a working session, for
example 9 to 12, as one row. When a patient picks a doctor and a date, my code
walks through that session in 30-minute steps and removes anything already booked
or already past, and sends the free slots back as JSON so the page updates without
reloading.

When they confirm, the booking is inserted inside a transaction that also works
out the queue number as the highest number for that doctor that day plus one. Two
patients cannot take the same slot, because the database has a unique constraint
on doctor, date and slot — not just a PHP check, since two requests could pass a
PHP check at the same moment. Cancelling sets the status to Cancelled rather than
deleting the row, and a generated column makes the slot key NULL, which frees the
slot while keeping the history.

The same two model files are used by the patient portal, the reception pages and
the doctor's dashboard, so the booking rules exist in one place only."
