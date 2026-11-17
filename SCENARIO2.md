# Scénář 2
```

BenchmarkDotNet v0.15.8, Windows 11 (10.0.26200.9974/25H2/2025Update/HudsonValley2)
Intel Core i7-8750H CPU 2.20GHz (Max: 2.21GHz) (Coffee Lake), 1 CPU, 12 logical and 6 physical cores
.NET SDK 10.0.401
  [Host]     : .NET 10.0.12 (10.0.12, 10.0.1226.42308), X64 RyuJIT x86-64-v3
  DefaultJob : .NET 10.0.12 (10.0.12, 10.0.1226.42308), X64 RyuJIT x86-64-v3


```
| Method               | UniqueWords | Mean       | Error      | StdDev     | Median     | Ratio | RatioSD | Gen0     | Gen1     | Gen2     | Allocated  | Alloc Ratio |
|--------------------- |------------ |-----------:|-----------:|-----------:|-----------:|------:|--------:|---------:|---------:|---------:|-----------:|------------:|
| **UseSortedList**        | **1000**        | **115.068 ms** |  **1.6959 ms** |  **1.5033 ms** | **114.966 ms** | **27.34** |    **3.46** |        **-** |        **-** |        **-** |   **24.44 KB** |        **0.21** |
| UseSortedDictionary  | 1000        |  94.355 ms |  1.8732 ms |  2.5640 ms |  93.766 ms | 22.42 |    2.89 |        - |        - |        - |   54.95 KB |        0.48 |
| UseDictionaryAndSort | 1000        |   4.271 ms |  0.1720 ms |  0.5016 ms |   4.300 ms |  1.01 |    0.17 |  23.4375 |   3.9063 |        - |  115.59 KB |        1.00 |
|                      |             |            |            |            |            |       |         |          |          |          |            |             |
| **UseSortedList**        | **20000**       | **271.278 ms** | **10.2153 ms** | **29.7986 ms** | **266.876 ms** |  **9.70** |    **1.66** |        **-** |        **-** |        **-** |  **768.67 KB** |        **0.35** |
| UseSortedDictionary  | 20000       | 186.341 ms |  3.7126 ms |  6.5024 ms | 187.158 ms |  6.66 |    0.91 |        - |        - |        - | 1088.02 KB |        0.49 |
| UseDictionaryAndSort | 20000       |  28.502 ms |  1.4188 ms |  4.1612 ms |  26.762 ms |  1.02 |    0.20 | 531.2500 | 500.0000 | 500.0000 | 2225.07 KB |        1.00 |


## Metodika měření

Benchmark porovnává tři přístupy k řešení kompletní úlohy *Četnosti slov* včetně finálního abecedního průchodu výslednými daty (simulovaného sečtením četností do proměnné `checksum`, což zároveň brání JIT kompilátoru v odstranění kódu).

Vstupní pole o délce 100 000 slov je připraveno v `[GlobalSetup]` pro dvě velikosti slovníku (`UniqueWords`):
* **1 000 unikátních slov:** Vysoký podíl opakování (dominuje aktualizace četností).
* **20 000 unikátních slov:** Vyšší podíl prvních vložení a větší finální sada k seřazení.

Pro férové srovnání používá ex-post třídění u `Dictionary` stejný výchozí porovnávač (`Comparer<string>.Default`) jako `SortedList` a `SortedDictionary`.

## Analýza výsledků

* **`Dictionary<string, int>` + ex-post `.Sort()` (nejrychlejší řešení):** Dosahuje času **4,271 ms** při 1 000 unikátních slovech a **28,502 ms** při 20 000 unikátních slovech. Během průchodu 100 000 slov probíhají vyhledávání i aktualizace v amortizovaném čase $O(1)$ pomocí hashe. Drahé lexikografické porovnávání řetězců (`Comparer<string>.Default`) se provede až **jednou na konci** při seřazení $U$ unikátních dvojic v čase $O(U \log U)$ nad souvislým polem v `List<T>` (Introsort), což výborně využívá CPU cache. Daní za rychlost je nejvyšší alokace paměti (115,59 KB resp. 2 225,07 KB), protože kromě interních polí slovníku alokujeme pomocný `List<KeyValuePair<string, int>>` pro setřídění.
* **`SortedDictionary<string, int>` (6,7× až 22× pomalejší než baseline):** Časy **94,355 ms** (1 000 slov) a **186,341 ms** (20 000 slov). Jako červeno-černý strom má garantovanou složitost $O(\log U)$, ale provádí dvě průchodové operace stromem (`TryGetValue` + zápis indexerem) pro každé ze 100 000 slov na vstupu, přičemž v každém uzlu provádí kulturově závislé porovnání řetězců. Navíc každý uzel stromu je samostatný objekt na haldě, což způsobuje časté výpadky CPU cache (pointer chasing).
* **`SortedList<string, int>` (nejpomalejší řešení):** Časy **115,068 ms** (1 000 slov) a **271,278 ms** (20 000 slov). Uvnitř udržuje dvě souvislá pole (`keys[]` a `values[]`), díky čemuž má nejnižší paměťovou náročnost (24,44 KB resp. 768,67 KB) a nejrychlejší finální sekvenční průchod. Při vkládání nového klíče doprostřed pole však musí posunout všechny následující prvky (`Array.Copy` v čase $O(U)$) a při každém slově provádí dvě binární vyhledávání v $O(\log U)$ s nákladným porovnáváním stringů.
* **Proč relativní rozdíl klesá z ~27× na ~9,7×:** Celkový počet zpracovaných slov (100 000) zůstává stejný, takže `SortedList` i `SortedDictionary` provedou podobný řád porovnání řetězců (čas vzroste jen ~2× až 2,4×). U `Dictionary + Sort` ale při 20 000 unikátních slovech naroste režie zvětšování hashovací tabulky a především jednorázového setřídění 20× většího seznamu řetězců (čas vzroste ~6,7×). Přesto zůstává řádově rychlejší.

## Závěr pro úlohu Četnosti slov

Pro reálné řešení úlohy *Četnosti slov* je jednoznačně nejlepší zvolit **`Dictionary<string, int>` v kombinaci s ex-post setříděním** (případně pro další zrychlení předat `StringComparer.Ordinal` tam, kde není vyžadováno kulturově závislé řazení). `SortedList` i `SortedDictionary` udržují uspořádání průběžně při každém ze $N$ vstupních slov, což je v situaci, kdy seřazený výstup potřebujeme až po načtení celého vstupu, zbytečná práce navíc.
Odevzdej jako druhé odevzdání v ReCodExu s komentářem přesně Scénář 2.
