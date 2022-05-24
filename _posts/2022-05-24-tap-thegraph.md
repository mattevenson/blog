---
layout: post
title: "Building blockchain data pipelines with The Graph & Singer"
---

[The Graph](https://thegraph.com/en/) makes lots of structured blockchain data available via GraphQL APIs called subgraphs. Almost every major DeFi protocol has a corresponding subgraph. For example, you can query [redeem events on Compound v2](https://thegraph.com/hosted-service/subgraph/graphprotocol/compound-v2?query=cDAI%20Transfers) or [Squeeth vaults](https://thegraph.com/explorersubgraph?id=Ao1QSKEQzsnNyyGKR1Faurjmkr6oNVTbgdxy6diAw9r&view=Playground).

Inspired by services like [Dune Analytics](https://dune.com/browse/dashboards) and [Flipside Crypto](https://app.flipsidecrypto.com/velocity?nav=Discover), I thought it would be fun to extract data from these subgraphs and load it into a SQL database for further analysis. It turns out that the folks at Decentraland had already done this for Decentraland's subgraph and [open-sourced their work](https://github.com/decentraland/tap-decentraland-thegraph). 

Under the covers, they used [Singer](https://www.singer.io/), which is an open source spec for writing data extraction scripts (taps) and data loading scripts (targets). Taps and targets communicate by passing JSON over Unix pipes, with [JSON Schema](https://json-schema.org/draft/2020-12/json-schema-core.html) for typing. The idea is that any tap is compatible with any target, so you don't need to roll your own script if one already exists for your source / destination!

Singer sounded like the perfect tool for the job, so I went ahead and wrote a tap to extract data from any subgraph: [`tap-thegraph`](https://github.com/superkeyio/tap-thegraph). You configure the subgraphs and entities you care about and you're done! Now you can load entire subgraphs into [any destination with a Singer target](https://hub.meltano.com/singer/targets/): Postgres, Snowflake, SQLite, CSV, etc.

For example, here's how to export all markets on Compound v2 to a SQLite database:


```bash
# 1. Install our packages for extracting subgraph data.
npm install -g graphql-api-to-json-schema
pipx install git+https://github.com/superkeyio/tap-thegraph.git

# 2. Install a Singer target for loading the data to a destination (for example, SQLite).
pipx install target-sqlite

# 3. Configure which subgraphs and entities to extract (for example, all markets on Compound V2).
echo "{\"subgraphs\":[{\"url\":\"https://api.thegraph.com/subgraphs/name/graphprotocol/compound-v2\",\"entities\":[{\"name\":\"Market\"}]}]}" >> tap.json

# 4. Configure the database name.
echo "{\"database\": \"compound.db\" }" >> target.json

# 4. Run the pipeline!
tap-thegraph --config tap.json | target-sqlite --config target.json
```


For more power, you can use [Meltano](https://docs.meltano.com/getting-started) to build robust data pipelines on top of Singer taps and targets.

### How it works

The tap treats each entity within a subgraph as its own table (or stream, in Singer terms). 

In order to figure out the schema for an entity, the tap makes an introspection query to the GraphQL API and then converts the GraphQL schema to JSON schema. Unfortunately, the only library I could find to do this conversion was in Javascript, and the Singer tap SDK was in Python, so [I hacked together a CLI](https://github.com/superkeyio/graphql-api-to-json-schema) using [`oclif`](https://oclif.io/) and published it as an npm package so I could call it with `subprocess`.

In subgraphs, it is possible for entities to have one-to-many, many-to-one, or many-to-many relationships with other entities. To simplify things and eliminate the possiblity of cycles, we normalize by replacing all associations with the ID of the associated entity. 

The actual GraphQL query is a gnarly mess of string formatting and GraphQL variables to account for pagination and normalization:

{% raw %}
```python
    @property
    def query(self) -> str:
        newline = "\n\t"
        return f"""
query($batchSize: Int!{ f', $latestOrderValue: {self.order_attribute_type}!' if self._latest_order_attribute_value else '' }) {{
    {self.query_type}(first: $batchSize, orderBy: {self.order_attribute}, orderDirection: asc{f', where: {{ {self.order_attribute}_gt: $latestOrderValue }}' if self._latest_order_attribute_value else ''}) {{
        {newline.join(self.query_fields)}
    }}
}}
"""
```
{% endraw %}

Fortunately, much of the plumbing involved in the Singer spec was taken care of by [Meltano's SDK for taps](https://sdk.meltano.com/en/latest/).

And that's basically it. If you have questions or are also interested in blockchain data engineering, I'd love to chat!










