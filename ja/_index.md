---
title: アプリ開発ガイダンス
summary: Application development guides for developers based on TiDB.
---

# App Development Guides

TiDB supports the MySQL protocol and most of the MySQL features. In general, to develop your application based on TiDB, you  can use either native driver/connectors or popular third-party frameworks like ORM and migration tools.

This series of documents are app development guides that show you how to build simple applications based on TiDB.

<NavColumn>
<ColumnTitle>TiDBデータベース開発規則</ColumnTitle>

- [まえがき](tidb-database-development-specification/introduction.md)
- [オブジェクト命名規則](tidb-database-development-specification/object-naming-guidelines.md)
- [データベースオブジェクト設計](tidb-database-development-specification/database-object-design.md)
- [データモデル設計](tidb-database-development-specification/database-model-design.md)
- [SQL開発規則](tidb-database-development-specification/sql-development-specification.md)
- [トランザクション制限](tidb-database-development-specification/transaction-restraints.md)
- [暗黙の型変換](tidb-database-development-specification/implicit-type-conversion.md)
- [結果セットの不安定さ](tidb-database-development-specification/unstable-result-set.md)
- [インデックス使用上の注意](tidb-database-development-specification/notes-on-indexes.md)
- [自動インクリメント列使用上の注意](tidb-database-development-specification/notes-on-auto-increment-columns.md)
- [TiDBの各種タイムアウト](tidb-database-development-specification/timeouts-in-tidb.md)
- [JDBCベストプラクティス](tidb-database-development-specification/jdbc-best-practices.md)
- [ホットスポットの軽減](tidb-database-development-specification/mitigation-of-hot-issues.md)
- [ページングのベストプラクティス](tidb-database-development-specification/best-practices-for-paging.md)
- [固有のシーケンス番号生成方法](tidb-database-development-specification/unique-serial-number-generation-scheme.md)
- [プロセス規則](tidb-database-development-specification/process-specification.md)

</NavColumn>