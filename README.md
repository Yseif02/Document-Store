# Document Store

A Java document store with ranked text search, metadata queries, undo, and disk-backed memory management. It brings together custom implementations of a B-tree, trie, min-heap, and stack behind one API.

Documents are identified by a `URI` and can contain either plain text or binary data. The store keeps frequently used documents in memory and moves least recently used documents to disk when a configured limit is reached.

## Features

- **Text and binary storage:** Insert, retrieve, replace, and delete documents by URI. URIs are identifiers; inserting a document does not fetch content from the web.
- **Ranked search:** Find text documents by keyword or word prefix. Keyword results are ordered by occurrence count; prefix results are ordered by the total number of matching word occurrences. Text search is case-sensitive.
- **Metadata:** Attach string key-value pairs to documents and query by metadata, alone or combined with keyword or prefix search.
- **Bulk deletion:** Delete documents matching a keyword, prefix, metadata query, or combined query.
- **Undo:** Reverse the latest mutation across the store, or the latest mutation for a particular URI. Command sets track grouped operations such as bulk deletion.
- **Memory limits:** Configure the maximum number of resident documents or the total bytes of their text/binary content. A min-heap tracks last-use timestamps for least recently used eviction.
- **Disk storage:** Serialize evicted documents as JSON with Gson, including metadata and word counts. Binary content is Base64-encoded. Documents are loaded back into memory when accessed.

## How it works

| Component | Responsibility |
| --- | --- |
| `DocumentStoreImpl` | Coordinates the public API, indexes, undo history, and memory limits. |
| `DocumentImpl` | Represents text or binary content, metadata, word counts, and last-use time. |
| `BTreeImpl` | Maps document URIs to documents and supports moving values to disk through a persistence manager. |
| `TrieImpl` | Indexes text-document URIs by word for keyword and prefix lookup. |
| `MinHeapImpl` | Orders resident documents by last use to select eviction candidates. |
| `StackImpl` | Maintains undo history through `GenericCommand` and `CommandSet`. |
| `DocumentPersistenceManager` | Writes and reads JSON documents beneath a configurable base directory. |

Retrieving or searching for a document updates its last-use time. When a memory limit is exceeded, the store removes the oldest document from the heap and serializes it to disk. The document remains part of the store and can be retrieved again by URI.

Disk storage acts as overflow for the running store. The implementation does not rebuild its B-tree and search indexes from an existing directory when a new store instance starts.

## Build

Requirements:

- JDK 17 or later; the Maven compiler configuration targets Java 17.
- Maven.

From the repository root:

```bash
cd Stage6
mvn -Dmaven.test.skip=true package
```

The implementation is a Java library used through the `DocumentStore` interface. The repository does not include a standalone application entry point.

## Usage

```java
import edu.yu.cs.com1320.project.stage6.DocumentStore;
import edu.yu.cs.com1320.project.stage6.impl.DocumentStoreImpl;

import java.io.ByteArrayInputStream;
import java.net.URI;
import java.nio.charset.StandardCharsets;
import java.nio.file.Files;
import java.nio.file.Path;
import java.util.Map;

public class DocumentStoreExample {
    public static void main(String[] args) throws Exception {
        Path directory = Files.createDirectories(Path.of("store-data"));
        DocumentStore store = new DocumentStoreImpl(directory.toFile());
        URI uri = URI.create("http://example.com/notes/storage");

        byte[] text = "Java storage systems use indexes. Java documents can be searched."
                .getBytes(StandardCharsets.UTF_8);

        store.put(new ByteArrayInputStream(text), uri, DocumentStore.DocumentFormat.TXT);
        store.setMetadata(uri, "category", "notes");

        System.out.println(store.get(uri).getDocumentTxt());
        System.out.println("Keyword matches: " + store.search("Java").size());
        System.out.println("Prefix matches: " + store.searchByPrefix("stor").size());
        System.out.println("Metadata matches: "
                + store.searchByMetadata(Map.of("category", "notes")).size());

        store.delete(uri);
        store.undo(uri); // Restore the deleted document.

        store.setMaxDocumentCount(10);
        store.setMaxDocumentBytes(1_000_000);
    }
}
```

Use `DocumentFormat.BINARY` with a binary input stream to store bytes. Binary documents can be retrieved and queried by metadata, but are not included in text search.

The persistence base directory must already exist. The no-argument constructor uses the current working directory. Serialized file paths are derived from document URIs beneath that directory.

In the current implementation, metadata queries compare the supplied map with the document's complete metadata map. Supplying only a subset of a document's metadata will not match it.

## Tests

`DocumentStoreImplTest` contains JUnit tests for storage and replacement, search, metadata, deletion and undo, and moving documents between memory and disk.

The checked-in test setup and some fixture helpers use absolute Windows paths. Before running the suite on another machine, change those paths to writable directories that exist locally; this does not require changing the document-store implementation.

Then, from `Stage6`, run:

```bash
mvn test
```

## Repository layout

```text
Stage6/
├── pom.xml
└── src/
    ├── main/java/edu/yu/cs/com1320/project/
    │   ├── impl/          # Custom B-tree, trie, heap, and stack
    │   ├── stage6/        # Document, DocumentStore, and persistence interfaces
    │   │   └── impl/      # Store, document, and persistence implementations
    │   └── undo/          # Undo commands and command sets
    └── test/java/edu/yu/cs/com1320/project/
        └── stage6/impl/   # Document-store tests
```

The repository contains the final Stage 6 implementation, combining the project's storage, search, undo, memory-management, and persistence work.
