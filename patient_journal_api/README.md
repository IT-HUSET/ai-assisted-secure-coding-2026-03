# Patient Journal System (1st phase)

This is the first phase of a patient journal system, on its first phase the system will provide functionality to start feeding patient information from the old system, this will occur via machine-to-machine communication (api integration) between the two systems.

The new solution is containerized, we are going to be building our own custom images and store them in a container registry.

## Functional requirements

The system should:

- Expose an endpoint to submit a new patient record
- Expose an endpoint to query/consult a patient profile / personal information
- Expose an endpoint to ingest a patient existing medical history
- Expose an endpoint to submit a new encounter/visit
- Expose an endpoint to get the medical history of a patient

## Non-functional requirements

- Application: Containerized solution, selection of secure base container images, as secure as possible without sacrificing much functionality
- Application: any backend service should not be reachable from the Internet.
- Network: Secure communication betweent the old and new system, to prevent tampering or eavesdropping.
- Network/application: New system must work only via HTTPs
- Application: Secrets, keys and certificates must not be stored in application configuration rather on a secrets/key/certficate manager solution.
- RBAC: Controlled/restricted access to the secrets manager.
- Data security: Data ought to be protected during transit and at rest given the the nature of the information and the amount of PII that the system processes and stores.
