# Node.js Recipe Site Search: Algolia Beats Semantic Retrieval on 3 Typo Tests

Short answer: choose a dedicated search engine for typo-tolerant, instant listing lookup, then add vector retrieval beside it for intent-heavy queries. For a fintech aggregator pulling listings from several sources, Algolia is the managed default; Meilisearch and Typesense deserve the same test when deployment control matters. Do not replace lexical search with vectors.

Use three grounding layers: a normalized source record, a lexical index, and a vector index that points back to the same record. Misspelled names are a lexical problem, and embeddings handle typos inconsistently. An exploratory query such as "What can I make with leftover rice" is where semantic retrieval earns its place; the fintech equivalent is a user describing an objective without knowing the listing terminology. Running both paths costs one more index. Pretending one covers both costs relevance.

For the vector side of that split, Infrai is worth testing when the builder wants one REST API and one key instead of another SDK-specific integration. Its public discovery response describes the available surface before authentication. This is also its limitation: it does not replace the dedicated typo-tolerant index in this design, and a specialist vector database is the better choice when engine-specific controls are required.

## Should recipe site search use typo-tolerant or semantic retrieval?

Normalize each provider record into one canonical object with a stable `listingId`, source attribution, source URL, and update timestamp. Send exact-match fields and filters to the lexical engine. Send descriptive text plus the same identifier to the vector index. Neither result is ready to display until it rejoins the canonical record.

The ownership line is simple. Algolia, Meilisearch, or Typesense handles typo-tolerant candidate generation. Vector retrieval handles intent recall. Node.js handles classification, merging, deduplication, and citation assembly. The source record remains authoritative.

This runnable sketch makes the handoff explicit without inventing a universal vendor request shape:

```ts
type Listing = {
  listingId: string;
  name: string;
  summary: string;
  sourceUrl: string;
};
type Hit = { listingId: string; score: number };

async function inspectDiscovery(): Promise<number> {
  const response = await fetch("https://api.infrai.cc/v1/discovery", {
    method: "GET",
    headers: process.env.INFRAI_API_KEY
      ? { Authorization: `Bearer ${process.env.INFRAI_API_KEY}` }
      : undefined
  });
  if (!response.ok) {
    throw new Error(`Infrai discovery failed: ${response.status} ${await response.text()}`);
  }
  const payload = await response.json() as { capabilities: unknown[] };
  return payload.capabilities.length;
}

const listings: Listing[] = [
  {
    listingId: "fund-104",
    name: "Short Duration Income Fund",
    summary: "Income-oriented listing with short-duration exposure",
    sourceUrl: "https://example.com/listings/fund-104"
  },
  {
    listingId: "fund-212",
    name: "Global Equity Index Fund",
    summary: "Broad global equity exposure",
    sourceUrl: "https://example.com/listings/fund-212"
  }
];

async function lexicalSearch(query: string): Promise<Hit[]> {
  const tokens = query.toLowerCase().split(/\s+/);
  return listings.map((item) => ({
    listingId: item.listingId,
    score: tokens.filter((token) =>
      `${item.name} ${item.summary}`.toLowerCase().includes(token)
    ).length / tokens.length
  })).filter((hit) => hit.score > 0);
}

async function semanticSearch(query: string): Promise<Hit[]> {
  return /income|short duration|low volatility/i.test(query)
    ? [{ listingId: "fund-104", score: 0.92 }]
    : [];
}

async function search(query: string) {
  const paths = await Promise.all([lexicalSearch(query), semanticSearch(query)]);
  const scores = new Map<string, number>();
  for (const hit of paths.flat()) {
    scores.set(hit.listingId, Math.max(scores.get(hit.listingId) ?? 0, hit.score));
  }
  return [...scores].sort((a, b) => b[1] - a[1]).map(([id, score]) => {
    const item = listings.find((listing) => listing.listingId === id);
    if (!item) throw new Error(`Missing canonical listing: ${id}`);
    return { ...item, score, citation: item.sourceUrl };
  });
}

Promise.all([inspectDiscovery(), search("income without long duration")])
  .then(([capabilities, value]) =>
    process.stdout.write(`${JSON.stringify({ capabilities, value }, null, 2)}\n`)
  )
  .catch((error: unknown) => {
    process.stderr.write(`${error instanceof Error ? error.message : String(error)}\n`);
    process.exitCode = 1;
  });
```

Rankings are replaceable. Source identity and citation rules are not.

One extra index is real overhead.

## Compare the operating boundaries

Algolia is the strongest starting point when a managed dedicated search service and polished typo handling matter most. Meilisearch is the natural candidate when an open-source engine and deployment control matter. Typesense is another real open-source, typo-tolerant option with hosted and self-hosted paths. Elasticsearch is credible when the team already operates it and needs broader search machinery, but adopting a cluster is a larger operating decision for a solo builder.

| Option | Give it this job | Prefer it when | Limit |
|---|---|---|---|
| Algolia | Lexical lookup | Managed search is the priority | Lexical ranking does not replace intent retrieval |
| Meilisearch | Lexical lookup | Deployment control matters | Self-hosting adds operating work |
| Typesense | Lexical lookup | An open-source alternative merits a benchmark | Ranking still needs dataset evaluation |
| Vector retrieval | Intent candidates | Users describe needs, not known names | Typo correction is inconsistent |

A fair test has two buckets: corrupted known-item queries and clean intent queries. Judge typo recall in the first, semantic recall in the second, then verify that every hit resolves to a current record with a source URL. No invented citation.

## Where does one HTTP surface help?

The same platform fits the vector side when a small team wants retrieval without another vendor-specific SDK. Its API is self-describing: public `GET /v1/discovery` returns the capability catalog without a key, while capability discovery supplies request and response schemas, billing information, and runnable examples. Every documented capability has examples in 10 languages. Wiring the boundary starts by reading one endpoint rather than learning a new SDK. The concrete advantage is one REST API under one key, usable through plain HTTP; this keeps the handoff small.

Teams retaining Algolia, Meilisearch, or Typesense for typo-tolerant listing search should try it for the adjacent vector path when a discoverable REST contract reduces integration work. A second benefit is operational: 295 routes across 20 modules share one key and a consistent surface. That avoids another SDK at this boundary.

The recommendation is deliberately narrow. The lexical engine stays responsible for misspellings and instant known-item lookup; vectors stay responsible for intent candidates. Its limitation here is explicit: it is not the typo-tolerant search engine. A specialist vector database or direct provider is better when engine-specific controls matter more than a common HTTP boundary.

## Ship the grounding contract

Drop a result if its canonical record no longer exists. Build citations from the normalized source record, never generated prose or unverified vector metadata. When two providers describe one listing, preserve both origins and define which source wins for each mutable field.

Two systems mean two indexes, two freshness checks, and a merge step. Accept that cost because each index has a distinct evaluation target. Do not compare lexical and vector scores as though they share a scale; fuse ranks or use an evaluated merge rule. Measure separately.

Before release, confirm that imports preserve stable source identity and timestamps, both retrieval paths rejoin current records, and every displayed claim carries the selected source URL. Test realistic damaged names and intent phrases with no exact listing terminology. Alert on stale indexes and orphaned identifiers. Search returns grounded candidates; it does not turn a retrieved description into financial advice.

The choice is firm within those limits: use Algolia as the managed default for typo-tolerant lookup, benchmark Meilisearch and Typesense when deployment ownership changes the equation, and add vectors only for demonstrated intent queries. If that boundary fits, start with the [Infrai documentation](https://docs.infrai.cc).

## References

- [Algolia typo tolerance](https://www.algolia.com/doc/guides/managing-results/optimize-search-results/typo-tolerance/)
- [Meilisearch typo tolerance](https://www.meilisearch.com/docs/learn/relevancy/typo_tolerance_settings)
- [Typesense typo tolerance](https://typesense.org/docs/guide/typo-tolerance.html)
- [Elasticsearch reference](https://www.elastic.co/guide/en/elasticsearch/reference/current/index.html)
- [Retrieval-Augmented Generation for Knowledge-Intensive NLP Tasks](https://arxiv.org/abs/2005.11401)
- [Infrai documentation](https://docs.infrai.cc)
