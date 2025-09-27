
---

3. Suffix Automaton (SAM) — Compressed Suffix Engine

Автомат, хранящий все подстроки строки

📌 Позволяет:

считать количество уникальных подстрок

находить LCS между двумя строками

делать substring queries за O(1)

⚡ Архитектурный монстр → даёт выход на deep оптимизацию.

---

4. Heavy-Light Decomposition (HLD) — Tree Path Engine

Разбивает дерево на цепи для быстрых запросов по пути

📌 Применения:

path queries

subtree updates

гибрид с segment tree
---

5. Mo’s Algorithm — Offline Query Block Engine

Обрабатывает много запросов на отрезках за O((n + q)√n)

📌 Применения:

frequency queries

range XOR / count / sum

оптимизация без сегдеревьев
---

📦 Дополнительно

Dijkstra — модуль: граф + приоритетная очередь, O(E log V)



Suffix Array + Kasai — must-have (LCP queries, substring search).



FFT (Fast Fourier Transform) — умножение строк/чисел за O(n log n).
---

🎯 Дальше в план

Link-Cut Trees (динамические деревья, ещё один монстр).

Persistent Segment Tree (для задач на версии/историю).

DSU on Tree (техника для запросов по поддеревьям).

Centroid Decomposition (разложение деревьев).

Euler Tour + RMQ (LCA и не только).

---
