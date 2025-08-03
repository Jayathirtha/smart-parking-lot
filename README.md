# smart-parking-lot
Low-Level Architecture Design for a Smart Parking Lot Backend System

This document outlines the low-level architecture for a smart parking lot backend system, focusing on vehicle entry/exit, parking space allocation, and fee calculation. The design emphasizes modularity, scalability, and data integrity.

1. System Overview

The backend system acts as the central for the smart parking lot.

 Core Modules and Their Responsibilities

a) Vehicle Management Service

Responsibility: Manages vehicle profiles, Entery System data, entry/exit timestamps, and current parking status.

Key Data Models:
   
    * Vehicle: vehicle_id (PK, UUID), license_plate_number (Unique), vehicle_type (e.g., Car, Motorcycle), registered_user_id (FK), created_at, updated_at.
    * ParkingSession: session_id (PK, UUID), vehicle_id (FK), entry_time, exit_time , assigned_spot_id (FK), status (e.g., 'ACTIVE', 'COMPLETED', 'CANCELLED'), total_fee, payment_status (e.g., 'PENDING', 'PAID', 'REFUNDED').
    * Log: log_id (PK, UUID), license_plate_number, timestamp, event_type (e.g., 'ENTRY_DETECTED', 'EXIT_DETECTED').
	
Interaction: Communicates with Parking Space Management Service for spot allocation and Fee Calculation Service for fee determination.

b) Parking Space Management Service

Responsibility: Maintains the real-time status of all parking spots, allocates available spots, and updates their occupancy.

Key Data Models:
  
    * ParkingLot: lot_id (PK, UUID), name, address, total_spots, available_spots.
    * ParkingZone: zone_id (PK, UUID), lot_id (FK), name (e.g., 'Level 1', 'EV Charging'), total_spots, available_spots.
    * ParkingSpot: spot_id (PK, UUID), zone_id (FK), spot_number, type (e.g 'STANDARD', 'EV', 'HANDICAPPED'), status (e.g., 'AVAILABLE', 'OCCUPIED', 'RESERVED', 'OUT_OF_SERVICE'), is_occupied (boolean).

Interaction: Receives requests from Vehicle Management Service for spot allocation. Receives updates from parking occupancy.

c) Fee Calculation Service

Responsibility: Calculates parking fees based on duration, parking zone, vehicle type, and predefined pricing rules.

Key Data Models:
    
    * PricingRule: rule_id (PK, UUID), lot_id (FK), zone_id (FK), vehicle_type, start_time, end_time, rate_per_hour, minimum_fee, daily_cap.

Interaction: Called by Vehicle Management Service during vehicle exit to determine the total fee.


Example Flow: Vehicle Entry

1.  Entry System: Detects a vehicle's license plate at the entry barrier.

2.  Backend service Receives a request (e.g., `POST /vehicles/entry`) from the Entry system, containing the `license_plate_number`.

3.Vehicle Management Service:

    * Validates the request.
    * Creates a new `ParkingSession` record with `entry_time` and `status = 'ACTIVE'`.
    * Calls `Parking Space Management Service` (`POST /spots/allocate`) to get an available spot.
    * Updates the `ParkingSession` with `assigned_spot_id`.

4.Parking Space Management Service:

    * Receives the allocation request.
    * Finds an available `ParkingSpot` (e.g., using `GET /spots/available` internally).
    * Updates the `ParkingSpot` status to 'OCCUPIED', based on the vehicle’s size (e.g., motorcycle, car, bus).
    * Returns the `spot_id`.

Example Flow: Vehicle Exit and Fee Calculation

1.  Exit System Detects vehicle at exit.

2.  Backend service Receives a request (e.g., `POST /vehicles/exit`) containing `license_plate_number` or `session_id`.

3.Vehicle Management Service:

    * Retrieves the active `ParkingSession`.
    * Updates `exit_time` and `status = 'COMPLETED'`.
    * Calls `Fee Calculation Service` (`GET /fees/calculate`) to determine the fee.
    * Updates `total_fee` in `ParkingSession`.
    * Initiates payment via `Payment Processing Service` (`POST /payments/initiate`).
	
4.Fee Calculation Service:

    * Receives `entry_time`, `exit_time`, `vehicle_type`, `assigned_spot_id`.
    * Retrieves relevant `PricingRule` based on these parameters.
    * Calculates the `total_fee`.
    * Returns the calculated fee.
	
5.Payment Processing Service:

    * Initiates payment with the external gateway.
    * Updates `PaymentTransaction` status.
  
6.Vehicle Management Service (after successful payment):

    * Updates `payment_status` in `ParkingSession`.
    * Calls `Parking Space Management Service` (`POST /spots/deallocate`).
	
7.Parking Space Management Service:

    * Receives de-allocation request.
    * Updates `ParkingSpot` status to 'AVAILABLE'.

This low-level architecture provides a robust, scalable, and maintainable foundation for a smart parking lot backend system.
