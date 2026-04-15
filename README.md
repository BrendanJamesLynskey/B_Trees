# 🌳 B-Trees — Presentation

An interactive slide deck covering B-tree operations, B+ trees, disk I/O models, database indexing, node splitting, and LSM-tree comparisons. Aimed at mid-level software engineers.

## ▶ [Open Presentation](https://brendanjameslynskey.github.io/B_Trees/index.html)

## 📄 [Markdown Version](presentation.md)

---

## Contents

| # | Topic |
|---|-------|
| 01 | Motivation — disk-based storage, minimising I/O |
| 02 | B-tree definition — order, minimum degree, properties |
| 03 | B-tree node structure |
| 04 | B-tree search |
| 05 | B-tree insertion — proactive splitting |
| 06 | Node splitting in detail |
| 07 | B-tree deletion — three cases |
| 08 | Deletion — merging and borrowing from siblings |
| 09 | B+ trees — all data in leaves, linked list |
| 10 | B+ tree range queries |
| 11 | B* trees — minimum 2/3 full |
| 12 | B-tree height analysis |
| 13 | Disk I/O model and why fanout matters |
| 14 | B-trees in databases (InnoDB, PostgreSQL) |
| 15 | B-trees in file systems (NTFS, HFS+, Btrfs) |
| 16 | LSM-trees vs B-trees |
| 17 | Fractal trees and write-optimised structures |
| 18 | Practical considerations |
| 19 | Summary and further reading |

---

## Slide Controls

| Action | Key |
|--------|-----|
| Next / Previous | `→` `←` or swipe |
| Overview | `Esc` |
| Fullscreen | `F` |
| Export to PDF | append `?print-pdf` to URL |

---

## Technology

[Reveal.js 4.6](https://revealjs.com) · [highlight.js](https://highlightjs.org) (Monokai) · Playfair Display + DM Sans + JetBrains Mono

Single self-contained `index.html` — no build step, no npm.

---

## References

- Cormen, T.H. et al. *Introduction to Algorithms* (CLRS), 4th ed. MIT Press, 2022 — Chapter 18: B-Trees
- Graefe, G. "Modern B-Tree Techniques." *Foundations and Trends in Databases*, 2011
- Kleppmann, M. *Designing Data-Intensive Applications*. O'Reilly, 2017 — Chapter 3
- Bayer, R. & McCreight, E. "Organization and Maintenance of Large Ordered Indexes." *Acta Informatica*, 1972
- Lehman, P.L. & Yao, S.B. "Efficient Locking for Concurrent Operations on B-Trees." *ACM TODS*, 1981
- Pavlo, A. *CMU 15-445: Database Systems* — [course recordings on YouTube](https://www.youtube.com/channel/UCHnBsf2rH-K7pn09rb3qvkA)

## License

Educational use. Code examples provided as-is. Standards references are to publicly available documentation.
