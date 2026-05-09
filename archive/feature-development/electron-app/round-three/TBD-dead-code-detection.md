# TODO

This is a new feature that identifies dead code. It must be inteligent enough to not have false positives with common import/export patterns in JS/TS implementations. It covers code that is uninvoked and unused imports and cross-references last modified times and potentially other metrics to truly identify dead code that can be safely deleted.
