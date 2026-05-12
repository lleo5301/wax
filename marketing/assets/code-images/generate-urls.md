# Code Image URLs (Carbon fallback)

Since Silicon is not available, use these Carbon URLs for code screenshots:

## Snippet 1 — Basic Memory API

```
https://carbon.now.sh/?l=swift&t=dracula&bg=rgba(10%2C10%2C10%2C1)&code=%2F%2F%20Persistent%20memory%20for%20AI%20agents%0Alet%20memory%20%3D%20try%20await%20Memory(at%3A%20url)%0A%0A%2F%2F%20Save%20a%20memory%0Atry%20await%20memory.save(%22User%20prefers%20dark%20mode%22)%0A%0A%2F%2F%20Hybrid%20search%20(text%20%2B%20vector)%0Alet%20results%20%3D%20try%20await%20memory.search(%0A%20%20%22What%20does%20the%20user%20prefer%3F%22%0A)
```

## Snippet 2 — Structured Memory

```
https://carbon.now.sh/?l=swift&t=dracula&bg=rgba(10%2C10%2C10%2C1)&code=%2F%2F%20Entity-Attribute-Value%20with%20temporal%20validity%0Aawait%20memory.upsertEntity(%0A%20%20key%3A%20%22user%22%2C%0A%20%20kind%3A%20%22person%22%0A)%0A%0Aawait%20memory.assertFact(%0A%20%20subject%3A%20%22user%22%2C%0A%20%20predicate%3A%20%22prefers%22%2C%0A%20%20object%3A%20%22dark%20mode%22%2C%0A%20%20valid%3A%20.init(fromMs%3A%20now)%0A)
```

## Snippet 3 — WAL Ring Buffer (from codebase)

```
https://carbon.now.sh/?l=swift&t=dracula&bg=rgba(10%2C10%2C10%2C1)&code=%2F%2F%20WAL%20ring%20buffer%20with%20wraparound%0Aprivate%20func%20append(payload%3A%20Data)%20throws%20-%3E%20UInt64%20%7B%0A%20%20let%20entrySize%20%3D%20headerSize%20%2B%20payload.count%0A%20%20%0A%20%20%2F%2F%20Handle%20wraparound%20with%20padding%0A%20%20if%20walSize%20-%20writePos%20%3C%20entrySize%20%7B%0A%20%20%20%20let%20padding%20%3D%20WALRecord.padding(%0A%20%20%20%20%20%20sequence%3A%20lastSequence%20%2B%201%2C%0A%20%20%20%20%20%20skipBytes%3A%20walSize%20-%20writePos%20-%20headerSize%0A%20%20%20%20)%0A%20%20%20%20try%20file.writeAll(padding.encode())%0A%20%20%20%20writePos%20%3D%200%0A%20%20%7D%0A%20%20%0A%20%20%2F%2F%20Write%20record%0A%20%20let%20record%20%3D%20WALRecord.data(...)%0A%20%20try%20file.writeAll(record.encode())%0A%20%20writePos%20%2B%3D%20entrySize%0A%7D
```

---

## Instructions

1. Click each URL to open Carbon
2. Click "Export" → PNG
3. Save to `code-images/` folder with appropriate name

Alternative: Use browser automation via Playwright to download.
