# Iceberg Write-Audit-Publish (WAP)

Iceberg Write-Audit-Publish lets you stage writes on a non-`main` branch, **audit** that branch in
isolation, then atomically **publish** it to `main` by fast-forwarding — readers see `main` the whole
time. In Ontul, the write branch is selected per table (SQL session property, streaming sink config,
or catalog option), and publishing is an `ALTER TABLE … EXECUTE fast_forward` / `cherrypick`.

Full documentation, configuration keys, and end-to-end Java/Python examples are in the dedicated
**[Iceberg Integration → Write-Audit-Publish (WAP)](../iceberg/wap.md)** page.
