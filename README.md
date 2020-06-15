# Introduction

Bitcoin transactions transfer funds by consuming unspent outputs of
previous transactions as inputs to create new outputs. The protocol
rules enforced by the network ensure that transactions do not
arbitrarily inflate the money supply and that outputs are spent at most
once. While some newer cryptocurrencies use more sophisticated
approaches to define such rules, in Bitcoin the amounts as well as the
specific outputs being spent are broadcast in the clear as part of the
transaction. This presents significant challenges to transacting
privately\[1\] as shown already in some of the earliest academic studies
of Bitcoin \[@reid2013analysis; @ron2013quantitative;
@androulaki2013evaluating; @ober2013structure; @moeser2013inquiry;
@meiklejohn2013fistful\].

The conditions for spending an output are specified in its
`scriptPubKey`, typically requiring that the spending transaction be
signed by a specific key. The signatures authorizing a transaction
usually commit to the transaction in its entirety, making it possible
for mutually distrusting parties to jointly create transactions without
risking misallocation of funds: participants will only sign a proposed
transaction after confirming that their desired outputs are included and
the transaction is only valid when all parties have signed.

Chaumian CoinJoin \[@mizrahi2013blind; @maxwell2013coinjoin; @zerolink\]
is a privacy enhancing technique that uses this atomicity property and
Chaumian blind signatures \[@chaum1983blind\] to construct collaborative
Bitcoin transactions, also known as CoinJoins. Participants connect to a
server, known as the coordinator, and submit their inputs and outputs
using different anonymity network identities. That alone would provide
anonymity but since outputs are unconstrained it’s not robust against
malicious users who may disrupt the protocol by claiming more than their
fair share. To mitigate this the coordinator provides blind signatures
representing units of standard denominations in response to submitted
inputs. By unblinding and presenting this valid signature, the
coordinator is unable to link the signed output to specific inputs but
can be still verify that an output registration is authorized.

The use of standard denominations in the resulting CoinJoin transaction
obscures the relationship between individual inputs and outputs, making
the origins of each output ambiguous. Unfortunately standard
denominations limit the use of privacy-enhanced outputs for payments of
arbitrary amounts and result in a change output which maintains a link
to the non-private input.

## Limitations of Wasabi

In this work, we aim to improve on ZeroLink \[@zerolink\] as implemented
by Wasabi, the most popular Chaumian CoinJoin implementation for
Bitcoin. We identify several privacy shortcomings and inefficiencies of
Wasabi CoinJoins. Some metrics comparing Wasabi, Samourai and other
apparent CoinJoin transactions are provided. The “Other” category
includes JoinMarket, but also has an inherent false positive error given
these transactions are identified heuristically.

### Denominations

Due to the nature of blind signatures, mixed outputs of Wasabi CoinJoins
are restricted to fixed set of multiples of a base denomination\[2\].
This creates friction when sending or receiving arbitrary amounts of
Bitcoin, as using fixed denomination generally creates change, both when
mixing and when spending mixed outputs.

We define *CoinJoin inefficiency* as the fraction of non-mixed change
outputs in a CoinJoin transaction, see
[\[fig:cjinefficiency\]](#fig:cjinefficiency).

![CoinJoin inefficiency of various privacy-focused Bitcoin
wallets.<span label="fig:cjinefficiency"></span>](Figures/CJInefficiency.pdf)

### Minimum denomination

In order to participate, a user’s combined input amount must be greater
or equal to the base denomination.\[3\] We observe, that considerable
portion of CoinJoin inputs are less than this minimum denomination, see
[\[fig:minimumdenomination\]](#fig:minimumdenomination).

![Fraction of inputs with value smaller than the minimum denomination in
Wasabi CoinJoin
transactions.<span label="fig:minimumdenomination"></span>](Figures/SmallValueInputsWasabi.pdf)

Even when users are able to provide several smaller value inputs with
total value greater than the minimum denomination, the coordinator knows
those inputs belong to the same user. In an ideal mixing protocol the
coordinator should not obtain more information than the already
available public ledger data by coordinating the CoinJoin transaction.
This information removes many degrees of freedom when assigning
non-derived sub-transactions \[@maurer2017anonymous\], potentially
removing ambiguity when there are multiple valid assignments or reducing
the computational cost of such an analysis.

Furthermore if users consolidate coins before the CoinJoin in an
additional transaction in order to be able to participate in a CoinJoin,
then this link is revealed publicly based on the common input ownership
heuristic \[@meiklejohn2013fistful\].

### Variable denominations

Since users pay mining and coordination fees the denominations are
gradually reduced between rounds of consecutive CoinJoins in order to
make it possible for users to mix several times without providing
additional inputs. This introduces a perverse incentive to minimize
coordination fees by remixing in quick succession in order, resulting in
a smaller anonymity set than with time-staggered remixes.

### Block-space efficiency

The rigidity of the current transaction structure, i.e. fixed
denominations, constrains users’ unspent transaction output set
structure as well. These limitations force users to consolidate their
coins (see [\[fig:postmixmerging\]](#fig:postmixmerging)) and create
additional intermediate outputs with constrained amounts when
interspersing CoinJoin transactions with transactions that send or
receive value.

![Average number of inputs from the first post-mix transactions in
various CoinJoin
schemes.<span label="fig:postmixmerging"></span>](Figures/postMixInputMerging.pdf)

### Lack of privacy-enhanced payments

Currently Wasabi supports neither payments from a CoinJoin, nor payments
in a CoinJoin. Payments from a CoinJoin would protect sender privacy and
improve efficiency by requiring fewer intermediate outputs. Payments
within a CoinJoin would protect both sender and receiver privacy, and
since they are a form of PayJoin\[4\] it would also improve privacy by
introducing degrees of freedom in the interpretation of CoinJoins.

## Our Contribution

We present WabiSabi, a generalization of Chaumian CoinJoin based on a
keyed-verification anonymous credentials (KVAC)
scheme \[@chase2019signal\]. The use of KVACs replaces blind
signatures’ standard denominations with homomorphic amount
commitments, similar to Confidential
Transactions \[@maxwell2016confidential\], where the sum of any
participant’s outputs does not exceed that of their inputs while hiding
the underlying values from the coordinator. In addition to being more
flexible this improves privacy compared to blind signatures and standard
denominations, since smaller inputs can be combined and change outputs
created with the same unlinkability guarantees as the privacy enhanced
outputs\[5\].

WabiSabi can be instantiated to construct a variety of CoinJoin
transaction structures that depart from the standard output denomination
convention, used by SharedCoin\[6\] and CashFusion\[7\] style
transactions and Knapsack \[@maurer2017anonymous\] mixing. Payments from
CoinJoin transactions are possible, as are payments within them,
effectively a multiparty PayJoin that trades the steganographic
properties for improved privacy from counterparties. Additionally,
restrictions on consolidation of inputs can be removed, and there are
opportunities for reducing unmixed change and relaxing minimum required
denominations, and improved block space efficiency.

# Preliminaries

Hereby we give an informal and high-level description of applied
cryptographic primitives. In the following the security parameter is
denoted as
<img src="svgs/fd8be73b54f5436a5cd2e73ba9b6bfa9.svg?invert_in_darkmode" align=middle width=9.58908224999999pt height=22.831056599999986pt/>.

## Commitment schemes

A commitment scheme allows one to commit to a chosen message while
preventing them from changing the message after publishing the
commitment. Secure commitments do not reveal anything about the chosen
message.

<img src="svgs/44243f3482fd48a1854fc659be308b17.svg?invert_in_darkmode" align=middle width=132.2858163pt height=24.65753399999998pt/>.
The
<img src="svgs/ccd2318b5f0daaf35a3bf54a220412e2.svg?invert_in_darkmode" align=middle width=54.703370699999994pt height=22.831056599999986pt/>
algorithm generates a commitment
<img src="svgs/db5f7b3e9934fbc5a2859d88c4ba84a3.svg?invert_in_darkmode" align=middle width=9.614228249999991pt height=22.465723500000017pt/>
to message
<img src="svgs/0e51a2dede42189d77627c4d742822c3.svg?invert_in_darkmode" align=middle width=14.433101099999991pt height=14.15524440000002pt/>
using randomness
<img src="svgs/89f2e0d2d24bcf44db73aab8fc03252c.svg?invert_in_darkmode" align=middle width=7.87295519999999pt height=14.15524440000002pt/>.

<img src="svgs/7f683f2b1cc9a349b20f17685cab1fb7.svg?invert_in_darkmode" align=middle width=249.90627915pt height=24.65753399999998pt/>:
one can verify the correctness of the opening of a commitment by
checking
<img src="svgs/1bd25a7239caa24026e82c490dcd2185.svg?invert_in_darkmode" align=middle width=128.63250344999997pt height=34.618666499999996pt/>.
If equality holds the algorithm outputs
<img src="svgs/5f09ecc633d7144c6043f0c32810a063.svg?invert_in_darkmode" align=middle width=35.05232279999999pt height=22.465723500000017pt/>,
otherwise
<img src="svgs/75e773f41015f3dc7611a20a8ab1d44d.svg?invert_in_darkmode" align=middle width=37.59112334999999pt height=22.831056599999986pt/>.

For ease of understanding one may assume in the following that the
commitment scheme is instantiated as a Pedersen commitment.

## MAC

A message authentication code (MAC) ensures the integrity of a message
and consists of the following three probabilistic polynomial-time
algorithms.

<img src="svgs/efb0326f1ccb95572c54271d04bcf6e9.svg?invert_in_darkmode" align=middle width=150.9594009pt height=24.65753399999998pt/>.
a party generates a secret key
<img src="svgs/ec468df79768509d15af76fd10a0ac47.svg?invert_in_darkmode" align=middle width=14.337957149999992pt height=22.831056599999986pt/>
for MAC generation and
verification.

<img src="svgs/99a73c765111182aa79b3f9db4b25892.svg?invert_in_darkmode" align=middle width=105.59567039999997pt height=24.65753399999998pt/>.
one can generate a MAC
<img src="svgs/4f4f4e395762a3af4575de74c019ebb5.svg?invert_in_darkmode" align=middle width=5.936097749999991pt height=20.221802699999984pt/>
on a message
<img src="svgs/0e51a2dede42189d77627c4d742822c3.svg?invert_in_darkmode" align=middle width=14.433101099999991pt height=14.15524440000002pt/>
by using their
<img src="svgs/ec468df79768509d15af76fd10a0ac47.svg?invert_in_darkmode" align=middle width=14.337957149999992pt height=22.831056599999986pt/>.

<img src="svgs/90bcdfbb8b61c9fff60fb3be5aab78e9.svg?invert_in_darkmode" align=middle width=249.24362759999997pt height=24.65753399999998pt/>.
The issuer of the MAC can verify a MAC
<img src="svgs/4f4f4e395762a3af4575de74c019ebb5.svg?invert_in_darkmode" align=middle width=5.936097749999991pt height=20.221802699999984pt/>
given the message
<img src="svgs/0e51a2dede42189d77627c4d742822c3.svg?invert_in_darkmode" align=middle width=14.433101099999991pt height=14.15524440000002pt/>
it was issued on.

One might intuitively think of a MAC as the symmetric-key counterpart of
digital signatures. They both have the same goals and similar security
requirements, however a MAC requires a secret rather than public key to
verify.

## Zero-knowledge proofs of knowledge

A very high-level, and hence somewhat imprecise, description of
zero-knowledge proofs is provided. This protocol involves a prover and a
verifier. A prover wishes to prove that a relation
<img src="svgs/0ea3cec9cd8f8324b87218b762b9a686.svg?invert_in_darkmode" align=middle width=13.931567099999992pt height=22.465723500000017pt/>
holds with respect to a secret input
<img src="svgs/31fae8b8b78ebe01cbfbe2fe53832624.svg?invert_in_darkmode" align=middle width=12.210846449999991pt height=14.15524440000002pt/>,
called witness, and public input
<img src="svgs/332cc365a4987aacce0ead01b8bdcc0b.svg?invert_in_darkmode" align=middle width=9.39498779999999pt height=14.15524440000002pt/>.
Specifically, the prover wants to prove that
<img src="svgs/aa841c18bd4ecee7c7f49e695e298fd6.svg?invert_in_darkmode" align=middle width=75.71985464999999pt height=24.65753399999998pt/>
without revealing anything about
<img src="svgs/31fae8b8b78ebe01cbfbe2fe53832624.svg?invert_in_darkmode" align=middle width=12.210846449999991pt height=14.15524440000002pt/>.

<img src="svgs/259460131580db9ca5a4d2dc866cc8ff.svg?invert_in_darkmode" align=middle width=137.68948709999998pt height=24.65753399999998pt/>.
Given
<img src="svgs/332cc365a4987aacce0ead01b8bdcc0b.svg?invert_in_darkmode" align=middle width=9.39498779999999pt height=14.15524440000002pt/>
and the private witness
<img src="svgs/31fae8b8b78ebe01cbfbe2fe53832624.svg?invert_in_darkmode" align=middle width=12.210846449999991pt height=14.15524440000002pt/>
the prover generates a proof
<img src="svgs/f30fdded685c83b0e7b446aa9c9aa120.svg?invert_in_darkmode" align=middle width=9.96010619999999pt height=14.15524440000002pt/>.

<img src="svgs/52374d2fa50dc14bdd44a402a8f23595.svg?invert_in_darkmode" align=middle width=222.82512779999996pt height=24.65753399999998pt/>.
The verifier is given the proof
<img src="svgs/f30fdded685c83b0e7b446aa9c9aa120.svg?invert_in_darkmode" align=middle width=9.96010619999999pt height=14.15524440000002pt/>
and
<img src="svgs/332cc365a4987aacce0ead01b8bdcc0b.svg?invert_in_darkmode" align=middle width=9.39498779999999pt height=14.15524440000002pt/>
with which they determine whether the prover knows a secret
<img src="svgs/31fae8b8b78ebe01cbfbe2fe53832624.svg?invert_in_darkmode" align=middle width=12.210846449999991pt height=14.15524440000002pt/>
such that
<img src="svgs/8997e80a7dd177c0b9632c94bd8faca1.svg?invert_in_darkmode" align=middle width=85.76555789999999pt height=24.65753399999998pt/>
holds.

# Protocol Overview

## Phases

A CoinJoin round consists of an Input Registration, an Output
Registration and a Transaction Signing phase. To defend against Denial
of Service attacks it is important to ensure the inputs of users who do
not comply with the protocol are identified so these inputs can be
excluded from the following rounds in order to ensure completion of the
protocol.

1.  While identifying non-compliant inputs during Input Registration
    phase is trivial, there is no reason to issue penalties at this
    point.

2.  Identifying non-compliant inputs during Output Registration phase is
    not possible, thus this phase always completes and progresses to the
    Signing phase.

3.  During Signing phase, inputs which are not signed are non-compliant
    inputs and they shall be issued penalties.

The cryptography in WabiSabi ensures honest participants always agree to
sign the final CoinJoin transaction if the coordinator is honest.
Anonymous credentials allow the coordinator to verify that amounts of
each user’s output registrations are funded by input registrations
without learning specific relationships between inputs and outputs.

## Credentials

The coordinator issues anonymous credentials which authenticate
attributes in response to registration requests. We use
keyed-verification anonymous credentials (introduced
in \[@chase2014algebraic\]), in particular the scheme
from \[@chase2019signal\] which supports group attributes (attributes
whose value is an element of the underlying group
<img src="svgs/a158a43ace9779e0a6109b3c9f9df93d.svg?invert_in_darkmode" align=middle width=12.785434199999989pt height=22.648391699999998pt/>).
A user can then prove possession of a credential in zero knowledge in a
subsequent registration request, without the coordinator being able to
link it to the registration from which it originates.

In order to facilitate construction of a CoinJoin transaction while
protecting the privacy of participants, we instantiate the scheme with a
single group attribute
<img src="svgs/f6e2d111bf312f383d5c12410e3abf3b.svg?invert_in_darkmode" align=middle width=23.077896599999992pt height=22.465723500000017pt/>
which encodes a confidential Bitcoin amount as a Pedersen commitment.
These commitments are never opened. Instead, properties of the values
they commit to are proven in zero knowledge, allowing the coordinator to
validate requests made by honest participants. In ideal circumstances
the coordinator would not learn anything beyond what can be learned from
the resulting CoinJoin transaction but despite the unlinkability of the
credentials timing of requests or connectivity issues may still reveal
information about links.

## Registration

To aid intuition we first describe a pair of protocols, where
credentials are issued during input registration, and then then
presented at output registration.
<img src="svgs/63bb9849783d01d91403bc9a5fea12a2.svg?invert_in_darkmode" align=middle width=9.075367949999992pt height=22.831056599999986pt/>
denotes the number of credentials used in registration requests, and
<img src="svgs/9768aee3dd5e797ce590c2453c567f07.svg?invert_in_darkmode" align=middle width=107.49404324999999pt height=26.76175259999998pt/>
constrains the range of amount values\[8\]. For better privacy and
efficiency these are then generalized into a unified protocol used for
both input and output registration, where every registration involves
both presentation and issuance of credentials. This protocol is
described in detail in [4](#details).

In order to maintain privacy clients must isolate registration requests
using unique network identities. A single network identity must not
expose more than one input or output, or more than one set of requested
or presented credentials.

For fault tolerance, request handling should be idempotent, allowing a
client to retry a failed request without modification using a fresh
network identity or one which was previously used to attempt that
request.

### Input Registration

1.  The user sends
    <img src="svgs/63bb9849783d01d91403bc9a5fea12a2.svg?invert_in_darkmode" align=middle width=9.075367949999992pt height=22.831056599999986pt/>
    credential requests with accompanying range and sum proofs to the
    coordinator:
    \(((M_{a_i},\pi^{\textit{range}}_{i})^{k}_{i=1},\pi^{sum},a_{\textit{in}})\).

2.  The coordinator verifies the received proofs. If they are not
    verified it aborts the protocol, otherwise it issues \(k\) MACs on
    the requested attributes
    \((\mathsf{MAC}_\mathsf{sk}(M_{a_i}), \pi_i^{\mathrm{iparams}})^{k}_{i=1}\).

The user submits an input of amount
<img src="svgs/fe6d8bce13575ec446165120ad627178.svg?invert_in_darkmode" align=middle width=21.533696249999988pt height=14.15524440000002pt/>
along with
<img src="svgs/63bb9849783d01d91403bc9a5fea12a2.svg?invert_in_darkmode" align=middle width=9.075367949999992pt height=22.831056599999986pt/>
group attributes,
<img src="svgs/fd9776ca95b1721114ee3cb1c72caa93.svg?invert_in_darkmode" align=middle width=41.892663449999986pt height=24.65753399999998pt/>.
She proves in zero knowledge that the sum of the requested sub-amounts
is equal to
<img src="svgs/fe6d8bce13575ec446165120ad627178.svg?invert_in_darkmode" align=middle width=21.533696249999988pt height=14.15524440000002pt/>
and that the individual amounts are positive integers in the allowed
range.

The coordinator verifies the proofs, and issues
<img src="svgs/63bb9849783d01d91403bc9a5fea12a2.svg?invert_in_darkmode" align=middle width=9.075367949999992pt height=22.831056599999986pt/>
MACs on the requested attributes, along with a proof of correct
generation of the MAC, as in *Credential Issuance* protocol of
\[@chase2019signal\].

### Output Registration

1.  The user sends
    <img src="svgs/63bb9849783d01d91403bc9a5fea12a2.svg?invert_in_darkmode" align=middle width=9.075367949999992pt height=22.831056599999986pt/>
    randomized commitments, a proof of a valid MAC for the corresponding
    non-randomized commitments, serial numbers with a proof of their
    validity, and finally a proof of the sum of the amounts:
    \(((C_{a_i},\pi_{i}^{\textit{MAC}},S_i,\pi_i^{\textit{serial}})^{k}_{i=1}, \pi^{\textit{sum}}, a_{\textit{out}})\).

2.  The coordinator verifies proofs and registers requested output iff.
    all proofs are valid and the serial numbers have not been used
    before.

To register her output the user randomizes the attributes and generates
a proof of knowledge of
<img src="svgs/63bb9849783d01d91403bc9a5fea12a2.svg?invert_in_darkmode" align=middle width=9.075367949999992pt height=22.831056599999986pt/>
valid credentials issued by the coordinator.

Additionally, she proves the serial number is valid. These serial
numbers are required for double spending protection, and must be
correspond but unlinkable to a specific
<img src="svgs/f6e2d111bf312f383d5c12410e3abf3b.svg?invert_in_darkmode" align=middle width=23.077896599999992pt height=22.465723500000017pt/>.

Finally, she proves that the sum of her randomized amount attributes
<img src="svgs/22855e24c86e4999797fdb82dc83ddfd.svg?invert_in_darkmode" align=middle width=18.879232349999988pt height=22.465723500000017pt/>
matches the requested output amount
<img src="svgs/de6e97763746027db53a5b7319af8f8f.svg?invert_in_darkmode" align=middle width=28.553551949999992pt height=14.15524440000002pt/>,
analogously to input registration.\[9\]

She submits these proofs, the randomized attributes, and the serial
numbers. The coordinator verifies the proofs, and if accepted the output
will be included in the transaction.

### Unified Registration

In order to increase flexibility in a dynamic setting, where a user may
not yet know her desired output allocations during input registration,
and to allow setting a small\[10\] value of
<img src="svgs/63bb9849783d01d91403bc9a5fea12a2.svg?invert_in_darkmode" align=middle width=9.075367949999992pt height=22.831056599999986pt/>
as a protocol level constant to reduce privacy leaks, we can generalize
input and output registration into a single unified protocol for use in
both phases, which also supports reissuance. For complete definitions
see [4](#details).

1.  During both input and output registration phases the user submits:
    
      - \(k\) credential requests with accompanying range and sum proofs
        to the coordinator:
        \((M_{a_i},\pi^{\textit{range}}_{i})^{k}_{i=1}\)
    
      - \(k\) randomized commitments, proofs of valid credentials issued
        for the corresponding non-randomized commitments, serial
        numbers, and proofs of their validity:
        \((C_{a_i},\pi_{i}^{\mathit{MAC}},S_i,\pi_i^{\textit{serial}})^{k}_{i=1}\)
    
      - A balance \(\Delta_{a}\) and a proof of its correctness
        \(\pi^{\textit{sum}}\)
    
      - If \(\Delta_{a} \ne 0\), an input or output with value
        \(|\Delta_{a}|\).

2.  The coordinator verifies the received proofs, and that the serial
    numbers have not been used before, and depending on the current
    phase, \(\Delta_{a} \geq 0\) (input) or \(\Delta_{a} \leq 0\)
    (output). If it accepts, it issues \(k\) MACs on the requested
    attributes
    \((\mathsf{MAC}_\mathsf{sk}(M_{a_i}), \pi_i^{\mathrm{iparams}})^{k}_{i=1}\),
    and if \(\Delta_{a} \ne 0\), registers the input or output with
    value \(|\Delta_{a}|\).

The user submits
<img src="svgs/63bb9849783d01d91403bc9a5fea12a2.svg?invert_in_darkmode" align=middle width=9.075367949999992pt height=22.831056599999986pt/>
valid credentials and
<img src="svgs/63bb9849783d01d91403bc9a5fea12a2.svg?invert_in_darkmode" align=middle width=9.075367949999992pt height=22.831056599999986pt/>
credential requests, where the sums of the underlying amount commitments
must be balanced ([\[fig:reissue\]](#fig:reissue)).

1.  During input registration phase the user submits
    <img src="svgs/63bb9849783d01d91403bc9a5fea12a2.svg?invert_in_darkmode" align=middle width=9.075367949999992pt height=22.831056599999986pt/>
    credential requests: \((M_{a_i},\pi^{\mathit{null}}_{i})^{k}_{i=1}\)

2.  The coordinator verifies the received proofs. If it accepts, it
    issues \(k\) MACs on the requested attributes
    \((\mathsf{MAC}_\mathsf{sk}(M_{a_i}), \pi_i^{\mathrm{iparams}})^{k}_{i=1}\).

To prevent the coordinator from being able to distinguish between
initial vs. subsequent input registration requests (which may merge
amounts) credential presentation should be mandatory. Initial
credentials can be obtained with an auxiliary bootstrapping operation
([\[fig:bootstrap\]](#fig:bootstrap)).

## Signing phase

The user fetches the finalized but unsigned transaction from the
coordinator. If it contains the outputs she registered she will sign her
inputs and submit each signature separately using the network identity
used for that input’s registration.

## Examples

To illustrate the above protocols,
[\[fig:ex1,fig:ex2\]](#fig:ex1,fig:ex2) show how a user might register
inputs and outputs when credentials are only presented during the output
registration phase and [\[fig:ex3,fig:ex4\]](#fig:ex3,fig:ex4) show the
unified protocol, when credentials are both presented and requested in
every registration request.

Registration requests are depicted as vertices labeled with
<img src="svgs/8e47f6081dd0cf790cc02b5d1f58f8d7.svg?invert_in_darkmode" align=middle width=20.829053849999994pt height=22.465723500000017pt/>,
a double stroke denoting output registrations. A credential is an edge
from the registration in which it was requested to the registration
where it was presented, also labeled with the amount. The sum of a
vertex’s label and the labels of its incoming edges must be equal to the
sum of the labels of its outgoing edges. Note that edges and their
labels are only known to the owners of the credentials. For simplicity
we omit credentials with zero value.

# Cryptographic Details

Following \[@chase2019signal\], the credential scheme for the protocol
in [3.3.3](#unified) is defined over a group
<img src="svgs/a158a43ace9779e0a6109b3c9f9df93d.svg?invert_in_darkmode" align=middle width=12.785434199999989pt height=22.648391699999998pt/>
of prime order
<img src="svgs/db5fafda9a00f45985de8992d8e78ade.svg?invert_in_darkmode" align=middle width=12.49431149999999pt height=14.15524440000002pt/>
written in multiplicative notation.
<img src="svgs/edc712484876946d0f09a330d8b2cade.svg?invert_in_darkmode" align=middle width=164.9542521pt height=24.65753399999998pt/>
is a function from strings to group elements, based on a cryptographic
hash function\[@fouque2012indifferentiable\].

We require the following fixed set of group elements for use as
generators with different
purposes:

<p align="center">

<img src="svgs/aae1a268d4674f54dad1d0e191058a2c.svg?invert_in_darkmode" align=middle width=483.39869325pt height=38.25102765pt/>

</p>

chosen so that nobody knows the discrete logarithms between any pair of
them, e.g.
<img src="svgs/3403d4b234d785260611bd5623875755.svg?invert_in_darkmode" align=middle width=146.3741169pt height=24.65753399999998pt/>.

Our notation deviates slightly from \[@chase2019signal\], in that we
subscript the attribute generators
<img src="svgs/9a615908a33eb3da7528aa0275517dc4.svg?invert_in_darkmode" align=middle width=23.976908999999992pt height=22.465723500000017pt/>
as
<img src="svgs/4509ae74a42e5c18a663522ee6324d1b.svg?invert_in_darkmode" align=middle width=20.05502564999999pt height=22.465723500000017pt/>
instead of using numerical indices, and we require two additional
generators
<img src="svgs/bee8c731a74cd7704c2eafecd09eb0d4.svg?invert_in_darkmode" align=middle width=19.75061384999999pt height=22.465723500000017pt/>
and
<img src="svgs/316af53c14c3acaf3ed626bda068df7e.svg?invert_in_darkmode" align=middle width=20.62067039999999pt height=22.465723500000017pt/>
for constructing the attribute
<img src="svgs/f6e2d111bf312f383d5c12410e3abf3b.svg?invert_in_darkmode" align=middle width=23.077896599999992pt height=22.465723500000017pt/>
as a Pedersen commitment.

As with the generator names, we modify the names of the attribute
related components of the secret key
<img src="svgs/784ec2f16f1a214851639d0fced28dd2.svg?invert_in_darkmode" align=middle width=213.31670085pt height=29.3516652pt/>
according to our fixed set of group attributes.

The coordinator parameters
<img src="svgs/9becce7215859239c9c0509d76a47ebf.svg?invert_in_darkmode" align=middle width=134.2391523pt height=24.65753399999998pt/>
are computed
as:

<p align="center">

<img src="svgs/a67f20421f7e9fe4bc7b7f649b8939c9.svg?invert_in_darkmode" align=middle width=294.94176855pt height=37.7389848pt/>

</p>

and published as part of the round metadata and are used by the
coordinator to prove correctness of issued MACs, and by the users to
prove knowledge of a valid MAC.

## Credential Requests

For each
<img src="svgs/0905f654cdcdfe9b0e4c9d390c8ba678.svg?invert_in_darkmode" align=middle width=59.48726684999998pt height=24.65753399999998pt/>
the user chooses an amount
<img src="svgs/1040a3addcd2452968a6196ff688c460.svg?invert_in_darkmode" align=middle width=128.37474869999997pt height=24.65753399999998pt/>
subject to the constraints of the balance proof ([4.5](#balance)). She
commits to the amount with randomness
<img src="svgs/1c6cbbc8005ea1cbe3c555a07e27cc37.svg?invert_in_darkmode" align=middle width=61.160741399999985pt height=22.648391699999998pt/>,
and these commitments are the attributes of the requested
credentials:

<p align="center">

<img src="svgs/921c825ad74b9adba785ad6afca6f36c.svg?invert_in_darkmode" align=middle width=116.6887161pt height=16.826464050000002pt/>

</p>

For each amount
<img src="svgs/65ed4b231dcf18a70bae40e50d48c9c0.svg?invert_in_darkmode" align=middle width=13.340053649999989pt height=14.15524440000002pt/>
she also computes a range proof which ensures there are no negative
values:

<p align="center">

<img src="svgs/167dd7b2983e0be29b88dafbb43219f1.svg?invert_in_darkmode" align=middle width=407.6328498pt height=17.55700485pt/>

</p>

In credential bootstrap requests the range proofs can be replaced with
simpler proofs of
<img src="svgs/bf4d4fc7c8c087f4c8bfe06091dbaa63.svg?invert_in_darkmode" align=middle width=44.298790799999985pt height=21.18721440000001pt/>:

<p align="center">

<img src="svgs/305b793d59f1380ae0cdad58f0d60641.svg?invert_in_darkmode" align=middle width=223.58635364999998pt height=18.88772655pt/>

</p>

We note that if Bulletproofs \[@bunz2018bulletproofs\] are utilized for
the range proofs
<img src="svgs/9d78bfeb6d23f54c2c2a91716d316d6d.svg?invert_in_darkmode" align=middle width=42.96847004999999pt height=25.70766330000001pt/>
a combined proof will significantly decrease the communication overhead
and that some implementations perform the
<img src="svgs/9924953b83d1e1007d3c2a8736a3d399.svg?invert_in_darkmode" align=middle width=31.569788249999988pt height=27.91243950000002pt/>
optimization already.

## Credential Issuance

If the coordinator accepts the requests (see
[\[presentation,serial,balance\]](#presentation,serial,balance)), it
registers the input or output if one is provided, and for each
<img src="svgs/a27c593d3d99c59063571284351b61a3.svg?invert_in_darkmode" align=middle width=59.48726684999998pt height=24.65753399999998pt/>
it issues a credential by responding with
<img src="svgs/be42bc09ea6d761dce4e05b29bb332bd.svg?invert_in_darkmode" align=middle width=117.74849294999997pt height=24.65753399999998pt/>,
which is the output of
<img src="svgs/d2ba0363c07bde0c379e2686035880ae.svg?invert_in_darkmode" align=middle width=88.76277134999998pt height=24.65753399999998pt/>,
where:

<p align="center">

<img src="svgs/a21307ddc70812dfd4f627fa76aad5d6.svg?invert_in_darkmode" align=middle width=433.07302995pt height=18.995433600000002pt/>

</p>

To rule out tagging of individual users the coordinator must prove
knowledge of the secret key, and that
<img src="svgs/2fb34d64b1ed37eb410b3c76810f991a.svg?invert_in_darkmode" align=middle width=70.5644775pt height=24.65753399999998pt/>
are correct relative to
<img src="svgs/9d6085c164b4ed27f7c25d2e9caadf05.svg?invert_in_darkmode" align=middle width=134.2391523pt height=24.65753399999998pt/>:

<p align="center">

<img src="svgs/107ece831005b5d7f938ae8fbe1a85c4.svg?invert_in_darkmode" align=middle width=300.5544399pt height=119.8551189pt/>

</p>

## Credential Presentation

The user chooses
<img src="svgs/63bb9849783d01d91403bc9a5fea12a2.svg?invert_in_darkmode" align=middle width=9.075367949999992pt height=22.831056599999986pt/>
unused credentials issued in prior registration requests, i.e. valid
MACs
<img src="svgs/b6c9657780a473ee4ba639ef2eaac4c9.svg?invert_in_darkmode" align=middle width=67.85687865pt height=27.91243950000002pt/>
on attributes
<img src="svgs/8c5f585fcbe6e9ba3b0a56d368dca48f.svg?invert_in_darkmode" align=middle width=63.18748259999999pt height=27.91243950000002pt/>.

For each credential
<img src="svgs/0905f654cdcdfe9b0e4c9d390c8ba678.svg?invert_in_darkmode" align=middle width=59.48726684999998pt height=24.65753399999998pt/>
she executes the
<img src="svgs/f8336050aaa87118dda2feeca5d8928b.svg?invert_in_darkmode" align=middle width=36.84942689999999pt height=22.831056599999986pt/>
protocol described in \[@chase2019signal\]:

1.  She chooses
    <img src="svgs/93d2ca33eacb2c125b743bf05193756a.svg?invert_in_darkmode" align=middle width=61.389074999999984pt height=22.648391699999998pt/>,
    and computes \(z_{0_i}=-{t_i} {z_i} (\bmod q)\) and the randomized
    commitments: \[\begin{aligned}
    C_{a_i}     &= {G_a}^{z_i} M_{a_i} \\
    C_{x_{0_i}} &= {G_{x_0}}^{z_i} {U_i} \\
    C_{x_{1_i}} &= {G_{x_1}}^{z_i} {U_i}^{t_i} \\
    C_{V_i}     &= {G_V}^{z_i} V_i\end{aligned}\]

2.  To prove to the coordinator that a credential is valid she computes
    a proof: \[\begin{aligned}
    \pi_{i}^{\mathit{MAC}}=\operatorname{PK}\{
    & (z_i, z_{0_i},t_i): \\
    & Z_i =I^{z_i} \land \\
    & C_{x_{1_i}} = {C_{x_{0_i}}}^{t_i} {G_{x_0}}^{z_{0_i}} {G_{x_1}}^{z_i} \}\end{aligned}\]
    which implies the following without allowing the coordinator to link
    \(\pi_{i}^\mathit{MAC}\) to the underlying attributes \((M_{a_i})\):
    \[\mathsf{Verify}((C_{x_{0_i}}, C_{x_{1_i}}, C_{V_i}, C_{a_i}, Z_i), \pi_i^{\mathit{MAC}})
    \iff
    \mathsf{VerifyMAC}_{\mathsf{sk}}(M_{a_i})\]

3.  She sends
    \((C_{x_{0_i}}, C_{x_{1_i}}, C_{V_i}, C_{a_i},\pi_i^{\mathit{MAC}})\)
    and the coordinator computes:
    \[Z_i=\frac{C_{V_i}}{{G_w}^w {C_{x_{0_i}}}^{x_0} {C_{x_{1_i}}}^{x_{1}}
    {C_{a_i}}^{y_a}
    }\] using its secret key (independently of the user’s derivation),
    and verifies \(\pi_i^{\mathit{MAC}}\).

## Double-spending prevention using serial numbers

The user proves that the group element
<img src="svgs/eee8d94153fbfd00c049ccfb1beb7ec6.svg?invert_in_darkmode" align=middle width=67.91035184999998pt height=24.246581700000014pt/>,
which is used as a serial number, was generated correctly with respect
to
<img src="svgs/8b3f78dbf9ef32e72d66b7ff5a6942d6.svg?invert_in_darkmode" align=middle width=23.26475744999999pt height=22.465723500000017pt/>:

<p align="center">

<img src="svgs/b18a780fd55f3470e5f0393407bc2856.svg?invert_in_darkmode" align=middle width=422.93468579999995pt height=19.4813124pt/>

</p>

The coordinator verifies
<img src="svgs/f28d898b96688980cba64dd521066eec.svg?invert_in_darkmode" align=middle width=43.38053609999999pt height=27.91243950000002pt/>
and checks that the
<img src="svgs/d28140eda2d12e24b434e011b930fa23.svg?invert_in_darkmode" align=middle width=14.730823799999989pt height=22.465723500000017pt/>
has not been used before (allowing for idempotent registration).

Note that since the logical conjunction of
<img src="svgs/b43e4e1a5c2c2d032d5760fd89b46894.svg?invert_in_darkmode" align=middle width=43.38053609999999pt height=27.91243950000002pt/>
and
<img src="svgs/09abc9e7ef0473fb358d066523da0cb4.svg?invert_in_darkmode" align=middle width=42.84862394999999pt height=27.53345100000001pt/>
is required for each credential, and because these proofs share both
public and private inputs it is appropriate to use a single proof for
both statements.

## Over-spending prevention by balance proof

The user needs to convince the coordinator that the total amounts
redeemed and the requested differ by the public input
<img src="svgs/39e231140eb0bd69012064a82523bbff.svg?invert_in_darkmode" align=middle width=20.829053849999994pt height=22.465723500000017pt/>,
which she can prove by including the following proof of
knowledge:

<p align="center">

<img src="svgs/beb836de7dd64724478307e69738e3e6.svg?invert_in_darkmode" align=middle width=277.23603929999996pt height=19.14156915pt/>

</p>

where

<p align="center">

<img src="svgs/5e7a4ff392a35a7f638a0fe93ca2f7a8.svg?invert_in_darkmode" align=middle width=384.4855773pt height=47.93392394999999pt/>

</p>

with
<img src="svgs/451ae922742ce25462fdc878eb175f61.svg?invert_in_darkmode" align=middle width=12.067218899999991pt height=24.7161288pt/>
denoting the randomness terms in the
<img src="svgs/8db70e85798d105dba27ef6dd4286a12.svg?invert_in_darkmode" align=middle width=63.18748259999999pt height=27.91243950000002pt/>
attributes of the credentials being requested and
<img src="svgs/f40459d04ba6d61892742760c6b75725.svg?invert_in_darkmode" align=middle width=32.49055259999999pt height=14.15524440000002pt/>
denoting the ones in the randomized attributes
<img src="svgs/db2d98db8bbca9c19b6834b36757e69d.svg?invert_in_darkmode" align=middle width=58.988816699999994pt height=27.91243950000002pt/>
of the credentials being presented.

During the input registration phase
<img src="svgs/39e231140eb0bd69012064a82523bbff.svg?invert_in_darkmode" align=middle width=20.829053849999994pt height=22.465723500000017pt/>
may be positive, in which case an input of amount
<img src="svgs/0959b651e4924fcdc9b9fab5edf8962b.svg?invert_in_darkmode" align=middle width=65.1022548pt height=22.465723500000017pt/>
must be registered with proof of ownership. During the output
registration phase
<img src="svgs/39e231140eb0bd69012064a82523bbff.svg?invert_in_darkmode" align=middle width=20.829053849999994pt height=22.465723500000017pt/>
may be negative, in which case an output of amount
<img src="svgs/8ba1cf11ba9b4e0512034155acd9f3e6.svg?invert_in_darkmode" align=middle width=84.90752489999998pt height=22.465723500000017pt/>
is registered. If
<img src="svgs/111618406e0b06e49912214f58f5fffa.svg?invert_in_darkmode" align=middle width=51.787807499999985pt height=22.465723500000017pt/>
credentials are simply reissued, with no input or output registration
occurring.

## Perfect Hiding

Note that
<img src="svgs/d28140eda2d12e24b434e011b930fa23.svg?invert_in_darkmode" align=middle width=14.730823799999989pt height=22.465723500000017pt/>
is not perfectly hiding because there is exactly one
<img src="svgs/221b254c5d1b3e5f56a6378fcb665729.svg?invert_in_darkmode" align=middle width=50.377032749999984pt height=22.648391699999998pt/>
such that
<img src="svgs/eee8d94153fbfd00c049ccfb1beb7ec6.svg?invert_in_darkmode" align=middle width=67.91035184999998pt height=24.246581700000014pt/>.
Similarly, randomization by
<img src="svgs/6af8e9329c416994c3690752bde99a7d.svg?invert_in_darkmode" align=middle width=12.29555249999999pt height=14.15524440000002pt/>
only protects unlinkability of issuance and presentation against a
computationally bounded adversary. Null credentials have the same issue,
since the amount exponent is known to be zero.

To unconditionally preserve user privacy in the event that the hardness
assumption of the discrete logarithm problem in
<img src="svgs/a158a43ace9779e0a6109b3c9f9df93d.svg?invert_in_darkmode" align=middle width=12.785434199999989pt height=22.648391699999998pt/>
is broken we can add an additional randomness term
<img src="svgs/d6b9828f3535066cb8eed52f6a0ab836.svg?invert_in_darkmode" align=middle width=12.067218899999991pt height=24.7161288pt/>
used with an additional generator
<img src="svgs/0774c177e347c9001f360f04f86d07b3.svg?invert_in_darkmode" align=middle width=20.62067039999999pt height=24.7161288pt/>
to the amount commitments
<img src="svgs/24a876c045eb056140e86f0b8dff4ea1.svg?invert_in_darkmode" align=middle width=27.46342169999999pt height=22.465723500000017pt/>,
and similarly another randomness term
<img src="svgs/980f7e5a53248edfc9291e3963dcd0a6.svg?invert_in_darkmode" align=middle width=12.29555249999999pt height=24.7161288pt/>
and generators
<img src="svgs/3709121eb447b5e6e6d84a504b435771.svg?invert_in_darkmode" align=middle width=121.54595969999998pt height=24.7161288pt/>
in order to obtain unconditional unlinkability for the
commitments.\[11\]

# Acknowledgements

We would like to acknowledge the inputs and invaluable contributions of
ZmnSCPxj, Yahia Chiheb, Thaddeus Dryja, Adam Gibson, Dan Gould, Ethan
Heilman, Max Hillebrand, Aviv Milner, Jonas Nick, Lucas Ontivero, Tim
Ruffing, Ruben Somsen and Greg Zaverucha to this paper.

\[1\] In this work we restrict the discussion of Bitcoin privacy to that
of public ledger transactions, but there are other considerations
especially at the network layer. For a more comprehensive discussion see
<https://en.bitcoin.it/wiki/Privacy>.

\[2\] Approximately
<img src="svgs/22f2e6fc19e491418d1ec4ee1ef94335.svg?invert_in_darkmode" align=middle width=21.00464354999999pt height=21.18721440000001pt/>

\[3\] The observed base denominations in Wasabi’s CoinJoins are usually
slightly higher than the announced, agreed upon base denomination. Thus
participants sometimes get back slightly more value in the CoinJoins
than they put in.

\[4\] <https://en.bitcoin.it/wiki/PayJoin>

\[5\] Note that the cleartext amounts appearing in the final transaction
might still link individual inputs and outputs.

\[6\] <https://github.com/sharedcoin/Sharedcoin>

\[7\] <https://github.com/cashshuffle/spec>

\[8\]
<img src="svgs/3a9f2d6b1cade1d64ae926c51d915e1e.svg?invert_in_darkmode" align=middle width=224.04163814999995pt height=24.65753399999998pt/>

\[9\] Note that there is no need for range proofs, since amounts have
been previously validated.

\[10\] Specifically,
<img src="svgs/7aa6d81030e08dbeb35910108b6afdd7.svg?invert_in_darkmode" align=middle width=308.09402745pt height=37.80850590000001pt/>
the maximum number of participants, because although
<img src="svgs/7eb22be4bf74527b54b6d60938478147.svg?invert_in_darkmode" align=middle width=39.21220214999999pt height=22.831056599999986pt/>
suffices for flexibility it limits parallelism, leaking privacy by
temporal fingerprinting. The limit on participant count is because 274
and 124 are the minimum weight units required for a participant with
only a single input and output, and 58 is the shared per transaction
overhead.

\[11\] Assuming the coordinator is not able to attack the network level
privacy and the proofs of knowledge are unconditionally hiding.
