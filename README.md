
# secure-channel

This is a project to design a secure-channel (encrypted duplex connection)

# design process

This protocol will be developed in an audit-first style.
The first phase was to study and summarize [prior art](./prior-art.md),
then to summarize the [abstract properties](./properties.md) of secure channels.

And then create draft proposals ([draft 1](./draft.md), [draft 2](./draft2.md)).

I have not started to write code yet.

# contributing

Discussions and questions are welcome in the issues board.
Decisions are not final until a PR updating the documents has been merged!

PRs should summarize the decision sufficiently so that you do not need to read the issues board unless you want to.
This includes summarizing choices which were discarded, and for what reasons.
Be thorough!
If something is not in the documentation, it has not been decided.

# Protocol Assumptions

I assume an underlying reliable connection oriented transport.
and that at least the client knows the public key of the server
before they establish a connection to them.
(it is assumed that the `{network_address, public_key}` pairing
is pre-established via another channel)

# License

MI
