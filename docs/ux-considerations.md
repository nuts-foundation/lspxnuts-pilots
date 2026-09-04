# LSPxNuts Medicatie Overdracht Pilot 1 UX considerations

Design trade-offs for where and when pilot flows surface in the vendor's
product. These aren't infrastructure or protocol questions — see the other
guides for those — and there's no pilot-wide answer: each vendor decides for
their own product and workflow.

## Patient enrollment: upfront or on first query?

Relevant to the `PatientEnrollmentCredential` (participation guide, B.3.c).

> **Note:** issue the `PatientEnrollmentCredential` upfront, as part of the
> patient enrollment workflow (before any data is queried), or lazily on the
> first MEDGEG query for that patient? Each vendor decides based on their own
> workflow:
>
> - **Upfront**: an extra step at intake, but staff already have the UZI card
>   in hand at that point.
> - **Lazy** (on first query): no separate enrollment step, but the
>   smartcard/PIN interaction moves into the first query's latency and error
>   path — the user is mid-task (looking up medication) when interrupted for
>   a card prompt.
