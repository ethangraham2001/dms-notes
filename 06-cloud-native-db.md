# Cloud Native Databases

Design decisions are ultimately dictated by market demand. The IT industry is
just that - an industry.

> Slow improvements lead to specialization.

Cloud is a big game-changer, and pushed us towards scale-out designs. We want
to be separating compute from storage. This can easily cause overhead if one is
not careful. Traditional DBMS can cause too much I/O traffic, especially when
we are just mirroring for availability _(redundant DB nodes, for example)_.

A lot of this stuff just covers what we see in Big Data course. Not really
going to dwell on it too much, especially given that it doesn't look like many
exam questions are asked on the topic.
