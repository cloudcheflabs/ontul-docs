# Encryption & KMS

Ontul provides built-in encryption with a Key Management Service (KMS) to protect sensitive data at rest across the cluster.

## Envelope Encryption

Ontul uses envelope encryption — each piece of sensitive data is encrypted with its own Data Encryption Key (DEK), and DEKs are encrypted with a master key:

- **Algorithm**: AES-256-GCM
- **Master Key Derivation**: PBKDF2-SHA256 with 200,000 iterations
- **Master Key Source**: Environment variable `ONTUL_MASTER_KEY` (minimum 32 characters)

## What Is Encrypted

- **Connection credentials**: S3 access keys, JDBC passwords, Kafka credentials stored in the ConnectionStore
- **IAM secrets**: User passwords, access key secrets, STS tokens
- **Cluster state metadata**: Sensitive metadata in the RocksDB state store

## Built-in KMS

Ontul includes a built-in KMS that:

- Generates and manages encryption keys in a RocksDB-backed keystore
- Distributes keys from the leader Master to all cluster nodes automatically
- Replicates the encrypted keystore to follower Masters for high availability

No external KMS service is required.

## Key Distribution

1. The leader Master generates and stores DEKs in the local encrypted RocksDB keystore
2. On leader election or key changes, the keystore is replicated to follower Masters via the internal NIO protocol
3. Workers receive relevant keys for decrypting connection credentials needed during query execution

## TLS

Ontul supports optional TLS for external-facing endpoints:

- **Arrow Flight SQL**: Encrypted JDBC and SDK connections
- **Admin HTTP**: HTTPS for the Admin UI and REST API
