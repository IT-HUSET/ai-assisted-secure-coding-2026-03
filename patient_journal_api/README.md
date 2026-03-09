# Patient Journal System (1st phase)

This is the first phase of a patient journal system, on its first phase the system will provide functionality to start feeding patient information from the old system, this will occur via machine-to-machine communication (api integration) between the two systems.

## Functional requirements

The system should:

- Expose an endpoint to submit a new patient record
- Expose an endpoint to query/consult a patient profile / personal information
- Expose an endpoint to ingest a patient existing medical history
- Expose an endpoint to submit a new encounter/visit
- Expose an endpoint to get the medical history of a patient

## Data Security

Data ought to be protected during transit and at rest given the the nature of the information and the amount of PII that the system processes.
