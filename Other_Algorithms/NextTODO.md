
🧠 2. Aho-Corasick Automaton — Multi-pattern Matching
Построение Trie + failure links

Позволяет искать множество паттернов одновременно за O(n)

Ведёт к задачам на:

фильтрацию по словарю

антиспам

биоинформатику

intrusion detection
---
🧠 3. Suffix Automaton (SAM) — Compressed Suffix Engine
Построение автомата, который хранит все подстроки строки

Позволяет:

считать количество уникальных подстрок

находить LCS между двумя строками

делать substring queries за O(1)

Это архитектурный монстр, но ведёт к глубокой оптимизации
---
🧠 4. Heavy-Light Decomposition (HLD) — Tree Path Engine
Разбивает дерево на цепи для быстрого запроса по пути

Ведёт к задачам на:

path queries

subtree updates

segment tree + tree fusion
---
🧠 5. Mo’s Algorithm — Offline Query Block Engine
Обрабатывает много запросов на отрезках за O((n + q)√n)

Ведёт к:

frequency queries

range XOR/count/sum

оптимизация без сегментных деревьев

---
Алгоритмы — Djikstra, Wilson
Djikstra — свернём как модуль: граф, приоритетка, O(E log V), с кейсами
---
Wilson — если ты про Wilson’s algorithm for uniform spanning trees — это вообще имба, мало кто юзает, но она архитектурно чистая


---


Suffix Array + Kasai — must-have (ты уже близко к нему подошёл).

Suffix Automaton — чутка сложнее, но даёт много мощных применений.

Rolling Hash (полиномные хэши) — часто в задачах на строки.

FFT (быстрое преобразование Фурье) — топ для умножения строк/чисел.

---
