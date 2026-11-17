# Scénář 1
```

BenchmarkDotNet v0.15.8, Windows 11 (10.0.26200.9974/25H2/2025Update/HudsonValley2)
Intel Core i7-8750H CPU 2.20GHz (Max: 2.21GHz) (Coffee Lake), 1 CPU, 12 logical and 6 physical cores
.NET SDK 10.0.401
  [Host]     : .NET 10.0.12 (10.0.12, 10.0.1226.42308), X64 RyuJIT x86-64-v3
  DefaultJob : .NET 10.0.12 (10.0.12, 10.0.1226.42308), X64 RyuJIT x86-64-v3


```
| Method                | UniqueWords | Mean       | Error     | StdDev    | Ratio | RatioSD | Gen0      | Gen1      | Gen2      | Allocated   | Alloc Ratio |
|---------------------- |------------ |-----------:|----------:|----------:|------:|--------:|----------:|----------:|----------:|------------:|------------:|
| **ContainsKeyAndIndexer** | **1000**        |   **5.136 ms** | **0.1715 ms** | **0.4864 ms** |  **1.01** |    **0.13** |   **15.6250** |         **-** |         **-** |    **99.74 KB** |        **1.00** |
| TryGetValue           | 1000        |   3.175 ms | 0.0630 ms | 0.1680 ms |  0.62 |    0.06 |   15.6250 |         - |         - |    99.74 KB |        1.00 |
| TryCatchKeyNotFound   | 1000        |   6.615 ms | 0.1586 ms | 0.4651 ms |  1.30 |    0.15 |  117.1875 |   15.6250 |         - |   560.73 KB |        5.62 |
|                       |             |            |           |           |       |         |           |           |           |             |             |
| **ContainsKeyAndIndexer** | **50000**       |  **10.565 ms** | **0.2101 ms** | **0.2945 ms** |  **1.00** |    **0.04** |  **937.5000** |  **906.2500** |  **906.2500** |  **3976.82 KB** |        **1.00** |
| TryGetValue           | 50000       |   9.119 ms | 0.1780 ms | 0.2925 ms |  0.86 |    0.04 | 1000.0000 |  953.1250 |  953.1250 |  3976.84 KB |        1.00 |
| TryCatchKeyNotFound   | 50000       | 141.204 ms | 2.8106 ms | 6.0500 ms | 13.38 |    0.67 | 5250.0000 | 1250.0000 | 1000.0000 | 24094.84 KB |        6.06 |


## Metodika měření

Benchmark měří čistou režii napočítání četností slov do `Dictionary<string, int>` bez vlivu I/O operací a parsování vstupu. Vstupní pole o délce 100 000 slov se generuje předem v `[GlobalSetup]` ve dvou variantách parametru `UniqueWords`:
* **1 000 unikátních slov:** Simuluje text s vysokou mírou opakování.
* **50 000 unikátních slov:** Simuluje text s vysokou diverzitou slovní zásoby.

Všechny měřené metody vracejí `dict.Count`.

## Analýza výsledků

* **`TryGetValue`:** Dosahuje času 3,175 ms při 1 000 unikátních slovech a 9,119 ms při 50 000 unikátních slovech. Provádí pouze jedno čtení ze slovníku a následně jeden zápis přes indexer. U 50 000 unikátních slov je rozdíl oproti baseline menší, protože při prvním výskytu slova se u baseline neprovádí čtení přes indexer a velkou část času spotřebuje samotné zvětšování kapacity slovníku.
* **`ContainsKey` + indexer:** Při každém opakovaném slově počítá hash a prohledává kolizní řetězec vícekrát (nejprve v `ContainsKey`, poté při čtení hodnoty a jejím přepsání), což vede k měřitelné režii navíc (5,136 ms resp. 10,565 ms).
* **`try / catch (KeyNotFoundException)`:** Při 1 000 unikátních slovech se vyhodí 1 000 výjimek (6,615 ms). Při 50 000 unikátních slovech se vyhodí 50 000 výjimek, což výrazně zhorší čas na 141,204 ms a vygeneruje 24 094,84 KB alokací kvůli vytváření instancí výjimek.
* **Paměťové alokace:** Přístupy `TryGetValue` i `ContainsKey` alokují totožné množství paměti (~99,74 KB pro 1 000 slov a ~3 976,83 KB pro 50 000 slov). Samotná inkrementace nealokuje žádné objekty na haldě – veškerá alokovaná paměť připadá na vnitřní pole `buckets` a `entries` při růstu slovníku.

## Závěr pro úlohu Četnosti slov

Pro standardní implementaci úlohy *Četnosti slov* je nejvhodnější použít **`TryGetValue`**. Oproti naivnímu `ContainsKey` odstraňuje zbytečné dvojí vyhledávání, zachovává plnou typovou i paměťovou bezpečnost a kód zůstává čitelný.
