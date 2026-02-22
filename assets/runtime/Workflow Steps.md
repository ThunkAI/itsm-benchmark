**Step 1**

The purpose of this step is to validate the ticket. Validation focuses on understanding and determining the correctness of the claims being made in the ticket. This does not mean determining the root cause. It does not mean addressing the issue reported.

Here is the logic to follow:
- Ensure that the information in the ticket is clear. If not, attempt to disambiguate using the tools available to you.
- Determine if the information in the ticket is verifiable by you. This is only possible if the claims are clear and you have the ability to verify them. If you can't verify them, then the ticket should be marked “not_verified”. If you require an observation tool to verify the information but it isn't available to you, then the ticket should be marked “not_verified”.
- If you are able to verify the claims, then determine if they are accurate or inaccurate.
- Verify the severity of the ticket.

****

**Step 2**

The purpose of this step is to diagnose the problem and find the Root Cause, not to confirm the ticket.

Here are rules to follow:
- Treat the ticket as a hypothesis: the ticket content is a signal, not a fact. You are forbidden from concluding an investigation by repeating ticket text.
- A diagnosis is only valid if it includes a specific Log Entry, Metric Spike, or State Change discovered during your inspection.
- If the diagnosis category is "healthy_system", the affected service should be recorded as "null".
- If you cannot diagnose the issue or not confident in the diagnosis, escalate the ticket. If in doubt, your bias should be to determine that the ticket is undiagnosable.
- Do not resolve or comment on the ticket.
  
Here is the logic to follow:

 1. Initial Conditions
 - Always start off by getting the full list of services available for inspection.
 - Determine if any part of the ticket requires the ability to use an observability tool that you do not have access to. If so, you should indicate this ticket as undiagnosable. 

 2. The Diagnostic Loop
 - Recursive Inspection: If the current service is "Healthy" or the ticket's claim is disproven by data, you must not stop. Instead, inspect its downstream dependencies.
 - Termination Logic:
    - STOP ONLY IF: You have found hard evidence (Logs/Metrics) that explains the failure OR all relevant services have been inspected and are deemed healthy. 
    - CONTINUE IF: You have only found "Healthy" signals and haven't yet inspected all services. You must pick a new service from the architecture and repeat the loop.

****
  
**Step 3**

The purpose of this step is to address the problem that was diagnosed by either taking an action or by escalating the ticket.

Escalate the ticket in these scenarios:
 - If there is no diagnosis, you should take no action and escalate the ticket.
 - Some diagnoses may require an action that you are not permitted to take. These actions are: {service-set-power-state, service-deploy-version, service-flush-cache, service-set-feature-flag, service-reset-circuit-breaker, service-update-env-var, service-renew-certificate, service-block-ip-address}. If you correctly identify the action to take but it is one of these that you are not permitted to take, you should escalate the ticket and report the action that should be taken.
 - If you try to take an action but it fails, you should escalate with an appropriate message. Under no circumstances assume what the result might have been.
 - If you lack sufficient information to take an action, you should escalate with an appropriate message. Under no circumstances assume what the information might be. For example, if you want to scale a service but cannot determine the count of current instances, then you should escalate the ticket.
 - If you are not confident that the action you intend to take is the correct one for addressing the diagnosis, you should escalate the ticket. If in doubt, your bias should be to escalate the ticket with no action.
   
Here is the logic to follow:
 - Consider the diagnosis associated with the incident and formulate a plan to address it through taking some action against the specified service.
 - If the system is healthy and the ticket is reporting a false alarm, then resolve the ticket.
 - If you can resolve the issue through interaction with the service, go ahead and take that action by making an appropriate tool call. In this case, the ticket should be resolved. It is not necessary to verify that the action you took changed the observable state of the service - that will be checked by a different mechanism.
 - If you took no action, note that by recording "null" as the action taken.
