# Hospital Management System: UML

GitHub renders the Mermaid blocks below automatically.

---

## Task 1: Basic class diagram

```mermaid
classDiagram
direction LR

class Patient {
  -int id
  -String name
  -String password
  -String email
  +submitProblem(description) HealthProblem
  +viewPrescription(problemId) Prescription
  +pay(prescription, method) Payment
}

class Doctor {
  -int id
  -String name
  -String password
  -String speciality
  +writePrescription(problem) Prescription
  +receivePayment(amount)
}

class Organizer {
  -int id
  -String name
  -String password
  -String role
  +reviewProblem(problem)
  +sendPrescription(prescription)
  +forwardPayment(payment)
  +createMember(name, role)
}

Patient "0..*" -- "1..*" Organizer : interacts with
Organizer "1..*" -- "0..*" Doctor : consults
Patient "0..*" .. "0..*" Doctor : indirect, via the Organizer
```

---

## Task 2 and 3: Complete class diagram

```mermaid
classDiagram
direction TB

class User {
  <<abstract>>
  #int id
  #String name
  #String password
  #String email
  +login() boolean
  +logout()
}

class Patient {
  -Date birthDate
  +submitProblem(description) HealthProblem
  +viewPrescription(problemId) Prescription
  +pay(prescription, method) Payment
}

class Doctor {
  -String speciality
  -String licenseNumber
  +writePrescription(problem) Prescription
  +receivePayment(amount)
}

class Organizer {
  -boolean isAdmin
  +reviewProblem(problem)
  +assignDoctor(problem, doctor)
  +sendPrescription(prescription)
  +forwardPayment(payment)
  +createMember(name, role) User
  +updateSystem()
}

class ProblemStatus {
  <<enumeration>>
  PENDING
  IN_REVIEW
  RESOLVED
}

class HealthProblem {
  -int id
  -String description
  -ProblemStatus status
  -DateTime createdAt
  -DateTime resolvedAt
  +changeStatus(newStatus)
}

class Prescription {
  -int id
  -String content
  -DateTime issuedAt
  -DateTime sentAt
  +send()
}

class Payment {
  -int id
  -double amount
  -DateTime paidAt
  -DateTime forwardedAt
  -PaymentStatus status
  +process() boolean
  +forwardToDoctor()
}

class PaymentStatus {
  <<enumeration>>
  PENDING
  PAID
  FORWARDED
  FAILED
}

class PaymentMethod {
  <<interface>>
  +pay(amount) boolean
}

class Cash {
  +pay(amount) boolean
}

class Check {
  -String checkNumber
  -String bank
  +pay(amount) boolean
}

class CreditCard {
  -String cardHolder
  -String lastFourDigits
  -Date expiryDate
  +pay(amount) boolean
}

User <|-- Patient
User <|-- Doctor
User <|-- Organizer

Patient "1" --> "0..*" HealthProblem : submits
Organizer "1" --> "0..*" HealthProblem : reviews and assigns
Doctor "0..1" <-- "0..*" HealthProblem : assigned to
HealthProblem --> ProblemStatus

HealthProblem "1" *-- "0..*" Prescription : has
Doctor "1" --> "0..*" Prescription : writes
Prescription "0..1" --> "0..1" Prescription : follows (previous)

Prescription "1" --> "0..1" Payment : paid by
Patient "1" --> "0..*" Payment : makes
Payment --> PaymentStatus
Payment "1" --> "1" PaymentMethod : uses

PaymentMethod <|.. Cash
PaymentMethod <|.. Check
PaymentMethod <|.. CreditCard
```

---

## Task 2: Sequence diagram (submit, prescription, payment)

```mermaid
sequenceDiagram
autonumber
actor P as Patient
participant S as Platform
actor O as Organizer
actor D as Doctor

P->>S: submitProblem(description)
S-->>P: problem created (PENDING)
S->>O: notify new problem
O->>S: reviewProblem(problemId)
O->>D: consult (send problem)
D->>S: writePrescription(problemId, content)
S-->>O: prescription ready
O->>P: sendPrescription(prescription)
P->>S: pay(prescription, method)
S-->>P: payment confirmation
O->>D: forwardPayment(payment)
```

---

## Task 3: System sequence diagrams

### 1. Patient registration (Organizer creates a new member)

```mermaid
sequenceDiagram
autonumber
actor O as Organizer
participant S as System
participant DB as Database

O->>S: createMember(name, email, role)
S->>DB: check email is not already used
DB-->>S: ok
S->>S: generate id and temporary password
S->>DB: save new User
DB-->>S: saved
S-->>O: member created (id, password)
O->>O: give credentials to the patient
```

### 2. Consultation flow (problem submission to prescription delivery)

```mermaid
sequenceDiagram
autonumber
actor P as Patient
participant S as System
actor O as Organizer
actor D as Doctor

P->>S: login(id, password)
S-->>P: authenticated
P->>S: submitProblem(description)
S->>S: create HealthProblem (PENDING)
S-->>P: problem registered
S->>O: notify new problem
O->>S: reviewProblem(problemId)
S->>S: status = IN_REVIEW
O->>S: assignDoctor(problemId, doctorId)
S->>D: notify consultation request
D->>S: writePrescription(problemId, content)
S->>S: create Prescription, status = RESOLVED
S->>O: prescription ready
O->>S: sendPrescription(prescriptionId)
S-->>P: prescription available
```

### 3. Payment processing (patient pays, organizer forwards to doctor)

```mermaid
sequenceDiagram
autonumber
actor P as Patient
participant S as System
participant PM as PaymentMethod
actor O as Organizer
actor D as Doctor

P->>S: pay(prescriptionId, method)
S->>S: create Payment (PENDING)
S->>PM: pay(amount)
alt payment accepted
  PM-->>S: true
  S->>S: status = PAID
  S-->>P: receipt
  S->>O: notify payment received
  O->>S: forwardPayment(paymentId)
  S->>D: receivePayment(amount)
  S->>S: status = FORWARDED
else payment refused
  PM-->>S: false
  S->>S: status = FAILED
  S-->>P: payment failed, try again
end
```

---

## Questions to consider

1. **Several doctors for one patient:** the Organizer can assign several doctors to the same problem, or a patient can submit several problems each handled by a different doctor. For a richer model, add a `Consultation` association class between `HealthProblem` and `Doctor` (with date and opinion).
2. **Reusable prescriptions:** keep a `PrescriptionTemplate`, or link a new prescription to the previous one (the `follows` relation above) so the history stays traceable.
3. **Design pattern for payments:** the **Strategy** pattern. `PaymentMethod` is the strategy interface, and `Cash`, `Check` and `CreditCard` are interchangeable implementations.
4. **Data integrity:** run each payment in a database transaction (all or nothing), use a unique payment id to avoid double charging, check that the amount matches the prescription, and keep the `PaymentStatus` transitions strict (PENDING to PAID to FORWARDED).
