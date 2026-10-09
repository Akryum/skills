# Generic Test Examples

These are framework-neutral pseudocode, not runnable files. Names such as `application`, `command`, and `mockDatabase` stand for existing production entries and shared fixtures in your project. Keep those production entries real; mock only permitted external boundaries. Translate assertions into your existing test runner.

Each example states a user contract, the exact proof boundary, and a plausible defect that should fail. Select examples relevant to the change rather than adding every scenario.

## 1. Public function: inclusive interval

Contract: API consumer receives exactly the events within both inclusive bounds, in input order.

Weak:

```text
assert length(eventsInRange(events, 0, 1000)) == length(events)
```

Stronger:

```text
events = [
  {id: "end", time: 20},
  {id: "before", time: 9},
  {id: "start", time: 10},
  {id: "after", time: 21},
  {id: "middle", time: 15}
]

assert eventsInRange(events, 10, 20) == [
  {id: "end", time: 20},
  {id: "start", time: 10},
  {id: "middle", time: 15}
]
```

Faults: remove upper bound, make either endpoint exclusive, or sort instead of preserving order. Complete expected values also reject dropped identifiers.

Boundary: real exported function; no dependency mock required.

## 2. Update through an allowed database mock

Contract: changing one document title targets that document, preserves unspecified fields, and returns its updated public representation.

Weak:

```text
mockDatabase.update returns {id: "doc-1", title: "New title"}
await application.renameDocument("doc-1", "New title")
assert mockDatabase.update was called
```

Stronger:

```text
mockDatabase.expectUpdate(
  table = "documents",
  predicate = {id: "doc-1"},
  values = {title: "New title"},
  returns = {id: "doc-1", title: "New title", owner: "alice"}
)

result = await application.renameDocument("doc-1", "New title")

assert result == {id: "doc-1", title: "New title", owner: "alice"}
mockDatabase.verifyExpectedRequests()
```

Faults: write empty values, target another ID, update owner accidentally, or discard a returned field. Exact boundary expectation must reject mismatched requests, not return canned success anyway.

Expected objects must be independent of mutable inputs and DB responses:

```text
input = {id: "doc-1", title: "New title"}
mockDatabase.returnedRow = {id: "doc-1", title: "New title", owner: "alice"}
mockDatabase.onUpdate = payload => recordSnapshot(payload)

result = await application.updateDocument(input)

assert input == {id: "doc-1", title: "New title"}
assert recordedWrites() == [{id: "doc-1", title: "New title"}]
assert result == {id: "doc-1", title: "New title", owner: "alice"}
```

Weak alternatives `recordedWrites() == [input]` or `result == mockDatabase.returnedRow` can stay green if production mutates those shared objects. Preserve caller input only when that is part of the API contract; request and returned-field expectations still need independent values.

Boundary: production application/database adapter, mocked DB. This proves request construction and response handling, not SQL execution or actual storage. If simulated read-after-write matters, reuse a stateful external fixture seeded with two documents and check both after the action.

## 3. Payment service: correct charge and failed-charge recovery

Contract: checkout charges correct amount/currency and marks an order paid only after payment succeeds.

```text
payments.expectCharge(
  orderId = "order-1", amountMinor = 1250, currency = "EUR",
  returns = {paymentId: "payment-1", status: "succeeded"}
)

result = await checkout.pay("order-1")

assert result == {orderId: "order-1", status: "paid", paymentId: "payment-1"}
assert await orders.read("order-1").status == "paid"
payments.verifyExpectedRequests()
```

Separate declined scenario:

```text
payments.nextCharge rejects PaymentDeclined

assert await checkout.pay("order-2") rejects PaymentDeclined
assert await orders.read("order-2").status == "unpaid"
assert await orders.read("order-2").paymentId == null
```

Faults: wrong currency, wrong money unit, wrong order, or marking paid before success. Do not mock checkout or its owned payment-state transition.

Boundary: real checkout logic with external payment and optionally DB mocks. No live payment account required.

## 4. Authorization: denied effects stay absent

Contract: member cannot delete another organization's document; owner can.

```text
seed documents = [
  {id: "a", organization: "north", title: "Owned"},
  {id: "b", organization: "south", title: "Other"}
]

denied = await application.requestAs(northMember, "DELETE", "/documents/b")

assert denied.status == 403
assert await documentReader.read("b") == {
  id: "b", organization: "south", title: "Other"
}
assert externalDeletionRequestsFor("b") == []

allowed = await application.requestAs(northMember, "DELETE", "/documents/a")
assert allowed.status == 204
assert await documentReader.read("a") == null
```

Fault: bypass authorization but still return 403 after deletion. Status assertion alone misses the forbidden effect. Fixtures may supply identity-provider and persistence dependencies; execute real authorization rules and request handler.

## 5. CLI export: artifact contents, not existence

Contract: users can consume exported records, including commas and quotes, with an ordinary CSV reader.

```text
seed records = [
  {id: "a", name: "Alice"},
  {id: "b", name: 'Bob, "Builder"'}
]

process = runActualCommand(["export", "--output", isolatedPath])
rows = standardCsvReader.read(isolatedPath)

assert process.exitCode == 0
assert rows == [
  {id: "a", name: "Alice"},
  {id: "b", name: 'Bob, "Builder"'}
]
assert sourceRecords() == records
```

Faults: exit successfully without exporting, omit last record, lose identifier, or fail escaping. Do not mock command entry or use exporter itself as expected-output parser.

Boundary: shipped command and real output file; source remote service/DB may be mocked. The reader is independent of exporter.

## 6. Background job: delivery can repeat

Contract, when specified: duplicate delivery of one job does not duplicate its external effect; later distinct job still works.

```text
await worker.handle({jobId: "job-a", recipient: "alice", message: "First"})
await worker.handle({jobId: "job-a", recipient: "alice", message: "First"})
await worker.handle({jobId: "job-b", recipient: "bob", message: "Second"})

assert mailService.deliveredMessages() == [
  {recipient: "alice", message: "First"},
  {recipient: "bob", message: "Second"}
]
assert await jobReader.status("job-a") == "completed"
assert await jobReader.status("job-b") == "completed"
```

Faults: remove duplicate handling or use a global flag that skips all later jobs. Mail and persistence may be external mocks; worker and idempotency logic stay real. Do not assert exactly-once semantics unless the product actually promises them.

## 7. UI save: optimistic display and rollback

Contract: rejected save preserves previously saved value, informs user, and allows retry.

```text
openActualEditor(document = {id: "a", title: "Original"})
externalApi.holdNextUpdate()
user.editTitle("Changed")
user.click("Save")

assert displayedTitle() == "Changed"  # Only optimistic state so far.
externalApi.rejectHeldUpdate(503)
await editorSaveSettled()

assert displayedTitle() == "Original"
assert visibleError() == "Save failed"

externalApi.acceptNextUpdate({id: "a", title: "Changed"})
user.editTitle("Changed")
user.click("Save")
await editorSaveSettled()
assert displayedTitle() == "Changed"
assert visibleError() == null
```

Faults: swallow error without rollback, show permanent false success, or prevent retry. Use actual UI lifecycle and accessible controls; adapt copy to product rather than requiring this literal message. API mock is allowed. A separate reopened read can verify simulated persistence if fixture maintains it.

## 8. Resource lifecycle: no stale delivery after shutdown

Contract: closed consumer receives no updates; a new consumer can receive them.

```text
first = startActualConsumer(externalEvents)
externalEvents.publish({id: "initial", value: 1})
await deliverySettled()
assert first.visibleEvents() == [{id: "initial", value: 1}]

await first.closeThroughPublicLifecycle()
second = startActualConsumer(externalEvents)
externalEvents.publish({id: "later", value: 2})
await deliverySettled()

assert first.visibleEvents() == [{id: "initial", value: 1}]
assert second.visibleEvents() == [{id: "later", value: 2}]
```

Faults: skip unsubscribe, retain stale callback, or close shared connection needed by new consumer. External broker/event service may be mocked. Real consumer registration and disposal must execute. The settlement helper must observe actual delivery completion, not an arbitrary sleep.

For storage promising shared physical connection leases, peer reads alone can hide premature close followed by automatic reopen. First ensure both consumers acquire the resource; constructing a lazy handle does not acquire it:

```text
first = await actualStorage.open("workspace")
second = await actualStorage.open("workspace")
await first.write({id: "a", title: "Saved"})
assert await second.read("a") == {id: "a", title: "Saved"}
await first.dispose()

maintenance = database.beginExclusiveMaintenance("workspace")
assert await maintenance.firstOutcome() == "blocked"
assert await second.read("a") == {id: "a", title: "Saved"}
await second.dispose()
assert await boundedContractOutcome(maintenance.completion()) == "completed"
```

The allowed DB fixture exposes native blocking/completion events. Choose an operation the active lease must block under this dependency's documented semantics, such as a database upgrade; do not assume every shared resource pins a connection. Before disposal, register blocking and completion observers. Bound the final outcome through a useful assertion or an external event indicating that release still blocks required work. Retain all acquired handles and pending requests in teardown even on failure; cancel or settle pending work without waiting forever on the leaked resource. A runner timeout alone gives a poor release diagnostic.

Faults: releasing another consumer's lease makes maintenance succeed too early; omitting final physical close prevents completion. Fresh-read persistence after disposal is a separate useful contract: maintenance that intentionally deletes data cannot prove disposal alone preserves rows. A registry being empty or an open count increasing does not prove release.

## 9. Property-based testing needs a useful oracle

For a real sorting API whose contract preserves records and orders by priority:

```text
for generated input in variedRecordLists:
  output = publicSort(input)
  assert output preserves every input record exactly once
  assert priorities(output) are nondecreasing

assert publicSort([
  {id: "b", priority: 2},
  {id: "a", priority: 1}
]) == [
  {id: "a", priority: 1},
  {id: "b", priority: 2}
]
```

A sortedness property alone accepts empty output. Preservation alone accepts unchanged unsorted input. Combine meaningful invariants with concrete examples. Add stability checks only when stable ordering is promised. Seed/replay failures using project's property-testing tools; don't introduce a new framework without need.

## 10. Async publication: helpers must not repair production

Contract: a batch publishes completed results only after every external update finishes.

Weak helper:

```text
async runHook():
  published = none
  await actualHook(onResult = value => published = value)
  return published
```

If production publishes a Promise instead of awaiting it, this helper adopts that Promise when returning it. `await runHook()` can still produce complete rows and hide early publication.

Stronger observation:

```text
firstReply = heldExternalReply()
secondReply = heldExternalReply()
publications = []
completion = actualHook(onResult = value => publications.append(value))

secondReply.complete({id: "second", title: "B"})
await secondReply.completionObserved()
assert publications == []

firstReply.complete({id: "first", title: "A"})
await completion
assert publications == [[{id: "first", title: "A"}, {id: "second", title: "B"}]]
```

Faults: publish an unresolved Promise, publish partial results, or return results in completion order when input order is promised. Callback only records observations; it does not await or replace owned behavior. Keep held replies and recorded values isolated per test and settle/clean them in teardown even if an assertion fails.

## 11. Recovery must cover the eligible population

Contract: refresh repairs every missing document association; cached and absent associations require no recovery request.

```text
seed local documents = [
  {id: "a", ownerId: "alice"},
  {id: "b", ownerId: "bob"},
  {id: "cached", ownerId: "carol"},
  {id: "unassigned", ownerId: null}
]
seed local owners = [{id: "carol", name: "Carol"}]
remote.expectRecovery(keys = unorderedSet("a", "b"), returns = [
  {id: "a", ownerId: "alice", owner: {id: "alice", name: "Alice"}},
  {id: "b", ownerId: "bob", owner: {id: "bob", name: "Bob"}}
])

rows = await actualDocumentService.listWithOwners()
assert projectedRows(rows) == [
  {id: "a", owner: {id: "alice", name: "Alice"}},
  {id: "b", owner: {id: "bob", name: "Bob"}},
  {id: "cached", owner: {id: "carol", name: "Carol"}},
  {id: "unassigned", owner: null}
]
remote.verifyExpectedRequests()
```

Faults: process first missing association only, omit a returned owner field, or refetch excluded rows. Compare membership without incidental order if API promises no ordering. Owned recovery/cache logic stays real; only remote service is simulated. Adapt which resource is fetched to actual API rather than assuming it requests documents.

## 12. A cloned caller does not prove shared payload safety

Contract: mutation keeps captured target while a supported data transform supplies stale identity; later mutation observers retain complete transport identity.

```text
seed external rows = [
  {id: "selected", title: "Before"},
  {id: "other", title: "Other"}
]
application.onBeforeMutation(transport => transport.patch.id = "other")
observed = []
application.onAfterMutation(event => observed.append(snapshot(event)))

result = await application.update(target = "selected", patch = {title: "After"})

assert result == {id: "selected", title: "After"}
assert await freshReader.read("selected") == {id: "selected", title: "After"}
assert await freshReader.read("other") == {id: "other", title: "Other"}
assert observed[0].target == "selected"
assert observed[0].transportPatch == {id: "other", title: "After"}
```

This example assumes the documented extension permits patch edits and promises observers that transport shape. Use actual contract; do not add hooks or demand these fields from a project lacking them. Callback is a supported consumer, not a replacement for owned normalization. If no such workflow exists but exported `omitIdentity` promises a non-mutating copy, test that utility directly with an independently fixed input and a frozen input. An application caller shielded by cloning can remain intact while the helper's argument or shared event is corrupted.

Faults: let stale patch identity override captured target; destructively remove identity from shared transport before later consumers see it. A valid narrower helper/adapter test can complement broader workflow; neither proves an unsupported extension.

## 13. Chronology must beat storage order

Contract: saved jobs are returned and replayed oldest first, regardless of identifier or insertion order.

```text
seed storage jobs in this insertion order = [
  {id: "a-new", createdAt: 20, target: "second"},
  {id: "z-old", createdAt: 10, target: "first"}
]

assert await actualJobs.listPending() == [
  {id: "z-old", createdAt: 10, target: "first"},
  {id: "a-new", createdAt: 20, target: "second"}
]
await actualJobs.replay()
assert remote.recordedTargets() == ["first", "second"]
assert await actualJobs.listPending() == []
```

Fault: remove chronological sorting. Both insertion order and alphabetical key order conflict with the required output. Only assert replay order when it is a promised contract. Boundary: real jobs/storage adapter and replay logic; external storage/remote service may be simulated with independent semantics.

## 14. Opt-out and expiry must preserve meaningful state

Contract: a consumer may skip advancing its checkpoint; future incremental reads still start from the saved checkpoint.

```text
seed external checkpoint = {position: 123}
remote.nextPage returns {rows: [{id: "new"}], position: 456}
await actualReader.refresh(advanceCheckpoint = false)
assert await checkpointStorage.read() == {position: 123}
await actualReader.refresh()
assert remote.recordedStartPositions() == [123, 123]
```

Fault: delete checkpoint on opt-out. Starting with no checkpoint cannot expose this loss. Expected checkpoint literal is independent of storage and response objects.

For a separate retention contract, seed one expired row and one still-valid row, invoke actual pruning with a supported time input, then assert the exact retained records. If retained state prevents stale reappearance, exercise that later consumer too. Fault: shorten retention so both rows expire. An expired-only fixture cannot reject it. No fake scheduling is needed for a public time argument; basic timer wiring should use native scheduling and bounded observations.

## 15. Prior approval does not authorize another resource

Contract: access is checked for each workspace, and provider errors deny effects while leaving valid use recoverable.

```text
policy permits client in "public", denies same client in "private"
assert await actualSession.join(client, "public") succeeds
assert await actualSession.join(client, "private") isDenied
assert privateObserver.received() == []
await publicPublisher.send("usable")
assert client.publicMessages() == ["usable"]
assert client.privateMessages() == []

policy.nextDecision throws ProviderUnavailable
assert await actualSession.join(otherClient, "public") reportsPolicyFailure
assert publicObserver.receivedFrom(otherClient) == []
policy.nextDecision permits
assert await actualSession.join(otherClient, "public") succeeds
```

Faults: reuse connection-wide approval for a different resource, or accept a thrown decision as approval. Adapt error shape and fail-open/fail-closed behavior to documented semantics. Boundary: real session routing and membership with an allowed external policy provider; observer records real delivery rather than replacing routing.

## 16. Closing while external work waits

Contract: ending a session prevents a pending admission from reviving it; a live replacement can use the freed slot.

```text
heldDecision = policy.holdNextDecision()
acquiredSession = actualSessions.open("first")
register independent cleanup for acquiredSession and heldDecision
pending = acquiredSession.join("shared")
await boundedObservation(heldDecision.entered)
assert publicCompletion(pending).settled == false
acquiredSession.close()
heldDecision.permit()
await boundedObservation(publicCompletion(pending).settled)
assert observer.receivedFrom("first") == []

replacement = actualSessions.open("replacement")
register independent cleanup for replacement before later assertions
assert await replacement.join("shared") succeeds
assert observer.presences() == ["replacement"]
```

Fault: remove ownership validation after awaited policy. When leave/re-entry is supported, similarly hold an edit from the old visit, leave/re-enter, release it, and verify only fresh edits reach the new visit. This is real owned lifecycle with a held external provider response, not a substituted session implementation.

Teardown records original handles before another acquisition or fallible observer runs. It releases held work and disposes known resources without indefinitely awaiting the broken operation, even if a discovery helper misses a resource. That fallback runs after the scored assertions and does not establish production cleanup correctness.
