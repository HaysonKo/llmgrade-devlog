# What broke when I tried to score policy answers with rules

## Context

I started LLMGrade with a narrow refund-policy demo because I wanted to test one product problem clearly: can I catch a support-bot regression before it ships?

The source policy was intentionally simple:

Returns are accepted within 30 days of delivery. The item must be unused and in original condition. No exceptions are listed.

That gave me a crisp failure mode. If a candidate answer says the customer can return something after 45 days because of a medical issue, it sounds helpful, but it changed the policy. That is exactly the kind of regression I want LLMGrade to catch.

## Test

I started with deterministic scoring because it was the cheapest way to make the first set of product invariants testable.

I was not trying to prove that rules can judge every LLM answer. I was trying to answer a smaller question: which failures are obvious enough that they should never need a model judge?

For this refund-policy pack, those hard-gate failures included:

* Changing 30 days to 45 days
* Changing delivery to purchase
* Changing calendar days to business days
* Allowing used items when the policy requires unused items
* Weakening original condition to good condition
* Inventing medical, manager, holiday, gift, store-credit, or exchange exceptions
* Treating “no exceptions listed” as permission to invent one
* Producing gibberish or an irrelevant answer

Those are not subtle style preferences. They are product-policy failures.

## Observation

The deterministic scorer worked well for some failures. It caught obvious hallucinations like invented return windows and unsupported medical exceptions.

Then I started stress-testing it with smaller wording changes.

The most useful bug was this pair:

Bad answer:

> We can make an exception.

Good answer:

> We can't make an exception.

A naive scorer can see the same phrase in both answers: “make an exception.”

But as a product decision, those two answers are opposites. One invents unsupported eligibility. The other correctly refuses to create an exception that the source policy does not list.

That bug was the point where the work became more interesting. The problem was not just “add more examples.” The problem was that the evaluator needed to understand polarity.

## Problem

The first version was too close to keyword matching.

It could detect that an answer mentioned an exception, but it could not always tell whether the answer was allowing the exception or denying it.

That matters because support-bot regressions often hide in small language shifts:

* “Support can approve an exception”
* “Support cannot approve an exception”
* “A manager can override the policy”
* “A manager override is not supported by the policy”
* “The policy allows medical exceptions”
* “The policy does not allow medical exceptions”

The words are similar. The product outcome is not.

This is why eval packs need adversarial minimal pairs. They force the scorer to distinguish the exact behavior the product cares about.

## Product decision

I decided to keep deterministic scoring, but narrow its job.

The deterministic layer should not try to understand every possible support answer. It should catch crisp policy violations and create useful failure data.

So I moved the refund-policy evaluator toward explicit flags:

* Does the answer affirm an unsupported exception?
* Does it deny an unsupported exception?
* Does it change the return window?
* Does it change delivery to purchase?
* Does it weaken unused or original-condition requirements?
* Does it preserve the “and” relationship across required conditions?
* Does it answer the user’s question?
* Does it handle missing or conflicting source information carefully?

That made the evaluator less like a generic text scorer and more like a product-specific regression check.

## Result

The refund-policy pack now has a larger stress suite with direct tests for source wording, user questions, correct answers, hallucinations, subtle wrong answers, over-refusals, gibberish, missing information, conflicting source truth, and exception-polarity minimal pairs.

The “can” versus “can’t” bug now behaves the way I expect:

* “We can make an exception” hard fails
* “We can't make an exception” passes as safe but incomplete
* “A manager can override the policy” hard fails
* “A manager override is not supported by the policy” scores higher
* “Since exceptions are not forbidden, they are allowed” hard fails
* “Since no exceptions are listed, I should not invent one” passes

I like that bare denials do not get a perfect score. “We can't make an exception” is safe, but it is not as complete as an answer that cites the actual policy: 30 days from delivery, unused, original condition, and no listed exceptions.

That distinction matters. A good eval should separate unsafe from safe, but it should also separate safe-but-thin from actually useful.

## Remaining gaps

This work also made the limits clearer.

Rules are useful for crisp policy invariants. They are weaker for open-ended semantic judgment.

I do not want LLMGrade to become an infinite pile of regex. The deterministic layer is useful because it shows me where the product failures are. It does not replace every other kind of judging.

The next architecture probably needs a few layers:

1. Deterministic hard gates for crisp policy violations
2. Structured fact extraction from the source and the answer
3. A model-backed judge for semantic interpretation
4. Regression or ranking signals once there are enough labeled eval cases

That is the real product direction: not one universal score, but app-specific eval packs that define what failure means for a specific workflow.

## Next step

The next product step is custom eval file support.

Right now, the refund-policy pack is still the bundled example. That is useful for proving the shape of the workflow, but the product becomes more real when a developer can bring their own source truth, questions, baseline answers, and candidate answers.

After that, the next steps are:

* Make the local CLI work with custom eval files
* Add a GitHub Action / pre-merge gate
* Add a semantic judging layer for cases rules should not own
* Build more eval packs beyond refund policy

The main lesson from this round is simple: deterministic evals did not solve the whole problem, but they helped me find the right problem to solve next.
