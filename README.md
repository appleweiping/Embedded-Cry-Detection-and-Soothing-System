## Overview

This project presents a fully integrated embedded system designed to automatically detect infant distress (crying) and respond with appropriate soothing actions. The system combines signal processing, decision-making algorithms, and hardware control to reduce crying duration efficiently.

The entire system — including architecture design, implementation, debugging, and integration — was independently developed.

## Key Achievement

- Successfully reduced infant crying duration to under **2 minutes** through automated intervention.
- Demonstrated real-time responsiveness across multiple subsystems.

---

## System Architecture

The project consists of several interconnected modules:

### 1. Cry Detection Module
- Processes audio input to detect crying patterns
- Uses signal thresholding and pattern recognition
- Outputs a crying intensity level

### 2. Decision-Making Module
- Determines the appropriate soothing response
- Implements a rule-based strategy based on crying intensity and duration
- Handles system coordination and state transitions

### 3. Motor Control Module
- Drives physical motion (rocking mechanism)
- Responds dynamically to control signals from the decision module

### 4. Heartbeat / Monitoring Module
- Provides system feedback and status monitoring
- Ensures system reliability during operation

### 5. Display / Interface
- Visualizes system state
- Provides debugging and real-time feedback

---

## Technical Highlights

- Embedded C programming across all subsystems
- Real-time system coordination using modular architecture
- Hardware-software co-design
- Signal processing for audio-based detection
- Robust debugging and iterative system refinement

---

## Project Structure


├── crying/ # Cry detection module (audio processing)
├── decision/ # Decision-making logic
├── Parts/ # Hardware design files (CAD models)
├── ... # Other supporting modules


---

## Current Status

- Core functionality: ✅ Completed
- System integration: ✅ Completed
- Cry detection: ⚠️ Functional but requires further robustness improvements
- Hardware interaction: ✅ Stable
- Decision logic: ✅ Working

---

## Future Improvements

- Improve reliability of cry detection algorithm
- Enhance decision-making with adaptive or learning-based methods
- Refine UI/UX for display module
- Add safety mechanisms (e.g., emergency stop across all modules)
- Optimize motor control smoothness

---

## Notes

This project was developed as a complete end-to-end system, covering:
- System design
- Embedded programming
- Hardware integration
- Testing and optimization

---

## Author

Developed independently as part of an embedded systems project (2025).