# Joel's software notes

What I learn about how software works underneath, written down so I can come back to it. Every topic is explained in small blocks, with hand-drawn diagrams, and ends with the takeaways and the questions it should let me answer in an interview.

## Categories

| Category | What's inside |
|---|---|
| [Database internals](database-internals/README.md) | WAL, replication, storage engines, MVCC, CDC and Kafka, online schema migrations |

## How each topic is organized

```
category/
└── NN-topic/
    ├── README.md        the notes, in reading order
    └── diagrams/
        ├── NN-name.excalidraw   editable source
        └── NN-name.png          exported image, shown in the README
```

Every topic README follows the same shape:

1. **What you'll learn**, in a few bullets.
2. **Before reading**: which topics to read first.
3. **The notes**, one section per diagram.
4. **Key takeaways**: the ideas worth remembering if everything else is forgotten.
5. **Interview cheat sheet**: questions and short answers.
6. **Further reading**.

New topics start from [TEMPLATE.md](TEMPLATE.md).

## Diagrams

The diagrams are drawn in [Excalidraw](https://excalidraw.com). GitHub can't render `.excalidraw` files, so each one is exported to PNG next to its source with the same name. To edit one, open the `.excalidraw` file in Excalidraw (or the VS Code extension), change it and export it again.
