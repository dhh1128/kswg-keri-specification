# Proposal: an AID-committed hybrid classical and post-quantum signing trait, with a post-quantum-only exit

Draft for discussion, November 2026. Nothing here is adopted; the trait codes are placeholders. The trait is algorithm-independent and depends on a companion change to the CESR specification that gives every verification key code a normative class.

## Motivation

KERI lets a controller choose signing and pre-rotation thresholds for each establishment event. A verifier may apply a stricter local acceptance rule, such as requiring that no threshold be satisfiable by keys whose security rests on a single class of hardness assumption. That rule is absent from the KEL. Validators with different local rules can then accept different rotations for the same AID, and neither holds two valid events that would count as duplicity evidence. Each validator's view is internally consistent, so the divergence is silent.

A configuration trait in the inception event fixes this. The trait is part of the inception that derives a self-addressing AID, so the controller commits to the rule before any key is exposed, and every validator implementing the extension enforces the same rule against the same events. The rule becomes a property of the AID.

The rule must also allow the AID to leave hybrid mode. A controller that started with classical and post-quantum keys will eventually want to drop the classical keys, either because they no longer add assurance or because their algorithm is retired. Abandoning the AID to do that would discard its accumulated reputation and every reference to it. Rotation is the existing operation that changes key state while preserving the AID, so the exit is a rotation, and the KEL records it. [KERI specification, Configuration traits field, Pre-rotation, and Rotation](https://github.com/trustoverip/kswg-keri-specification/blob/740412d/spec/spec-body.md).

The trait states a protocol property in terms of key classes. It names no algorithm. Which classical and post-quantum algorithms a given community accepts is a separate question, answered by deployment profiles (see [Deployment profiles](#deployment-profiles)).

## Terms

*Key class.* A normative attribute of a CESR verification key code, with one of two values:

- `C`, *classical*: the code's signature algorithm relies on a hardness assumption that a cryptographically relevant quantum computer is known to break in polynomial time, such as integer factorization or a discrete logarithm over a finite field or an elliptic curve group.
- `Q`, *post-quantum*: the code's signature algorithm was designed to resist known quantum attacks, and the CESR specification designates it so.

A CESR code that is not a verification key code has no key class.

*Class of a key.* The key class of the CESR code that qualifies the key. A validator reads it from the code table. A validator never assigns it.

*Class-safe threshold.* Defined in [Class-safe thresholds](#class-safe-thresholds).

*Hybrid state, post-quantum-only state.* The two states of an AID that carries `HYB`, defined in [State](#state).

## Companion change to the CESR specification (normative)

The CESR master code table gains a column, *Key class*, with the values `C`, `Q`, or empty. Every verification public key code in the table, whether transferable or non-transferable, and whether listed as "verification" or as "verification or encryption", MUST carry `C` or `Q`. Every other code MUST leave the column empty. Every key code added in the future MUST declare its class in the same change that assigns the code.

A code's class MUST NOT change, within a genus and version of the code tables or in any later version. If an algorithm's class assumption fails, the remedy is to deprecate the code and assign a new one, and the deprecated code keeps its class. Reclassifying a code would change the validity of events already accepted. All codes that qualify keys of the same signature algorithm MUST carry the same class.

Indexed signature codes carry no class. A signature's class is the class of the public key it verifies against, which KERI's indexed or positional mapping identifies.

Rationale: if each validator decided which algorithms count as post-quantum, two validators could classify the same key differently and accept different rotations, which is the divergence this trait exists to remove. The class must therefore be fixed by the specification that defines the code. The code table is where a validator already looks up everything else about a primitive, so the class belongs there as one more column.

This change is a prerequisite for `HYB`. It also needs coordination between the two specifications: a KERI version that defines `HYB` must name the CESR table version whose classes it relies on.

## Proposed configuration traits (normative)

Add the following rows to the configuration traits table. Both code strings are proposed and unassigned.

| Trait | Title | Placement | Description |
|:---:|:---|:---|:---|
| `HYB` | Hybrid classical and post-quantum | `icp` or `dip` only | Starts the hybrid state for this AID. Every current signing threshold and every prior-next threshold MUST be unsatisfiable by `C`-class keys alone and by `Q`-class keys alone. |
| `PQO` | Post-quantum-only exit | `rot` or `drt` only | Moves an AID carrying `HYB` to the post-quantum-only state, in which only `Q`-class keys hold authority. |

### State

1. `HYB` MUST appear at most once in a KEL, and only in the inception of a transferable, self-addressing AID. The inception MUST have a nonempty `n`.
2. The hybrid state begins at inception and continues until a valid `PQO` rotation.
3. `PQO` MUST appear at most once in a KEL, and only in a `rot` or `drt` of an AID whose inception contains `HYB`.
4. The post-quantum-only state begins with the `PQO` event and is permanent.
5. A `c` list that contains both codes, or a duplicate of either code, is invalid.
6. Once `HYB` appears in the inception, the hybrid or post-quantum-only state governs every later event, whatever later `c` lists contain. A later `c` list that omits `HYB` does not end the state.
7. The state is a property of each branch of a KEL. When competing establishment events exist at one sequence number, each is validated against the events that precede it on its own branch, under KERI's normal duplicity and recovery rules.
8. A `dip` carrying `HYB` is held to these rules when it is validated. The state takes effect when the delegated inception does, under KERI's normal delegation rules.
9. A validator MUST invalidate any event that violates these placement or transition rules.
10. This extension requires a KERI version, or a mandatory-understanding rule, under which a validator that does not implement `HYB` and `PQO` rejects an event carrying either. Until such a boundary exists, the trait's guarantee holds only among validators that implement it. The exact boundary is an open question (see [item 4](#open-questions-and-assumptions)).

### Classifying keys and next digests

1. A validator MUST determine the class of each public key in a controller `k` list from its CESR code and the Key class column of the CESR code table. Because a code's class never changes (see [Companion change](#companion-change-to-the-cesr-specification-normative)), every table version that contains the code gives the same answer. A validator MUST reject an event whose `k` list, while `HYB` or `PQO` governs it, contains a key whose code is unsupported or whose class is undetermined.
2. Each `k` list MUST contain distinct public keys, compared by raw key material regardless of code. Each `n` list MUST contain distinct digest values.
3. Thresholds are evaluated over sets of distinct indices. Each index in `k` counts at most once toward `kt`, and each prior `n` index counts at most once toward the prior `nt`, however many signatures the event attaches for it. Each public key contributes at most once to each threshold evaluation.
4. A signature's class is the class of the public key it verifies against. The verified signature MUST identify its public key in `k` and, for rotation authority, one matching prior `n` entry, using KERI's indexed or positional mapping, including the prior-next index of a dual-indexed signature. Each revealed key counted for prior-next authority MUST match exactly one prior `n` entry, and each prior `n` entry MUST be matched by at most one revealed key. An event in which a revealed key matches more than one prior `n` entry, for example under two different digest codes, is invalid. The prior `n` index fixed by this match supplies both the weight in the prior `nt` and the slot class for the reveal-time check.
5. A digest's CESR code identifies the digest algorithm and reveals nothing about the class of the hidden key. A validator MUST classify a prior `n` entry only when a later `rot` or `drt` reveals a public key whose digest matches that entry under its committed digest code.
6. A slot's class is its position in the `n` list of the immediately preceding establishment event. A digest that a controller recommits from an older `n` takes the class of its new position, and the reveal-time check uses that position. Its position in any older event plays no part. Controller tooling that carries unrevealed reserve digests forward SHOULD place each in a slot of the class it held before, reordering them if necessary, since a carried digest placed in a slot of the other class can never be revealed. [KERI specification, Key list, Next key digest list, and Indexed Signatures](https://github.com/trustoverip/kswg-keri-specification/blob/740412d/spec/spec-body.md).

### Class-safe thresholds

For a threshold `T` over an indexed list, let `T(S)` mean that signatures at the indices in `S` satisfy `T` under KERI's integer or fractionally weighted threshold rules. Let `C-all` be every index assigned class `C`, and `Q-all` every index assigned class `Q`. `T` is *class-safe* when all three of the following hold:

1. `T(C-all)` is false.
2. `T(Q-all)` is false.
3. `T(C-all ∪ Q-all)` is true.

Under KERI's rules a fractionally weighted threshold is a list of clauses that are all required, and a clause is satisfied when the weights of its verified signers sum to at least one. Weights are non-negative rational fractions, so zero is a permitted weight. [KERI specification, Fractionally weighted threshold](https://github.com/trustoverip/kswg-keri-specification/blob/740412d/spec/spec-body.md). KERI thresholds are therefore monotone: adding verified signatures never turns a satisfied threshold into an unsatisfied one. These three evaluations therefore cover every single-class subset and confirm that some full set satisfies `T`. A validator MUST evaluate the complete threshold expression, including every weighted clause and nesting level. Counting keys, or comparing the number of keys of each class with an integer threshold, is insufficient.

The current `kt` test uses classes read from the CESR codes in `k`. The next `nt` test uses classes committed by position in `n` (see [Inception and hybrid rotation](#inception-and-hybrid-rotation)). A validator MUST apply the prior-next test to signatures matched to the prior `n` list separately from the current signing threshold test, so a key newly added in `k` cannot supply missing prior-next authority of either class.

The test is independent of algorithm. A `k` list may hold several classical algorithms and several post-quantum algorithms at once. Only class membership enters the test.

### Inception and hybrid rotation

1. An `icp` or `dip` containing `HYB` MUST have a class-safe `kt` over `k`, and its establishment signatures MUST satisfy `kt`. Every key in `k` MUST have class `C` or `Q`.
2. Its `n` list MUST have an even number of entries, at least two. Positions 0, 2, 4, and so on are committed `C` slots. Positions 1, 3, 5, and so on are committed `Q` slots. Its `nt` MUST be class-safe over these slot assignments.
3. Every `rot` or `drt` in the hybrid state that does not carry `PQO` MUST pass all ordinary KERI rotation and delegated-rotation checks, and in addition:
   1. Every key in its `k` list MUST have class `C` or `Q`, its `kt` MUST be class-safe, and its verified current signatures MUST satisfy `kt`.
   2. Each signature counted for prior-next authority MUST match a digest in the previous establishment event's `n`. The revealed public key's class MUST equal the class of the slot that digest occupies. The digest-matched signatures MUST satisfy the prior `nt`, which was class-safe when committed, so they include both classes even in a partial rotation that leaves reserve entries unrevealed.
   3. A new nonempty `n` MUST follow the alternating-slot and class-safe `nt` rules.
   4. A rotation with `n=[]` and `nt=0` MAY abandon the AID under the ordinary KERI rule. It MUST still satisfy rules 3.1 and 3.2.
4. Interaction events in the hybrid state are governed by the current class-safe `kt`, and their verified controller signatures MUST satisfy it. `EO`, if present, forbids them as usual.

The slot position supplies the class claim while the key is hidden. The actual class stays unknown until reveal, when the validator checks it against the slot. A controller that commits a key under the wrong slot may make its AID unrotatable (see [item 3](#open-questions-and-assumptions)).

### Post-quantum-only exit

1. The exit is one `rot` or `drt` carrying `PQO` in `c`. It MUST first pass every ordinary KERI `rot` or `drt` requirement, including prior-event chaining, sequence number, threshold well-formedness, backer changes, and for a `drt` the delegator's anchoring under the normal delegation and escrow rules.
2. Before the state changes, the event MUST pass the prior-next test of rule 3.2 above against the last hybrid establishment event: verified, digest-matched signatures whose revealed keys match their committed `C` and `Q` slots, satisfying the class-safe prior `nt`.
3. The new signing authority MUST belong to `Q`-class keys only. The new `kt` MUST give a weight of zero to every occurrence of every `C`-class key, in every clause and at every nesting level, and it MUST be satisfiable by the event's `Q`-class keys. The event's verified current signatures from `Q`-class keys alone MUST satisfy the new `kt`. Because rule 4 places at least one `C`-class key in `k`, the exit `kt` is necessarily a fractionally weighted threshold.
4. Every `C`-class key in the exit event's `k` MUST match a `C` slot in the prior `n`, MUST have a verified controller signature mapped to that slot, and MUST carry zero current signing weight.
5. The exit event's new `n` MUST be nonempty, and its `nt` MUST be satisfiable under ordinary KERI rules.
6. The alternating-slot rule ends at this event. From here on the reveal-time class check is the only enforcement on next keys, since a hidden digest's class cannot be verified. Controller tooling SHOULD refuse to commit the digest of a `C`-class key after the exit, because revealing it later would be invalid and could leave the AID unrotatable.

### Post-quantum-only state

1. Every subsequent `rot` and `drt` MUST reveal only `Q`-class keys, MUST place only `Q`-class keys in `k`, and MUST use only `Q`-class signatures for current and prior-next authority.
2. Every revealed next key remains subject to the reveal-time class check, now against class `Q`.
3. Keys of different post-quantum algorithms MAY coexist in `k` and in `n`, and a rotation MAY move from one post-quantum algorithm to another.
4. A later `rot` or `drt` MAY set `n=[]` and `nt=0` to abandon the AID.
5. Interaction events use the post-quantum-only current threshold.
6. A second `HYB` or `PQO` is invalid.

### Why the exit cannot be forced by one class

The exit rotation needs the precommitted next keys of both classes, because rule 2 of the exit applies the class-safe prior `nt` before the state changes. A party that holds only `Q`-class keys, or only `C`-class keys, cannot satisfy that threshold. Supplying new keys of the missing class in the event does not help: the prior digest match excludes keys that were not committed, and the prior `nt` counts only matched signatures. Once the exit is valid, the move to post-quantum-only control was authorized by both classes and is bound to the same AID by the KEL chain. A `PQO` rotation with an empty `n` would abandon the AID, so rule 5 of the exit makes it invalid as an exit. [KERI specification, Rotation and Next key digest list](https://github.com/trustoverip/kswg-keri-specification/blob/740412d/spec/spec-body.md).

### Witnesses, receipts, and other traits

`HYB` and `PQO` constrain the controller AID's key state and controller signatures, and nothing more. An AID in the post-quantum-only state may still depend on a delegator or on witnesses whose keys are classical. Such parties can delay or withhold support for its events, and cannot forge its controller signatures. A community that wants post-quantum operation end to end has to require it of delegators and witnesses separately, for example in a deployment profile. They leave the witness threshold `bt`, the witness list `b`, witness signatures, the receipt format, and registrar backer endorsements unchanged. A witness receipt on a hybrid event attests to that event under the witness's own AID and key rules, and acquires no two-class requirement. A validator MUST validate the controller event against its active trait before treating receipts as support for it. A receipt cannot cure an invalid controller event.

`EO` keeps its ban on interaction events. `DND` keeps its ban on delegation by the AID. A `dip` or `drt` carrying `HYB` or `PQO` is checked against the delegatee's own key state, and the delegator's anchoring event is checked under the delegator's own traits and the normal delegation rules. `DID` leaves each AID's key state separate, so it does not carry the hybrid state from one AID to another. `RB` and `NRB` keep their backer interpretation and precedence. `HYB` and `PQO` form a one-way transition, so the table's rule that the later of two conflicting traits in one list wins does not apply to them. [KERI specification, Configuration traits field](https://github.com/trustoverip/kswg-keri-specification/blob/740412d/spec/spec-body.md).

## Deployment profiles

This section is informative, except for the sentence marked normative.

`HYB` guarantees one protocol property: no current or prior-next threshold is satisfiable by one class of keys. It does not say which classical or post-quantum algorithms are good enough, which security levels should be paired, or whether a community accepts a particular post-quantum family at all. Those choices vary between communities and change faster than a protocol specification should.

Normative: a deployment profile MAY restrict the key codes of each class that controllers use and that validators accept within that profile, and such a restriction MUST NOT relax any `HYB` or `PQO` rule.

A profile narrows which algorithms satisfy the trait. It cannot widen the trait. For example, one profile might require a specific elliptic curve at a stated security level for its `C` keys and a specific lattice-based signature at a matching level for its `Q` keys, while another might accept any `Q`-class code in the table.

## Worked threshold sketch (informative)

`C0`, `C1` denote keys with class-`C` codes and `Q0`, `Q1`, `Q2`, `Q2'` keys with class-`Q` codes. The algorithms are irrelevant to the rules. `H(x)` is the existing qualified digest operation over a qualified public key.

1. Inception: `k=[C0,Q0]`, `kt=2`, `n=[H(C1),H(Q1)]`, `nt=2`, `c=["HYB"]`. `kt` is class-safe: `{C0}` and `{Q0}` each fail, both together succeed. `n[0]` is a `C` slot and `n[1]` a `Q` slot, and `nt=2` is class-safe over them.
2. Hybrid rotation: reveals `C1,Q1`, checks each digest and each revealed class against its slot, and needs both signatures for the prior `nt`. Its new `k`, `kt`, `n` and `nt` follow the same rules.
3. Unsafe threshold, rejected: `k=[C0,C1,Q0]`, `kt=2`. `T({C0,C1})` is true, so `kt` is not class-safe.
4. Exit, from the commitment in step 1: reveals `C1,Q1` and verifies both against the prior `nt`. Sets `k=[C1,Q1]` with the single-clause weighted threshold `kt=["0","1"]` (weight 0 for `C1`, weight 1 for `Q1`), commits `n=[H(Q2),H(Q2')]` with `nt=1`, and places `PQO` in `c`. `C1` supplies prior-next authority and has zero weight in the new state. The KERI specification permits zero weight for a revealed rotation key, and keripy's `Tholder` accepts a `"0"` weight. [KERI specification, Fractionally weighted threshold and Partial Rotation](https://github.com/trustoverip/kswg-keri-specification/blob/740412d/spec/spec-body.md); [keripy Tholder](https://github.com/WebOfTrust/keripy/blob/0f1966f88/src/keri/core/coring.py).
5. After the exit, `Q2` and `Q2'` may use different post-quantum algorithms. The next rotation reveals either, checks that its code has class `Q`, and uses post-quantum-only authority.

## Open questions and assumptions

1. The class is a judgment made once, by spec editors. Generalizing moves the question "is this algorithm post-quantum?" from each validator into the CESR table. That gives every validator the same answer, which is the point, but it makes the CESR editors responsible for a designation that cryptanalysis can overturn. Several post-quantum candidates have been broken late in evaluation. The proposal answers a broken designation with deprecation and a new code, because reclassifying in place would change the validity of accepted events. Existing KELs that rely on the broken code keep their validity under the trait until the controller rotates away. The specifications need a stated deprecation path for that case.
2. The trait guarantees less than a named-algorithm trait would. A trait that named its algorithms could guarantee a specific pair. This one guarantees only that each threshold needs both classes. A KEL can satisfy `HYB` with a weak classical key and a low-level post-quantum key, and two validators applying different deployment profiles can still disagree about whether to accept it. The trait removes the divergence about the class property. It leaves algorithm acceptance to profiles, which are verifier-local again. Whether a profile identifier should also be committed in-band is open.
3. False class claims behind hidden digests. Hidden `n` entries prove nothing about the class of the key behind them. A controller can put a `C`-class key's digest in a `Q` slot and create an unrotatable AID. Controller tooling SHOULD classify next keys, check their slots and test `nt` before committing. The validator rule prevents single-class acceptance at reveal. Future liveness still depends on the controller's hidden keys and their custody.
4. Old validators. Existing validators may ignore unknown configuration strings. Conformance needs an adopted KERI specification, the CESR Key class column, and a version or mandatory-understanding boundary that makes an unsupported `HYB` or `PQO` fail closed. The trait cannot change old validators' behavior. Code stability matters because a self-addressing inception commits the key codes' bytes.
5. Two-specification coupling. `HYB` depends on a column in the CESR tables. The two specifications version separately, so a KERI version that defines `HYB` has to name the CESR table version it relies on, and every new key code has to declare a class before any `HYB` AID can use it. A validator that implements an older table rejects a key under a newer code, as it would for any unknown code, so moving a live `HYB` AID to a newly assigned post-quantum algorithm waits for validators to adopt the table that contains it. The cost also lands on every CESR consumer, since the column is added to a table that many protocols share.
6. Composite keys. Some proposals define one key code that combines a classical and a post-quantum algorithm. Such a code fits neither class. This proposal gives it no class, so a `HYB` or `PQO` `k` list rejects it. A third class value, and its threshold semantics, could be defined later.
7. Two post-quantum algorithms are not hybrid. A threshold that needs two different post-quantum algorithms and no classical key fails `HYB`, because the trait is defined against the classical and post-quantum split. A trait for diversity among post-quantum algorithms would be a separate proposal.
8. Cost of interaction events. Every interaction event in the hybrid state carries a classical and a post-quantum signature, and post-quantum signatures run to kilobytes. `EO` avoids the cost by forbidding interaction events. This proposal gives no size analysis.
9. The final classical signature. The exit needs one last signature from a precommitted `C`-class key. A controller that has lost those keys cannot exit, and cannot abandon the AID either, since abandonment in the hybrid state also needs both classes. The proposal accepts this so that holding one class alone cannot force the transition. Whether every application wants that requirement is open. A related liveness risk comes from code lifecycle: if the precommitted classical key's code is deprecated, or a deployment profile stops accepting it, before the controller exits, validators that refuse the code cannot accept the exit that retires it. The specifications would need to distinguish a code that is invalid from one that is no longer accepted for new commitments, and keep precommitted reveals of the second kind verifiable.
10. Positional slots. The alternating slots constrain the order and count of `n`. They preserve reserve and partial rotation whenever revealed keys from both classes can satisfy the class-safe `nt`. This positional convention is a new protocol rule. A typed commitment would carry the same claim with different serialization, and may be preferable if a CESR code can express it.
11. Witness AIDs. The KERI specification text reviewed here requires witness AIDs to be non-transferable qualified public keys. A separate proposal for self-addressing digest witness AIDs must settle that before such witnesses are assumed to work with this trait. The controller trait does not prescribe a witness algorithm or require hybrid witness receipts.
12. Code strings. `HYB` and `PQO` are provisional.

## Source notes

The KERI mechanics used here come from the [KERI specification at `740412d`](https://github.com/trustoverip/kswg-keri-specification/blob/740412d/spec/spec-body.md): configuration traits, key and next-digest thresholds, the next digest list, rotation, delegated rotation, and witnesses and backers. The CESR code table structure, including the master table for genus and version `-_AAACAA` and its "verification or encryption key" rows, comes from the [CESR specification at `bad6edd`](https://github.com/trustoverip/kswg-cesr-specification/blob/bad6edd/spec/spec-body.md); the Key class column is new.
