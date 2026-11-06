# DID-locked ecash 1.0

Protocol URI: `https://archetech.com/didcomm/cashu/1.0`

A sender moves [Cashu](https://cashubtc.github.io/nuts/) ecash to a DID by locking it
to a key of that DID and sending the token as an encrypted DIDComm v2 message. A token
intercepted in transit cannot be spent, and one never claimed returns to the sender.

The keywords MUST, MUST NOT, SHOULD and MAY are used as in RFC 2119.

## Roles

- **Sender**: holds ecash at a mint, and locks some of it to the recipient.
- **Recipient**: holds the private key the token is locked to.
- **Mint**: enforces the lock. It MUST support NUT-11.

## Message

```json
{
  "id": "2f4f6c0e8f0b4c1ca1e7f2f3f9a7a3f1",
  "type": "https://archetech.com/didcomm/cashu/1.0/token",
  "from": "did:cid:sender",
  "to": ["did:cid:recipient"],
  "body": {
    "token": "cashuB...",
    "memo": "optional"
  }
}
```

The body is defined by [token.schema.json](token.schema.json).

| Field | Required | Meaning |
|---|---|---|
| `token` | yes | A Cashu token, in the V4 `cashuB` format of NUT-00. A receiver SHOULD also accept V3 `cashuA`. |
| `memo` | no | Text for the recipient. It has no effect on the value. |

`id` identifies the message across resends: a sender resending a message MUST use the
same `id`. The message SHOULD be authcrypted, so the recipient learns who sent it. A
recipient MUST NOT treat `from` as the sender unless the envelope authenticates it.

## Choosing a mint

The sender chooses the mint. Accepting a mint's tokens means trusting that mint with the
value, so a recipient MUST NOT contact a mint it has not accepted in order to redeem a
token. A recipient that does not accept the mint leaves the token unredeemed, and the
sender can reclaim the value after the locktime.

The sender MUST check that the mint advertises NUT-11 in its NUT-06 info, and MUST NOT
send through a mint that does not, because such a mint may treat locked proofs as bearer
tokens.

## Choosing the key

The sender locks the token to the public key of the first verification method of the
recipient's DID document, as resolved at its most recent anchored version. DID keys are
secp256k1, the curve NUT-11 uses; the key is written in compressed form, 33 bytes in hex.
A recipient MUST match a lock to its keys by x-coordinate, because NUT-11 signatures are
BIP-340 and either parity prefix names the same key.

A recipient that rotates its keys MUST keep the private keys of earlier derivations until
tokens locked to them are redeemed.

## Locking

Every proof in the token MUST have a NUT-11 `P2PK` secret with:

| Field or tag | Value |
|---|---|
| `data` | The recipient's key |
| `sigflag` | `SIG_INPUTS`. As this is the NUT-11 default, the tag MAY be omitted. |
| `locktime` | Unix seconds at which the sender may take the value back |
| `refund` | A key the sender controls |

The `refund` tag is REQUIRED: under NUT-11 a proof past its locktime with no refund key
is spendable by anyone.

The locktime SHOULD be no later than the recipient's mailbox would keep an
unacknowledged message, so the value can be reclaimed before the message could expire
unread. Archon uses three days by default and allows at most seven.

## Redeeming

The recipient signs each proof's secret as NUT-11 requires, a BIP-340 signature on the
SHA-256 of the secret, and swaps the proofs at the mint for new ones.

A recipient:

- MUST NOT delete the message from its mailbox until the redemption has settled, or has
  been recorded so that it can be finished after a restart. The message may be the only
  copy of the value.
- MUST deduplicate by sender and `id`, so a resent message is not redeemed twice.
- SHOULD leave a message it cannot redeem yet in its mailbox, as when the mint is not
  accepted or cannot be reached, and report the reason.
- SHOULD redeem what remains of a token only partly spent, swapping the unspent proofs and
  reporting how much was already gone. It MUST NOT count those proofs as its own until the
  mint has signed new ones for them, and MUST keep the message until that has settled.
- MAY delete a message whose token the mint reports as wholly spent, once that has been
  reported.
- MUST NOT let a malformed message stop it from processing the others.

General-purpose inbox handling SHOULD NOT acknowledge these messages, for the same reason.

## Refund

From the locktime on, the sender can spend the proofs with the refund key.

The lock does not end the recipient's right to redeem: NUT-11 keeps the main key valid
past the locktime. Both can then spend, and the first to reach the mint gets the value.
A recipient that finds its token spent should conclude it was refunded or already
redeemed, not that anything failed.

A sender SHOULD keep what it needs to refund, the proofs with their secrets, until the
token is redeemed or refunded. These secrets are random rather than derived from a seed,
so they cannot be recovered from a mnemonic.

## Security

- The token is bearer value to whoever holds the key. The envelope hides it from relays,
  and the lock protects it if the envelope is opened or the token is copied elsewhere.
- The mint can see the locked public key and the locktime when it signs and spends.
  It cannot tell which DID the key belongs to unless it resolves DID documents to look.
- A mint can still steal or refuse ecash, as with any Cashu token. The lock limits who
  else can spend it, not what the mint can do.
