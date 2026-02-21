# ITSM Benchmark

assets/runtime

The runtime folder contains the core benchmark context assets:
- DIAGNOSIS_CATEGORIES.md definitions of possible incident diagnoses.
- DIAGNOSIS_ACTION_GUIDE.md guidance around which actions to take against the services given a specific diagnosis category.
- GENERAL_INFO_SERVICES_DOMAIN.md context around the services domain.
- GENERAL_INFO_TICKET_DOMAIN.md context aroung the ticketing domain.

assets/arena

The arena folder contains specifications of the cloud service application as well as the observability and management tools that are known to the agent for inspecting and interacting with the system.

assets/data

The data folder contains the specification for scenario description files, the scenario descriptions themselves as well as all of the individual scenarios that make up the data set. This includes the mock data required to back the system tools available in the arena.

This folder also contains the ticket_build.json file which is the compelete set of tickets compiled out of the individual scenario files, including the ground truth associated with each ticket.
