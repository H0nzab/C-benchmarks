# C#-benchmarks

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
| GetValueOrDefault     | 1000        |   3.166 ms | 0.0629 ms | 0.1773 ms |  0.62 |    0.07 |   19.5313 |         - |         - |    99.74 KB |        1.00 |
| CollectionsMarshalRef | 1000        |   2.357 ms | 0.0470 ms | 0.0981 ms |  0.46 |    0.05 |   19.5313 |         - |         - |    99.74 KB |        1.00 |
| TryCatchKeyNotFound   | 1000        |   6.615 ms | 0.1586 ms | 0.4651 ms |  1.30 |    0.15 |  117.1875 |   15.6250 |         - |   560.73 KB |        5.62 |
|                       |             |            |           |           |       |         |           |           |           |             |             |
| **ContainsKeyAndIndexer** | **50000**       |  **10.565 ms** | **0.2101 ms** | **0.2945 ms** |  **1.00** |    **0.04** |  **937.5000** |  **906.2500** |  **906.2500** |  **3976.82 KB** |        **1.00** |
| TryGetValue           | 50000       |   9.119 ms | 0.1780 ms | 0.2925 ms |  0.86 |    0.04 | 1000.0000 |  953.1250 |  953.1250 |  3976.84 KB |        1.00 |
| GetValueOrDefault     | 50000       |   9.530 ms | 0.2055 ms | 0.5728 ms |  0.90 |    0.06 | 1000.0000 |  953.1250 |  953.1250 |  3976.84 KB |        1.00 |
| CollectionsMarshalRef | 50000       |   7.319 ms | 0.1458 ms | 0.3464 ms |  0.69 |    0.04 | 1031.2500 |  984.3750 |  984.3750 |  3976.85 KB |        1.00 |
| TryCatchKeyNotFound   | 50000       | 141.204 ms | 2.8106 ms | 6.0500 ms | 13.38 |    0.67 | 5250.0000 | 1250.0000 | 1000.0000 | 24094.84 KB |        6.06 |


## Metodika měření

Benchmark měří čistou režii napočítání četností slov do `Dictionary<string, int>` bez vlivu I/O operací a parsování vstupu. Vstupní pole o délce 100 000 slov se generuje předem v `[GlobalSetup]` ve dvou variantách parametru `UniqueWords`:
* **1 000 unikátních slov:** Simuluje text s vysokou mírou opakování (dominuje aktualizace existujících klíčů, slovník se zvětšuje minimálně).
* **50 000 unikátních slov:** Simuluje text s vysokou diverzitou slovní zásoby (polovinu operací tvoří první vložení nového klíče a častá realokace vnitřních polí slovníku).

Všechny měřené metody vracejí `dict.Count`, aby JIT kompilátor nevyoptimalizoval samotný výpočet (dead-code elimination).

## Analýza výsledků

* **`CollectionsMarshal.GetValueRefOrAddDefault` (nejrychlejší v obou scénářích):** Dosahuje času 2,357 ms při 1 000 unikátních slovech (více než dvojnásobné zrychlení oproti baseline) a 7,319 ms při 50 000 unikátních slovech[cite: 3]. Výpočet hashe a vyhledání v bucketech proběhne pouze jednou – metoda vrátí přímou referenci (`ref int`) na hodnotu v interním poli `entries` slovníku a inkrementace se provede přímo na místě bez druhého vyhledávání.
* **`TryGetValue` a `GetValueOrDefault` (téměř shodný výkon):** Při 1 000 unikátních slovech dosahují časů 3,175 ms a 3,166 ms, což je zrychlení přibližně o 38 % oproti baseline[cite: 3]. U 50 000 unikátních slov je rozdíl menší (9,119 ms a 9,530 ms oproti 10,565 ms u baseline)[cite: 3], protože při prvním výskytu slova se u baseline neprovádí čtení přes indexer a velkou část času spotřebuje samotné zvětšování kapacity (`Resize`) slovníku. Obě metody provádějí jedno čtení (pokud klíč neexistuje, vrátí `0`) a následně jeden zápis přes indexer.
* **`ContainsKey` + indexer (baseline):** Při každém opakovaném slově počítá hash a prohledává kolizní řetězec vícekrát (nejprve v `ContainsKey`, poté při čtení hodnoty a jejím přepsání), což vede k měřitelné režii navíc (5,136 ms resp. 10,565 ms)[cite: 3].
* **`try / catch (KeyNotFoundException)` (nevhodný antipattern):** Při 1 000 unikátních slovech se vyhodí 1 000 výjimek (6,615 ms, alokace 560,73 KB)[cite: 3]. Při 50 000 unikátních slovech se vyhodí 50 000 výjimek, což degraduje čas na 141,204 ms (přes 13× pomalejší než baseline) a vygeneruje 24 094,84 KB alokací[cite: 3] kvůli vytváření instancí výjimek, zachytávání stack trace a procházení zásobníku (stack unwinding).
* **Paměťové alokace:** První čtyři přístupy alokují totožné množství paměti (~99,74 KB pro 1 000 slov a ~3 976,84 KB pro 50 000 slov)[cite: 3]. Samotná inkrementace nealokuje žádné objekty na haldě – veškerá alokovaná paměť připadá na vnitřní pole `buckets` a `entries` při růstu slovníku (včetně průchodů GC v Gen1 a Gen2 u většího slovníku)[cite: 3].

## Závěr pro úlohu Četnosti slov

Pro standardní implementaci úlohy *Četnosti slov* je nejvhodnější použít **`TryGetValue`** (nebo `GetValueOrDefault`). Oproti naivnímu `ContainsKey` odstraňuje zbytečné dvojí vyhledávání, zachovává plnou typovou i paměťovou bezpečnost a kód zůstává čitelný.

Pokud by bylo cílem vytěžit absolutní maximum výkonu (např. při zpracování obřích korpusů v řádu desítek milionů slov), je nejlepší volbou **`CollectionsMarshal.GetValueRefOrAddDefault`**. Jeho nevýhodou je však nižší bezpečnost – vrácená `ref int` reference se stane neplatnou při jakékoli strukturální změně slovníku (vložení dalšího prvku, `Clear`, `Remove`), což vyžaduje opatrnost při údržbě kódu.
