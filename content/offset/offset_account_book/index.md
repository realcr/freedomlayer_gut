+++
title = "Design of Offset account book"
description = "Thoughts about the design of the Offset account book"
date = 2021-02-17
+++

This post is about to describe some thoughts I had regarding
the design of the Offset account book. Just in case this is your first time hearing
about Offset, [Offset](https://www.offsetcredit.org) is a decentralized
infrastructure allowing trading without money. Members can trade with each
other on the basis of credit accounting. To save some energy for both the
readers and me, Offset is not a blockchain, and it will not make you rich.

Recently a friend asked me what I was working on, and my reply was that I was
having some problems with the design of the Offset account book. "Well, what's
so difficult about designing a phone book?" was the immediate reaction. I tried
to explain, but the best I could do in the 2 minutes I had was to mumble
something about the fact that Offset is decentralized, transactions are
asynchronous, things might get cancelled, etc.  Obviously this was not enough
to convince my friend, but this got me thinking, maybe I need to write this all
down, to get a better understanding myself of what is so difficult with
designing Offset's account book.

# Motivation

A good start might be to explain why I want to have an account book in the
first place. The first version of Offset has a very simple mobile app that
allows payments using the Offset network. One repeating feedback I received
from users of Offset v1 was that once they received a payment successfuly,
there was no record showing that a transaction ever happend. This occured
because indeed Offset v1 did not save any record for any past transaction! 

Offset v1 saves only the final balance against every friend. The mobile app
goes one step forward and keeps track of succesful (sent) payments by keeping
a cryptographic receipt, but as this information is only saved in the mobile
app, it can not be synchronized between multiple devices that connect to the
same Offset node.

From the feedback I realized that people want to know their history of
transactions. As I tried to dig deeper I realized some of the reasons behind
this feature request. Some people wanted to obtain better understanding of
their own earnings or spendings. Others mentioned that they would like to have
neat records to help them do their taxes, having a convincing and auditable
account book.

In the spirit of the rest of Offset's design, this feature should never
compromise user's privacy. A user transactions' history will only be saved
inside his own node, and can be deleted if the user wanted to. There should
also be a way to cancel this feature altogether, for users that do not want to
ever remember past transactions.


# Requirements

The only account book I see regularly is my own account book at my bank's
website. It looks somewhat like this:


```
Date       | Balance |  Change | Description 
===========+=========+=========+==================
10.02.2021 | +20000  |   -100  | Supermarket
12.02.2021 | +29000  |  +9000  | Salary
13.02.2021 | +28500  |   -500  | Electricy bill
15.02.2021 | +28400  |   -100  | Water bill
```

Every row in the table describes a transaction that took place at a certain
date. Balance denotes the balance right after the transaction took place. So in
the table above: `20000 + 9000 = 29000`, and `29000 - 500 = 28500`. The change
column is either positive (if I received money), or negative (if I sent money).
The description column is a very short textual description of the transaction.

I would like to have something as similar as possible to this account book for
Offset. However, due to the nature of Offset having an exact copy of this
account book design might not be easily acheived. Hence, I would like to
somehow extract the core characteristics of this simple account book, and at
least somehow satisfy those characteristics.

One of the most important characteristic of the above simple account book I
would like to imitate is what I call the *locality principle*. Wherever you are
at the account book, you can audit the balance by looking the the previous
transaction's balance, add the change from the new transaction, and be able to
calculate the balance of the next row. This applies even if you are given a
partial account book, for example, only beginning from a certain date.

The other problem I would like the account book to solve is to somehow allow
a third party (Like a tax collecting entity) to somehow audit the account
book, or even cross audit account books of different people that traded with
each other. 

Finally, a note about privacy: Offset will allow the user to produce an account
book, but will never send this account book anywhere without explicit user's
request.

# Core Offset protocol

To better understand how to create a sound account book for Offset, some
understanding about how Offset works is required. I will try to outline the
main credit related parts, abstracting away all the technical details of low
level communication, encryption etc.

Offset allows trading on the basis of credit accounting. The basic component of
the Offset protocol is a **pair of friends**. Offset friends give certain
amount of credit to each other. A relationship between two Offset friends is
called mutual credit. 

![Mutual credit between Bob and Charli](./bob_charli_mutual_2.svg)

Offset lets any user pay another user along of chain of friendships. This
allows users to trade only using credit accounting, without using the
traditional money systems. The following two images show an example for payment
along a chain of Offset friendships, where Bob pays Dan through Charli.

Status before payment:

![Bob before Dan through Charli](./bob_charli_daniel_mutual.svg)

Status after payment:

![Bob after paying Dan through Charli](./bob_charli_daniel_mutual_paid.svg)

If Offset was built using a single centralized database, invoking such chain
transactions could have been trivial. However, Offset is a decentralized
system, and the mutual credit between any two users is only saved on the
devices of those two users. Therefore, performing a chain payment is a bit more
tricky. For example, what can we do if some user along the chain decides to not
forward the transaction, or maybe he becomes offline?

Offset attempts to solve this problem in a decentralized way using the core
Offset protocol. At its core, the **Offset protocol is made of three messages:
Request, Response and Cancel**. Those messages are sent between Offset friends.

Let's revisit the payment drawn earlier, this time with finer details of the
Offset protocol taking place:

![Offset protocol payment along a chain](./mutual_credit_payment_success.svg)


Order of events:

- Bob sends Charli a Request message.
- Charli sends Dan a Request message.
- Dan sends Charli a Response message.
- Charli sends Bob a Response message.

When a Request message is sent, credits are frozen in the mutual credit channel
between the two parties. For example, when Bob sent Charli a Request message,
the mutual credit channel between Bob and Charli has frozen 100 credits. Frozen
credits can not be used. They are waiting for the transaction to be completed
or cancelled. 

When a Respnose message is sent, the corresponding frozen credits are released
and transferred to the relevant party. For example, when Charli sent Bob the
last Response message, the 100 frozen credits were unfrozen, and the balance
between Bob and Charli changed to be +100 in favor of Charli.

Frozen credits are created by sending a Request message. Frozen credits can
only be unfrozen by:

- Sending a Response message. In this case the unfrozen credits are transferred
    to the party that received the original Request.
- Sending a Cancel message. In this case the frozen credits are erased.

Example for a Request being cancelled:

![Offset protocol Cancel example](./mutual_credit_payment_cancel.svg)

In the example above, Bob tried to send 170 credits all the way to Dan.
However, the mutual credit between Bob and Charli did not have sufficient
capcity (Because Charli has set up a credit limit of 150). Therefore instead of
forwardin the Request to Dan, Charli sends back a Cancel message to Bob. This
Cancel message has the effect of erasing the frozen credits. The balance
between Bob and Charli is left unchaged, and Bob is notified that the
transaction has failed.

Some rules to remember:

- Every Response or Cancel message must correspond to a previously sent Request
 in the opposite direction. The correspondence is done using a `request_id`
- It is not possible to return both a Response and Cancel messages for a
    Request message. It must be exclusively one of Response or Cancel, but not
    both.
- Every Request message is expected to eventually be countered by a Response or
    Cancel message. Otherwise, the credits are kept frozen and can not be used.


Essentially, the payment process can be summarized as two stages:

1. A Request is sent from the buyer to the seller, freezing the required
   credits along the chain.

2. A Response is sent from the seller all the way back to the buyer, claiming
   all the frozen credits.


Why are the credits collected only on the backwards Response message? Because
this gives the intermediate nodes the correct incentives to make the protocol
work.

Imagine for a moment that we only used a single Request message to push the
credits from the buyer to the seller. In this construction, a node along the
chain could keep the credits and never forward the Request to the next node.

In our construction, using Request and Response, this kind of attack is not
possible. A node in the middle of the chain only freezes credits during the
Request message phase. During the Response message phase, the intermediate node
first pays credits to the next node in the chain (When Response message is
received by the intermediate node), and only then receives more credits from
the previous node in the chain (When Response message is sent to the previous
intermediate node). Hence, forwarding the protocol messages correctly aligns
with the intermediate nodes along the chain to earn credits.


# Generalized mutual credit

We can now combine our previous discussion of balance and pending credits into
one diagram:

![Generalized mutual credit](./generalized_balance.svg)

This diagram shows the point of view of one node in a mutual credit channel.
The node keeps track of the current balance, and of all in-flight incoming and
outgoing Request messages. 

- For every outgoing (or forwarded) Request message, the local pending debt is increased. 
- For every incoming Request message, the remote pending debt is increased. 
- For every outgoing (or forwarded) Response message, the remote pending debt
    is decreased, and the balance is increased.
- For every incoming Response message, the local pending debt
    is decreased, and the balance is decreased.

Note that local pending debt can not be offset against remote pending debt,
because both represent frozen credits of in-flight Request messages. 


# Conflicts

Every pair of Offset friends maintain a mutual credit channel, where both
friends share an identical state, containing the current balance and in flight
Requests between the two friends. The state is identical up to direction and
sign, of course. For example, if one friend has a balance of +100, the other
friend will have balance of -100.

This is all fine in a nice theoretical world, but problems can, and probably
will, occur. Here are some examples:

1. One of the friends node's has a hardware issue, and his hard disk is damaged.
2. One friend can buy lots of things, have a large debt, close his node and run away.
3. One friend can maliciously change the internal state of his node, claiming
    that he has more credit.

To mitigate those issues, Offset uses two important mechanisms: Token Channel
based communication, and Conflicts resolving protocol.

## Token Channel

Every pair of Offset friends maintain a mutual credit. The communication
between two friends is done using a mechanism we call a Token Channel. A Token
channel allows only one friend at at time to send a message to the other
friend: the friend that holds the token. Whenever a message is sent, it
transfers the token from one friend to the other, allowing the other friend to
send a message. 

This mechanism has a few roles, the one important to our discussion is the role
of synchronization: The two friends can always agree on the order of messages
sent. 

This is how a message sent in a Token Channel looks like in Offset v2:

```rust
pub struct MoveToken {
    pub old_token: Signature,
    pub operations: Vec<FriendTcOp>,
    pub new_token: Signature,
}
```

`MoveToken` is mainly a container, that contains a batch of operations to be
made. Every operation is either Request, Response or Cancel. We batch the
operations together because of speed considerations. If we only sent one
operation at a time, we would have to wait for a whole roundtrip until we get
the token back.

Besides the operations, there are two more fields: `old_token` and `new_token`.
Those are cryptographic signatures over the state of mutual credit between the
two parties. Here, `old_token` is the old signature, and `new_token` is a
signature over a hash of the new state, and the previous `old_token`.

What are those signatures for? Recall case (2) mentioned above. If one friend
has a debt and runs away, the other friend can produce a digitally signed proof
of the latest mutual credit state between the two friends. This proof can then
be enforced using out of band means, for example, using a court.


## Conflict resolving

Conflicts resolve protocol is activated when two friends maintaining a mutual
credit channel do not agree on the shared state. Conflicts should almost never
happen, but they could, in cases like:

- Hardware issue for one of the friend's nodes.
- A friend maliciously changes his balance.

A conflict occurs when a `MoveToken` message arrives with an invalid signature,
or invalid contents. Some examples for invalid `MoveToken` contents:

- A Request message attempts to freeze credits, causing more than 2^128 credits
    to be frozen on the mutual credit channel.
- Response message has invalid signature field or plain lock field.
- Response or Cancel message has a `request_id` field that does not correspond
    to any previously sent Request message.

When any of the stated conditions is detected, a Conflict message is sent to
the remote friend, containing the local friend's reset terms. The reset terms
mostly contain the claimed balance for the mutual credit channel (and some
other stuff that are less relevant here). When the remote friend receives the
Conflict message, he also prepares his own reset terms and sends a
corresponding Conflict message to his friend.

At this stage, each of the friends can see the remote friend's reset terms, and
can decide whether he wants to accept them. The two friends are expected at
this point to communicate out of band (For example, by phone), try to find
out the reason for the conflict, and discuss a way to resolve it.

Whenever a friend accepts the remote friend's reset terms, the channel will be
reset, and the new balances will match the accepted reset terms balance.

There are a few more delicate details to be told about Offset conflicts. 
I am going to ignore most of them here, except for one which is important to
our consideration of account book: **We need to realize what happens to
in-flight Requests during an Offset Conflict**. Let me provide an example.

Consider the same Offset network configuration of Bob, Charli, Dan and Ellie.
Bob is sending a Request message all the way to Ellie. Charli and Dan are
intermediate Offset nodes in this transaction.

![Offset Conflict resolve example](./conflict.svg)

The Request arrives at Ellie, and right after Charli and Dan have a Conflict.
Charli immediately cancels all in-flight Requests it has: In this example, it
is only one Request, the Request sent from Bob to Ellie. Let's assume for
example that:

- Charli's reset terms are 0 balance (Also 0 from Dan's point of view).
- Dan's reset terms are +100 balance (-100 from Charli's point of view).

On Dan's side, the in-flight Request originally forwarded through Charli is
considered an "orphan Request", because a Conflict occured. Dan later receives
a Response from Ellie. Dan can not propagate this Response to Charli.
Therefore, Dan lost 100 credits when he received the response from Ellie, but
he could not recover his credits, due to the Conflict with Charli.

In the diagram above, we assumed that Charli accepted Dan's reset terms. As a
result, the final balance between Charli and Dan is -100 from Charli's point of
view, or +100 from Dan's point of view.

In the final state of the diagram, Charli's total balanace is -100, and Dan's
total balance is 0 (100 - 100). This means that Charli has lost credits because
of the Conflict. If Dan had accepted Charli's reset terms, Dan would have lost
credits due to this Conflict.

This example shows an important phenomenon regarding Conflicts and in-flight
Requests. When a Request is in progress and two friends have a Conflict:

- The friend who sent the Request will speculate that the Request will fail.
- The friend who received the Request will speculate that the Request will succeed.


In other words, when a conflict occurs, a node will not propose `balance` as
his reset terms. Instead, the node will propose `balance +
remote_pending_debt`, speculating that all the `remote_pending_debt` will
materialize into real credits in the future. See the generalized balance
diagram above to better understand the relationship between the two different
balances.


# Fees

Offset nodes that mediate payments might need incentives to do their work, for
a few reasons:

- Intermediate nodes take credit risk when they mediate transactions.
- Processing Offset protocol messages requires resources:
    - Live Internet connection,
    - Certain computation ability
    - Certain amount of hard disk space.

The core Offset protocol provides nodes means of collecting fees in return for
mediating Offset transactions through them. Every Request message, in addition
to the amount of credits being paid, contains an extra field called
`left_fees`. `left_fees` is being used like a jar of fees, given to the nodes
along the chain.

Every node can take as many credits that he wants from `left_fees`, update
`left_fees` accordingly, and then forward the Request message to the next node.
If a node receives a Request message with a too low `left_fees` value, he
cancels the Request by sending back a Cancel message.

How can the buyer of goods know how many fees credits he should leave to the
intermediate nodes? Every node has the responsibility of updating some index
server about his latest fees settings. Whenever a node asks for a chain from an
index server, the index server provides a chain, taking into consideration the
amount of fees that needs to added to the Request. The buyer can then decide
whether or not he agrees to pay those fees. If not, the transaction will fail,
or a different (cheaper) chain can be found to the seller.

An important observation here is that an Offset node can earn credits by
mediating other people's transactions.


# Payments protocol

The core Offset protocol described in the above section is underlying
everything that happens in Offset. However, on its own it is too basic to allow
what a usual user expects from a payments application. 

Offset has an extra protocol layer that manages higher level structures:
Invoices and Payments. In this document we call this protocol layer the
**payments protocol**. The payments protocol works above the Offset core
protocol, and it also requires some form of direct communication between the
buyer and the seller.

Whenever an Offset seller wants to request a payment, he creates an Invoice and
sends the Invoice to the buyer. The Invoice is sent out of band. The buyer then
creates a corresponding Payment to pay the Invoice. A Payment can either
succeed or fail. If the Payment succeeded, the buyer will get a signed receipt,
proving that the payment occured.

By saying **"out of band"**, I mean that the Offset network currently does not
provide inherent means of communication to send this Invoice to the buyer, and
you have to find your own way to do that. A few examples for sending an Invoice
out of band:

- Scanning a QR code 
- Sharing a file
- Sharing a link

There will always be something out of band when you buy or sell things. For
example, the goods will have to go out of band. When you visit the local
grocery store and buy a bag of tomatoes, Offset will not be able to transfer
the tomatoes for you. You will have to do it in the real physical world, out of
band.

To better understand the account book considerations, we need to go deeper to
how Invoices and Payments actually work in Offset. 

The following is an overview of the payment process:

1. The seller sends the buyer an Invoice (Out of band)
2. The buyer sends multiple Requests through separate chains of friends. The
   total of the credits pushed to the seller should sum up to the amount
   specified inside the original invoice. All the requests are locked using the
   same hash lock.
3. The buyer reveals the plain lock to the seller. At this point the
   payment is considered complete.
4. Buyer receives the goods (Out of band)
5. The seller sends corresponding Response-s for all sent Requests, effectively
   collecting the credits.

The following is a diagram demonstrating the main stages of payment. We omitted
(1) and (4) which are done out of band:

![Offset multi-route payment](./multiroute_payment.svg)

In the above diagram, **blue** arrow represent messages sent using the core
Offset protocol. **green** arrows represent messages sent using the **payments
protocol**. Note that the plain lock sent from the buyer to the seller is
sent along a relay, and not along a route of friends. This is done to provide a
sense of atomicity to the payment. The 

The events of selling (Invoice) or buying (Payment) are events that we would
like to have in our account book.


# Account book plan

There are a few factors that effect an Offset node's balance along time:

- Invoices (Incoming payments)
- Payments (Outgoing payments)
- fees
- Friend events

We already mentioned the first two (Invoices and Payments) in the previous
section.

TODO




