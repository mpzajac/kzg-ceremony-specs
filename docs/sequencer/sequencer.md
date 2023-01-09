# Sequencer

The primary job of the sequencer is to sequence participants in the ceremony by acting as the point of contact for participants. The sequencer is required to have a DoS-resilient internet connection and sufficiently powerful hardware to verify the transcript rapidly so that they are able to send the transcript to the next participant as soon as possible. Furthermore, the sequencer should be a semi-trusted entity as they have the power to censor participants, although they cannot affect the safety of the setup.


## BatchTranscript object

### `Witness`

```python
@dataclass
class Witness:
    running_products: List[bls.G1Point]
    pot_pubkeys: List[bls.G2Point] = []
    bls_signatures: List[bls.G1Point] = []
```

### `Transcript`

```python
@dataclass
class Transcript:
    num_g1_powers: int
    num_g2_powers: int
    powers_of_tau: PowersOfTau  # as defined in ./docs/contribution/contribution.md
    witness: Witness
```

### `BatchTranscript`

```python
@dataclass
class BatchTranscript
    transcripts: List[Transcript] = []
    participant_ids: List[str] = []
    participant_ecdsa_signatures: List[bytes] = []
```

## Verification

Upon receiving the transcript back from a participant, the Sequencer MUST perform the following checks.

### Contribution schema structure:

- __Schema Check__ - Verify that the received `contribution.json` matches the `contributionSchema.json` schema.
```python
def schema_check(contribution_json: str, schema_path: str) -> bool:
    with open(schema_path) as schema_file:
        schema = json.load(schema_file)
        try:
            jsonschema.validate(contribution_json, schema)
            return True
        except Exception:
            pass
    return False
```


### Contribution parameter check

This check verifies that the number of $\mathbb{G}\_1$ and $\mathbb{G}\_2$ powers meets expectations.

```python
def parameter_check(batch_contribution: BatchContribution, batch_transcript: BatchTranscript) -> bool:
    for (contribution, transcript) in zip(batch_contribution.contributions, batch_transcript.transcripts):
        if contribution.num_g1_powers != transcript.num_g1_powers:
            return False
        if contribution.num_g2_powers != transcript.num_g2_powers:
            return False
        if contribution.num_g1_powers != len(contribution.powers_of_tau.g1_powers):
            return False
        if contribution.num_g2_powers != len(contribution.powers_of_tau.g2_powers):
            return False
    return True
```


### Point Checks

- Prime Subgroup checks
    - __G1 Powers Subgroup check__ - For each of the $\mathbb{G}\_1$ Powers of Tau (`g1_powers`), verify that they are actually elements of the prime-ordered subgroup.
    - __G2 Powers Subgroup check__ - For each of the $\mathbb{G}\_2$ Powers of Tau (`g2_powers`), verify that they are actually elements of the prime-ordered subgroup.
    - __PoTPubkey Subgroup checks__ - Check that the PoTPubkey is actually an element of the prime-ordered subgroup.

```python
def subgroup_checks(batch_contribution: BatchContribution) -> bool:
    for contribution in batch_contribution.contributions:
        if not all(bls.G1.is_in_prime_subgroup(P) for P in contribution.powers_of_tau.g1_powers):
            return False
        if not all(bls.G2.is_in_prime_subgroup(P) for P in contribution.powers_of_tau.g2_powers):
            return False
        if not bls.G2.is_in_prime_subgroup(contribution.pot_pubkey):
            return False
    return True
```

- __Non-zero check__ - Check that the `pot_pubkey`s are not equal to the point at infinity.
```python
def non_zero_check(batch_contribution: BatchContribution) -> bool:
    for contribution in batch_contribution.contributions:
        if bls.G2.is_inf(contribution.pot_pubkey):
            return False
    return True
```

### Pairing Checks

_Note:_ The following pairing checks SHOULD be batched via a random linear combination which would reduce the number of final exponentiations to 2 and decrease the number of Miller loops needed.

- __Tau update construction__ - Verify that the latest contribution is correctly built on top of the previous ones. Let $[\tau^1_{k}]\_1$ be the first power of tau from the most recent ( $k$ -th )`contribution` and $[\pi_{k - 1}] := [\tau^1_{k-1}]\_1$ be the transcript after updating from the previous correct `contribution`.  Verifying that $\tau$ was updated correctly can then be performed by checking 
$e([\pi_{k-1}]\_1, [pk_{k}]\_2) \stackrel{?}{=}e([\pi_{k}]\_1, g\_2)$, where $[pk_{2}]\_k$ is the public key of the $k$-th contribution.

```python
def tau_update_check(batch_contribution: BatchContribution, batch_transcript: BatchTranscript) -> bool:
    for (contribution, transcript) in zip(batch_contribution.contributions, batch_transcript.transcripts):
        tau = contribution.powers_of_tau.g1_powers[1]
        transcript_product = transcript.witness.running_products[1] #transcript.witness.running_products is a list, we probably want to take it's 2nd element
        pk = contribution.pot_pubkey #Check: contribution.pot_pubkey is only OPTIONAL
        if bls.pairing(transcript_product, pk) != bls.pairing(tau, bls.G2.g2):
                return False
    return True
```

- __Correct construction of G1 Powers__ - Verify that the $\mathbb{G}\_1$ points provided are indeed incremental powers of (the same) $\tau$. This check is done by asserting that the next $\mathbb{G}\_1$ point is the result of multiplying the previous one with $\tau$: $e([\tau^{i + 1}]\_1, g_2) \stackrel{?}{=}e([\tau^i]\_1, [\tau]\_2)$. Note that the check that the $\tau$ in $[\tau]\_2$ is the same as $\pi_k$ is done via the `g2_powers_check()` below which verifies that $e([\tau^i]\_1, g_2) \stackrel{?}{=}e(g_1, [\tau^i]\_2)$ and $[\tau^i]\_1 = [\pi_k]\_1$ due to the `Transcript` updates rules.

```python
def g1_powers_check(batch_contribution: BatchContribution) -> bool:
    for contribution in batch_contribution.contributions:
        powers = contribution.powers_of_tau.g1_powers
        pi = contribution.powers_of_tau.g2_powers[1]
        for power, next_power in zip(powers[:-1], powers[1:]):
            if bls.pairing(next_power, bls.G2.g2) != bls.pairing(power, pi):
                return False
    return True
```

- __Correct construction of G2 Powers__ - Verify that the $\mathbb{G}\_2$ points provided are indeed incremental powers of $\tau$ and that $\tau$ is the same across $\mathbb{G}\_1$ and $\mathbb{G}\_2$. This check is done by verifying that $\tau^i$ is the same across $[\tau^i]\_1$ and $[\tau^i]\_2$. $e([\tau^i]\_1, g_2) \stackrel{?}{=}e(g_1, [\tau^i]\_2)$

```python
def g2_powers_check(batch_contribution: BatchContribution) -> bool:
    for contribution in batch_contribution.contributions:
        g1_powers = contribution.powers_of_tau.g1_powers
        g2_powers = contribution.powers_of_tau.g2_powers
        for g1_power, g2_power in zip(g1_powers, g2_powers):
            if bls.pairing(bls.G1.g1, g2_power) != bls.pairing(g1_power, bls.G2.g2):
                return False
    return True
```

### Verifying a `Contribution`

The sequencer MUST verify all above checks:

```python
def verify_batch_contribution(batch_contribution: BatchContribution, identity: str) -> bool:
    for check in [schema_check, parameter_check, subgroup_checks, tau_update_check, g1_powers_check, g2_powers_check]:
        if not check(batch_contribution):
            return False
    return True
```

## Signature Verification & Pruning

The sequencer MUST verify and prune the signatures in the contributions.

- __BLS Signature Verification__ - Verify that the participant correctly signed their contributor identity. It is assumed that signature encoding errors return `False`.

```python
def bls_signature_check(batch_contribution: BatchContribution, identity: str) -> bool:
    for contribution in batch_contribution.contributions:
        pubkey = contribution.pot_pubkey
        message = identity
        signature = contribution.bls_signature
        if not bls.Verify(pubkey, message, signature):
            return False
    return True
```

- __ECDSA Signature Verification__ - Verify that the participant correctly signed their PoT Pubkeys. It is assumed that signature encoding errors return `False`.

```python
def ecdsa_signature_check(batch_contribution: BatchContribution, identity: str) -> bool:
    ecdsa_signature = batch_contribution.ecdsa_signature
    typed_data = contribution_to_typed_data_str(batch_contribution)
    message = encode_structured_data(typed_data)  # Encode Data as per EIP 712
    recovered_address = recover_message(message, contribution.ecdsa_signature)  # Address recovery
    return recovered_address == identity[4:]  # Strip off leading 'eth|' padding from id & verify
```

### Signature Pruning

The sequencer MUST prune invalid signatures from the contribution. If any of the BLS signatures are invalid, they must all be set to `''` and if the ECDSA signature is invalid, it must also be set to `''`.

```python
def prune_invalid_signatures(batch_contribution: BatchContribution, identity: str) -> BatchContribution:
    if not all([bls_signature_check(contribution, identity) for contribution in batch_contribution]):
        for contribution in batch_contribution:
            contribution.bls_signature = ''
    if identity[:4] != 'eth|' or not ecdsa_signature_check(batch_contribution, identity):
        batch_contribution.bls_signature = ''
    return batch_contribution
```

## Updating the transcript

Once the sequencer has performed the above verification checks and assuming they all passed, and the signatures have been pruned as above, they MUST update the transcript. Thy do so by adding the `batch_contribution` to the `batch_transcript`. The `identity` is a string that represents the user. It is generated by taking the Ethereum address or GitHub account from the users' login session and feeding them into `eth_address_to_identity` or `github_handle_to_identity` respectively.

```python
def update_transcript(
        old_batch_transcript: BatchTranscript,
        batch_contribution: BatchContribution,
        identity: str) -> BatchTranscript:
    if not verify_batch_contribution(batch_contribution):
        return old_batch_transcript
    batch_contribution = prune_invalid_signatures(batch_contribution)
    new_batch_transcript = BatchTranscript()
    for (transcript, contribution) in zip(old_batch_transcript.transcripts, batch_contribution.contributions):
        new_transcript = Transcript(
            num_g1_powers=transcript.num_g1_powers,
            num_g2_powers=transcript.num_g2_powers,
            powers_of_tau=contribution.powers_of_tau,
            witness=Witness(
                running_products=transcript.witness.running_products + [contribution.powers_of_tau.g1_powers[1]],
                pot_pubkeys=transcript.witness.pot_pubkeys + [contribution.pot_pubkey],
                bls_signatures=transcript.witness.bls_signatures + [contribution.bls_signature]
            )
        )
        new_batch_transcript.transcripts.append(new_transcript)
    new_batch_transcript.participant_ids = old_batch_transcript.participant_ids + [identity]
    new_batch_transcript.participant_ecdsa_signatures =\
        old_batch_transcript.participant_ecdsa_signatures + [batch_contribution.ecdsa_signature]
    return new_batch_transcript
```

## Generating the contribution file for the next participant

Once the transcript has been updated, the sequencer MUST get a new `Contribution` file to send to the next participant.

```python
def get_new_contribution_file(batch_transcript: BatchTranscript) -> BatchContribution:
    batch_contribution = BatchContribution()
    for transcript in batch_transcript.transcripts:
        batch_contribution.contributions.append(
            Contribution(
                num_g1_powers=transcript.num_g1_powers,
                num_g2_powers=transcript.num_g2_powers,
                powers_of_tau=transcript.powers_of_tau,
            )
        )
```

