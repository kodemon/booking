# Booking

A event driven booking system written in .NET/C#

## System Domains

Core domain modules of the booking system:

* **Catalog**: Manages bookable resources (rooms, equipment, services) and resource metadata.
* **Reservation**: Orchestrates reservation lifecycle, state transitions, and booking persistence.
* **Principal**: Handles user authentication, authorization, and role-based permissions (ABAC + RBAC) using Cerbos.
* **Calendar**: Computes availability, manages time slots, and handles calendar constraints/conflicts.
* **Payment**: Processes transactions, manages invoices, and integrates with third-party providers.
* **Notification**: Dispatches asynchronous alerts (Email/SMS) triggered by booking events.
* **Review**: Collects and stores post-service ratings and user reviews.
