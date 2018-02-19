## Problem

We want a peer to prove that they "know" a block of data. By "know" we mean that
they somehow have access to the data. We cannot guarantee that they have a local
physical copy, but by playing this game, we can ensure that they can quickly
access the data. Their best long-term play is to have the data stored locally,
when questioned over a long period of time for random blocks.

## Properties

This scheme utilises the fact that Blake2 can be used as a prefix MAC. The way
it works is that you prefix the hash computation with a random nonce and public
key. Because the nonce is at the beginning, you cannot precompute any
challenges, and because the public key is mixed in, you cannot reuse a solution
from another peer, so either the peer has the data available or several peers
have conspired together.

You send off a random nonce and a block index, and wait for the peer to respond
within a reasonable time-frame, which depends on your block size and latency.
When the peer responds you check the signature by computing the solution
yourself, and react to whether the peer solved the challenge correctly, eg. by
flagging a bad peer.

A challenge cannot be forwarded since the solution depends on the peer locally
mixing their public key, so a peer acting in good faith would not compute a
challenge with another peer's public key.


### Init

1. Retrieve a public key from a peer.
2. Check that two peers do not share a public key, otherwise flag both

### Challenge

1. Send `{index, nonce}`, where `index` is a random block we believe the peer
   has and `nonce` is random bytes
2. Compute `solution = Blake2b(nonce, PK || data)` where `Blake2b(key, data)`
3. Compute `signature = EdDSASign(SK, solution)`, where `EdDSASign(key, data)`
3. Reply `{signature}`

### Verify

1. Compute `localSolution = Blake2b(nonce, peerPK || data)`
2. Check `EdDSAVerify(peerPK, localSolution, peerSignature)`.

### Issues

* The interrogating peer always needs to have access, somehow, to the data
  that it is challenging, or have precomputed challenges for peers.
* Several peers conspire together to solve challenges on each others behalf.
  They do this by forwarding challenges to the peer that has the copy, and use
  the forwarder's public key. This means they can lead a honest peer to believe
  that N peers have the data / there are N copies of the data.
  In fact M peers can get away with 1 copy divided into N / M pieces
  This is attempted mitigated by using signatures and mixing in the Public Key
  of a peer
* Making a peer's public key valuable. If the public key that a peer uses has no
  value, there's no cost in sharing it with multiple peers, making the above
  attack risk free. Somehow trust needs to built up around the public key and in
  a way that sharing the secret key would be devastating to the peer.
* A peer may fetch the data as they are requested to solve a challenge. This may
  be possible to mitigate by setting a low timeout, but with the risk of failing
  a peer due to high latency
* Correlate asks and challenges and dissallow `download` (only `upload`)
