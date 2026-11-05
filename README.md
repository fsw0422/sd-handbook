# System Design Handbook

System design study notes, with explanations, examples, diagrams, and recall questions.

## Designing Data-Intensive Applications

Chapter summaries of Martin Kleppmann's *Designing Data-Intensive Applications*, first edition (O'Reilly, 2017). Each summary includes key ideas, trade-offs, references to printed page numbers, Mermaid diagrams, and recall questions with expandable answers.

These are study notes. The book and its PDF are not included.

| Chapter | Summary |
|---|---|
| 1 | [Reliable, Scalable, and Maintainable Applications](ddia/01-reliable-scalable-and-maintainable-applications.md) |
| 2 | [Data Models and Query Languages](ddia/02-data-models-and-query-languages.md) |
| 3 | [Storage and Retrieval](ddia/03-storage-and-retrieval.md) |
| 4 | [Encoding and Evolution](ddia/04-encoding-and-evolution.md) |
| 5 | [Replication](ddia/05-replication.md) |
| 6 | [Partitioning](ddia/06-partitioning.md) |
| 7 | [Transactions](ddia/07-transactions.md) |
| 8 | [The Trouble with Distributed Systems](ddia/08-the-trouble-with-distributed-systems.md) |
| 9 | [Consistency and Consensus](ddia/09-consistency-and-consensus.md) |
| 10 | [Batch Processing](ddia/10-batch-processing.md) |
| 11 | [Stream Processing](ddia/11-stream-processing.md) |
| 12 | [The Future of Data Systems](ddia/12-the-future-of-data-systems.md) |

### Companion References

- [Isolation Level Cheat Sheet](cheatsheets/isolation-level.md): anomaly tables and transaction timelines.
- [Kafka Cheat Sheet](cheatsheets/kafka.md): brokers, partitions, consumer groups, idempotency, and database effects with diagrams.
- [Kubernetes Cheat Sheet](cheatsheets/k8s.md): core resources and diagrams, graceful shutdown, service meshes, sidecars, and rollout troubleshooting.
- [DynamoDB Cheat Sheet](cheatsheets/dynamodb.md): keys, indexes, single-table design, capacity, Global Tables, DAX, and DynamoDB Streams versus Kinesis.
- [gRPC and HTTP/2 Cheat Sheet](cheatsheets/grpc.md): Protobuf, multiplexing, channels, deadlines, retries, keepalive, and observability.
- [Service Boundaries and Testing Cheat Sheet](cheatsheets/testing.md): domain ownership, ports and adapters, test boundaries, and gradual module extraction.
- [OAuth Cheat Sheet](cheatsheets/oauth.md): how OAuth works with Google as the example, application-only access, tokens and client types, the code-interception attack PKCE stops, OIDC sign-in, and session binding.
- [JWT and Distributed Sessions Cheat Sheet](cheatsheets/distributed-sessions.md): authorization claims, session storage, token validation, caching, and revocation.
- [SSO, SAML, and SCIM Cheat Sheet](cheatsheets/sso-saml-scim.md): how SAML login works, validating assertions, SP- versus IdP-initiated login, domain verification, JIT versus SCIM provisioning, and the SCIM API.
- [HTTP and TLS Cheat Sheet](cheatsheets/tls-docsign.md): HTTP and HTTPS, cryptographic keys, key agreement, CA certificates, TLS handshakes, and how electronic signatures sign and verify documents.
