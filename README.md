# Smart Hospital Appointment & Queue Management System

A RESTful backend built with Spring Boot, Spring Data JPA, and MySQL for managing
hospital appointments, doctor schedules, and patient queues.

## Tech Stack
- Java 17
- Spring Boot 3.2 (Spring Web, Spring Data JPA, Validation)
- MySQL 8
- Maven
- Lombok

---

## 1. Prerequisites

- JDK 17+ installed (`java -version`)
- Maven installed (`mvn -version`) — or use the included `mvnw` wrapper if you add one
- MySQL 8 running locally, with a user that can create databases

## 2. Setup

1. Clone/open this project folder.
2. Open `src/main/resources/application.properties` and set your real MySQL
   username/password:
   ```
   spring.datasource.username=root
   spring.datasource.password=yourpassword
   ```
   The database `hospital_db` will be created automatically on first run
   (`createDatabaseIfNotExist=true`), and Hibernate will create all 5 tables
   for you (`spring.jpa.hibernate.ddl-auto=update`).

3. Build and run:
   ```bash
   mvn clean install
   mvn spring-boot:run
   ```
   The API will start on `http://localhost:8080`.

## 3. Database Schema (5 tables)

```
Specialty (id, name)
    |
    | one-to-many
    v
Doctor (id, name, email, specialty_id)
    |
    | one-to-many
    v
TimeSlot (id, doctor_id, slot_date, start_time, end_time, booked)
    ^
    | one-to-one
    |
Appointment (id, patient_id, time_slot_id, status, queue_number, created_at)
    ^
    | many-to-one
    |
Patient (id, name, email, phone)
```

**Why TimeSlot is separate from Appointment:** a slot represents doctor
availability and can exist before anyone books it. This separation is what
makes the "real-time availability check" possible — availability is just
`SELECT * FROM time_slots WHERE booked = false`.

---

## 4. Full API Reference (17 endpoints)

| Method | Endpoint | Description |
|---|---|---|
| POST | `/api/specialties` | Create a specialty |
| GET | `/api/specialties` | List specialties |
| POST | `/api/doctors` | Register a doctor |
| GET | `/api/doctors?specialtyId=` | List doctors (optionally by specialty) |
| POST | `/api/patients` | Register a patient |
| GET | `/api/patients` | List patients |
| POST | `/api/timeslots` | Create a time slot for a doctor |
| GET | `/api/doctors/{doctorId}/slots?date=YYYY-MM-DD` | Get available slots |
| POST | `/api/appointments/book` | Book an appointment |
| PUT | `/api/appointments/{id}/cancel` | Cancel an appointment |
| PUT | `/api/appointments/{id}/reschedule` | Reschedule to a new slot |
| PUT | `/api/appointments/{id}/complete` | Mark appointment completed |
| PUT | `/api/appointments/doctors/{doctorId}/cancel-day?date=` | Cascading cancellation |
| GET | `/api/appointments/patient/{patientId}` | Patient's appointment history |
| GET | `/api/appointments/doctor/{doctorId}?date=` | Doctor's day list / queue |

**HTTP status codes used deliberately:**
- `201 Created` — POST that creates a resource
- `200 OK` — successful GET/PUT
- `400 Bad Request` — validation failure or illegal state transition
- `404 Not Found` — ID doesn't exist
- `409 Conflict` — slot already booked (double booking attempt)
- `500 Internal Server Error` — unexpected failure (caught by the global handler so no raw stack trace ever reaches the client)

---

## 5. Walkthrough — try the full lifecycle with curl

```bash
# 1. Create a specialty
curl -X POST http://localhost:8080/api/specialties \
  -H "Content-Type: application/json" \
  -d '{"name":"Cardiology"}'
# -> {"id":1,"name":"Cardiology"}

# 2. Register a doctor
curl -X POST http://localhost:8080/api/doctors \
  -H "Content-Type: application/json" \
  -d '{"name":"Dr. Rao","email":"rao@hospital.com","specialtyId":1}'
# -> {"id":1,"name":"Dr. Rao","email":"rao@hospital.com","specialtyName":"Cardiology"}

# 3. Register a patient
curl -X POST http://localhost:8080/api/patients \
  -H "Content-Type: application/json" \
  -d '{"name":"Amit Sharma","email":"amit@example.com","phone":"9876543210"}'
# -> {"id":1,"name":"Amit Sharma",...}

# 4. Create a time slot for the doctor
curl -X POST http://localhost:8080/api/timeslots \
  -H "Content-Type: application/json" \
  -d '{"doctorId":1,"date":"2026-07-10","startTime":"10:00:00","endTime":"10:30:00"}'
# -> {"id":1,"doctorId":1,...,"booked":false}

# 5. Check availability
curl "http://localhost:8080/api/doctors/1/slots?date=2026-07-10"

# 6. Book the appointment
curl -X POST http://localhost:8080/api/appointments/book \
  -H "Content-Type: application/json" \
  -d '{"patientId":1,"timeSlotId":1}'
# -> {"id":1,...,"status":"BOOKED","queueNumber":1}

# 7. Try booking the SAME slot again with a second patient -> 409 Conflict
curl -X POST http://localhost:8080/api/appointments/book \
  -H "Content-Type: application/json" \
  -d '{"patientId":2,"timeSlotId":1}'
# -> 409 {"status":409,"error":"Conflict","message":"This slot is already booked..."}

# 8. Mark it completed
curl -X PUT http://localhost:8080/api/appointments/1/complete

# 9. Or, on another appointment, cancel it instead
curl -X PUT http://localhost:8080/api/appointments/2/cancel
```

Import `postman_collection.json` into Postman to run all of these with a UI
instead of curl.

---


## 6. Project Structure

```
src/main/java/com/hospital/appointment/
  entity/          Specialty, Doctor, Patient, TimeSlot, Appointment, AppointmentStatus
  repository/       JpaRepository interfaces (Spring Data generates the SQL)
  dto/              Request/response shapes exposed by the API
  exception/        Custom exceptions + GlobalExceptionHandler
  service/          Business logic, transaction boundaries
  controller/       REST endpoints
  HospitalAppointmentSystemApplication.java
```
