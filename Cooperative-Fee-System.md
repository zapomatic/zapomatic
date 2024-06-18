# Lightning Routing Node Cooperative Fee System

> A non-zero-sum game-theory model for routing fees in the Lightning Network

Choosing the right fees for channels on the lightning network is a difficult task that is often done poorly. Many nodes rely on fee automation software to bring fees from a high point (e.g. 2000 PPM) to a low point (e.g. 0 PPM), stopping when traffic appears and hovering around a theoretical optimal fee rate for the duration of the channel. This strategy has several flaws:

1. fee rates for a given channel depend largely on the ever-changing state of neighboring nodes, rates of peer channels, liquidity availability, and opening/closing of channels.
2. low fees are abused by rebalancers
3. automated fee rates can be manipulated by immediate peers to achieve liquidity theft at cheaper rates
4. PeerSwap and other submarine swaps disrupt and confuse fee automation

Large routing nodes on the network have discovered ways to pray on the poor fee policy of smaller nodes, creating a zero-sum winning model for themselves on the network. An example of this strategy is the Vampiric strategy:

1. Open large bandwidth channels to major traffic nodes (BFX, FixedFloat, Kraken, etc) and set high fees (3000 PPM - 3500 PPM)
2. Have a default static fee for ALL other channels at 2500 PPM
3. Use all other channels to rebalance high value channels at less than 2500 PPM -- this chokes out your neighboring peers of liquidity on your higher priced alpha channels.
4. When a useful routing node connect to you, pull 95% of the channel capacity to your side via cheap rebalance and use the channel only for egrees at 2500 PPM.

## High Routing Fees Discourage Rebalancers

## Permanent High Routing Fees

## Charge-lnd Rules

```

```
