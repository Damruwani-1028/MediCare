# Viva Guide — Clinic Appointment & Patient Record Management System

Read your own section carefully. Read section 0 no matter which part you did,
because the examiner will almost certainly ask how the parts fit together.

Rule for the whole viva: if you don't know, say "I'm not sure, but I think..."
and reason out loud. Guessing confidently and being wrong is worse.

---

## 0. What everyone should be able to say

**What the system does.** A patient books an appointment online and gets a queue
number. The doctor sees that patient in their queue, writes down the diagnosis
and prescribes medicines. Reception bills the visit and can see reports.

**How it is organised.** There is one database, `clinic_db`, with 13 tables,
and ONE entry point. Every request in the system looks like
`index.php?page=something` — patient pages, doctor pages and reception pages
alike. `index.php` holds a route table that maps the page name to a controller:

```php
'patient/book'    => ['PatientController', 'book'],
'doctor/schedule' => ['DoctorController',  'schedule'],
'staff/queue'     => ['StaffController',   'queue'],
```

**Three layers (MVC).** The controller takes the input and decides what
happens. The model is the only place SQL is written. The view only prints HTML.
So if a query is wrong you look in `models/`, and if a page looks wrong you look
in `views/`.

**Three roles, one login.** `user_account` holds the username, the hashed
password and `User_Type`. After logging in, the session remembers the role, and
`require_role('Patient')` at the top of a controller blocks anyone else.

**The one journey to memorise.** Patient books → row in `appointment`. Doctor
completes the consultation → row in `medical_record`, appointment becomes
Completed. Doctor prescribes → row in `prescription` plus one row per medicine
in `prescription_item`. Reception bills → row in `payment`. One appointment row
is touched by all three portals.

**Why we use prepared statements.** Every query passes values as `?` parameters
instead of pasting them into the SQL string. That stops SQL injection: if
someone types `' OR 1=1 --` into a form it is treated as text to search for, not
as SQL to run.

---

## Member 1 — Patient module

**Your files:** the `Patient` class in `models/People.php`, `controllers/PatientController.php`, `views/patient/`
**Your table:** `patient` (and you create the login row in `user_account`)

### What your part does
A patient registers, logs in, sees a dashboard, updates their details, browses
doctors, books an appointment, views their appointments, and reads their own
medical history and prescriptions.

### The three things to be able to explain

**1. Registration writes two tables inside one transaction.**
A patient needs a row in `patient` (who they are) and a row in `user_account`
(how they log in). If the first succeeded and the second failed, you would have
a patient who can never log in. So `register.php` does:

```
beginTransaction()
  insert into user_account   -> gives us the new User_ID
  insert into patient        -> stores that User_ID as a foreign key
commit()
```

If anything throws an error, `rollBack()` undoes both. Either the patient exists
completely, or not at all.

**2. Passwords are never stored as typed.**
`password_hash()` turns the password into a long scrambled string before it goes
into the database. At login, `password_verify()` re-checks the typed password
against that string. You cannot reverse a hash, so even someone with a copy of
the database cannot read anyone's password.

**3. A patient can only touch their own data.**
Every query filters by the `Patient_ID` stored in the session at login, not by
anything sent from the browser. In `my_appointments.php`, before cancelling, the
code checks the appointment really belongs to that patient. Otherwise anyone
could change the number in the URL and cancel a stranger's booking.

### Likely questions

- *Why did you split FullName into First_Name and Last_Name?* Because the ERD
  stores them separately, and searching or sorting by last name is impossible if
  the name is one field.
- *What happens if two people register with the same NIC?* The code checks first
  and shows a message, and `NIC` is also `UNIQUE` in the database, so it is
  blocked in two places.
- *Where does the booking logic live?* Not in my pages. `book_appointment.php`
  calls Member 2's `Appointment` model, so a patient booking follows exactly the
  same rules as a reception booking.

---

## Member 2 — Appointments, schedules and queue

**Your files:** `models/Appointment.php`, `models/Schedule.php`, the queue and
booking methods in `controllers/StaffController.php`, `views/staff/queue.php`
**Your tables:** `appointment`, `doctor_schedule`

### What your part does
It turns a doctor's working hours into bookable time slots, stops two patients
taking the same slot, gives each patient a queue number, and shows reception the
live queue for the day.

### The three things to be able to explain

**1. Sessions become slots.**
A doctor does not enter every 30-minute slot by hand. They enter a session, for
example 9:00 to 12:00. `Schedule::freeSlots()` walks through that session in
30-minute steps, removes any slot already booked, removes slots that have
already passed today, and returns what is left. That list is what the patient
sees.

**2. Double booking is stopped by the database, not just by PHP.**
The `appointment` table has:

```sql
UNIQUE (Doctor_ID, Appointment_Date, Slot_Key)
```

Even if two people click at the exact same second, the second insert is rejected
by MySQL. PHP catches error code 23000 and shows "that slot was just taken"
instead of crashing.

`Slot_Key` is a generated column: it equals `Time_Slot` normally, but becomes
NULL when the appointment is cancelled. A UNIQUE index allows repeated NULLs, so
cancelled appointments keep their history while freeing the slot for someone
else. Checking only in PHP would not be enough, because between the check and
the insert another user could slip in.

**3. Queue numbers are generated inside a transaction.**

```sql
SELECT IFNULL(MAX(Queue_No), 0) + 1
FROM appointment
WHERE Doctor_ID = ? AND Appointment_Date = ? AND Status <> 'Cancelled'
```

The SELECT and the INSERT run together inside `beginTransaction()` and
`commit()`, so two people booking simultaneously cannot both read "last number
is 7" and both become 8.

### Likely questions

- *Why is the queue number per doctor per day, not global?* Because the queue is
  the line outside one doctor's room on one day. A global counter would be
  meaningless to the patient.
- *What is `ApiController` for?* It returns free slots as JSON so the booking
  page can load them without reloading. The patient portal and the staff app
  both call the same endpoint.
- *Why `Status <> 'Cancelled'` in the queue query?* A cancelled patient is not
  standing in the line.

---

## Member 3 — Doctors, clinics, specializations, doctor portal

**Your files:** the `Doctor` class in `models/People.php`, `controllers/DoctorController.php`, `views/doctor/`
**Your tables:** `doctor`, `clinic`, `specialization`

### What your part does
A doctor logs in, sets their working sessions, sees today's queue, opens a
patient, writes the consultation record, prescribes medicines, and can search
their past records.

### The three things to be able to explain

**1. Specialization is an entity, not a text field.**
Instead of typing "Cardiology" into every doctor row, there is a
`specialization` table and `doctor.Special_ID` points at it. This means the
spelling is consistent, renaming a specialization is one update, and you can
list all cardiologists with a join. Same idea for `clinic`.

```
SPECIALIZATION 1 ----- M DOCTOR
CLINIC         1 ----- M DOCTOR
```

**2. Writing a record and completing the appointment happen together.**
In `consultation.php`, saving the medical record and setting the appointment to
`Completed` are wrapped in one transaction. A saved record with the appointment
still showing "Confirmed" would be contradictory data.

**3. Sessions feed the patient's booking page.**
`schedule.php` inserts into `doctor_schedule`, the same table Member 2's slot
generator reads. That is the direct link between your portal and theirs. There
is also `UNIQUE (Doctor_ID, Available_Date, Start_Time)` so a doctor cannot
enter the same session twice.

### Likely questions

- *Can one doctor work at several clinics?* Not in our design —
  `doctor.Clinic_ID` allows one. To support many we would need a `doctor_clinic`
  bridge table. We chose the simpler option deliberately.
- *Why doesn't `medical_record` store `doctor_id` and `patient_id`?* Because
  `Appointment_ID` already tells you both. Storing them again would be duplicated
  data that could disagree with the appointment.
- *How do you know the record you are opening is yours?* Every query includes
  `AND a.Doctor_ID = ?` using the ID from the session, so one doctor cannot open
  another doctor's records by changing the URL.

---

## Member 4 — Medical records, prescriptions, medicines

**Your files:** `models/Medical.php` (the MedicalRecord, Prescription and
Medicine classes), the record and medicine methods in `controllers/StaffController.php`
**Your tables:** `medical_record`, `prescription`, `prescription_item`, `medicine`

### What your part does
It stores what happened during a consultation, and what was prescribed. It also
keeps the master list of medicines.

### The three things to be able to explain

**1. One appointment has at most one record.**

```sql
Appointment_ID INT NOT NULL UNIQUE
```

That single word `UNIQUE` is how the 1:0..1 in the ERD becomes real. "At most
one" because a cancelled or upcoming appointment has no record yet. The list of
consultations still waiting to be written up is:

```sql
LEFT JOIN medical_record r ON r.Appointment_ID = a.Appointment_ID
WHERE r.Record_ID IS NULL
```

A LEFT JOIN keeps every appointment even when no record matches; the `IS NULL`
then picks exactly those with nothing on the right.

**2. `prescription_item` is the most important table in your part.**
One prescription can hold many medicines, and one medicine appears on many
prescriptions. That is a many-to-many, and a relational database cannot store it
directly — so we break it with a bridge table:

```
PRESCRIPTION 1 --- M PRESCRIPTION_ITEM M --- 1 MEDICINE
```

Dosage, frequency, duration and quantity live on the **item**, not on the
medicine, because they describe this patient's use of it. Paracetamol has one
row in `medicine` no matter how many patients receive it.

**3. A prescription and its medicines save as one unit.**
`Prescription::create()` opens a transaction, inserts the prescription, then
inserts each medicine line. If no medicine was chosen it calls `rollBack()`, so
the database never holds an empty prescription.

### Likely questions

- *Why not just store the medicine name in the prescription row?* Then the same
  medicine would be typed hundreds of times with different spellings, you could
  not count how often it is prescribed, and renaming it would mean updating every
  row. That is exactly what normalization prevents.
- *Can you delete a medicine?* Only if it has never been prescribed. The code
  counts its uses first; otherwise the page shows "in use". Deleting it would
  break the foreign key from `prescription_item`.
- *How do you find the most prescribed medicine?* `GROUP BY` on the medicine
  with `COUNT` of the items and `SUM` of the quantity.

---

## Member 5 — Payments, user accounts, reports

**Your files:** `models/Billing.php` (the Payment and Report classes),
`models/UserAccount.php`, the payment, report and user methods in
`controllers/StaffController.php`
**Your tables:** `payment`, `user_account`

### What your part does
It bills appointments, prints receipts, manages the logins for all three roles,
and produces the management reports.

### The three things to be able to explain

**1. A visit can exist before the money does.**
`payment.Appointment_ID` is `NOT NULL UNIQUE`, which gives 1:0..1: at most one
bill per visit, and a visit with no bill is perfectly legal, because patients
often pay after the consultation. Unbilled visits are found with the same
`LEFT JOIN ... IS NULL` pattern.

**2. Money is `DECIMAL(10,2)`, never `FLOAT`.**
FLOAT is an approximate type — it rounds. Rounding money loses rupees over
thousands of transactions. DECIMAL stores the exact value. Small detail,
examiners like it.

**3. One login table serves three roles.**
`user_account` holds the username, the hashed password and `User_Type`
(Patient, Doctor or Receptionist). The `patient`, `doctor` and `receptionist`
tables each hold `User_ID` as a foreign key pointing back. So there is one place
to log in, and `User_Type` decides what you are allowed to see. The alternative —
three separate login tables — would mean writing the login logic three times.

### Likely questions

- *Why is refund not the same as delete?* A refunded bill keeps its row and
  amount with the status changed, because the clinic needs the history. The
  UPDATE also carries `AND Payment_Status = 'Paid'`, so you cannot refund money
  that was never taken.
- *Why does a failed login say "wrong username or password" instead of "no such
  user"?* If it said "no such user", anyone could use the form to discover which
  usernames exist.
- *Can you delete any account?* No. An account linked to a patient, doctor or
  receptionist cannot be deleted, and you cannot delete the account you are
  logged in with.
- *Where do the report numbers come from?* All from SQL, not PHP loops:
  `GROUP BY` with `COUNT` and `SUM`, `CASE WHEN` inside `SUM` to count only paid
  bills, a sub-query for the percentage share, and `DATEDIFF` for how old an
  unpaid bill is.

---

## Questions the examiner may ask the whole group

**"What was the hardest part?"** Merging. Three people had built on three
different databases with different table and column names, so we had to agree on
one schema and rewrite the queries in two portals to match it.

**"What would you add next?"** Rescheduling an appointment, part payments, and
recording which receptionist made a booking — the column exists but nothing
fills it yet.

**"Is it secure?"** Prepared statements everywhere against SQL injection, hashed
passwords, `htmlspecialchars` on output against XSS, and session checks on every
protected page. It is not production-ready — there is no HTTPS, no rate limiting
on login, and no audit log.

**"Show us that it is really one system."** Book as a patient, log in as the
doctor and complete that same appointment, log back in as the patient to see the
diagnosis appear, then bill it from reception. Practise this once before the
viva so nobody is clicking around looking for a page.
