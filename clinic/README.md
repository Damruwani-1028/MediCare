# MediCare Clinic — Clinic Management System

## What is included
- Patient, Doctor and Receptionist role-based portals
- Modern responsive UI with light/dark mode
- Appointment booking with live time-slot availability
- Live queue management and appointment status
- Patient and doctor directories with search
- Medical records and consultation workflow
- Digital prescriptions and medicine master list
- Billing, payment, refund and printable receipts
- Reports and analytics with visual summaries
- Responsive mobile navigation and print-friendly pages
- Password hashing and session protection

## Run locally with WAMP
1. Extract the `clinic` folder into `C:\wamp64\www\`.
2. Start Apache and MySQL in WAMP.
3. Import `database/clinic_db.sql` into MySQL.
4. Check `config/database.php` and make sure the MySQL username, password and port match your WAMP/MySQL installation.
5. Open `http://localhost/clinic/` in Edge/Chrome.

## Demo accounts
- Patient: `kamal@mail.com` / `patient123`
- Doctor: `dr.perera` / `doctor123`
- Reception: `reception` / `admin123`

## Important
The SQL file recreates `clinic_db`, so import it only when you are happy to reset the sample database.
