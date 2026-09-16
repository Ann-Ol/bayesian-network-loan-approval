[README.md](https://github.com/user-attachments/files/32298095/README.md)
# Analiza decyzji kredytowej z użyciem sieci Bayesowskiej

## Opis projektu

Projekt przedstawia zastosowanie dyskretnej **sieci Bayesowskiej** do analizy decyzji kredytowych.  
Celem nie jest budowa produkcyjnego systemu scoringowego, lecz analiza zależności probabilistycznych pomiędzy cechami klienta, historią kredytową, parametrami pożyczki oraz końcową decyzją kredytową.

Analiza została wykonana w **R** na zbiorze zawierającym **50 000 wniosków kredytowych**.

Projekt obejmuje:

- eksploracyjną analizę danych i kontrolę jakości,
- analizę redundancji cech,
- dyskretyzację zmiennych liczbowych,
- uczenie struktury sieci Bayesowskiej,
- porównanie algorytmów Hill-Climbing, Tabu Search i MMHC,
- wybór modelu na podstawie kryterium BIC,
- 5-krotną walidację krzyżową,
- ocenę stabilności łuków metodą bootstrap,
- analizę tablic prawdopodobieństw warunkowych (CPT),
- wnioskowanie prognostyczne i diagnostyczne.

## Dane

W analizie wykorzystano zbiór **Realistic Loan Approval Dataset: US and Canada** dostępny na Kaggle:

https://www.kaggle.com/datasets/parthpatel2130/realistic-loan-approval-dataset-us-and-canada

Zbiór zawiera informacje o profilu klienta, jego sytuacji finansowej, historii kredytowej oraz parametrach wnioskowanej pożyczki.

Zmienną wynikową jest:

- `loan_status` — decyzja kredytowa (`Pozytywna` / `Odrzucona`).

Istotną cechą analizowanego zbioru jest fakt, że wszystkie obserwacje z `defaults_on_file = 1` mają negatywną decyzję kredytową. W projekcie traktowane jest to jako właściwość tego konkretnego zbioru danych, a nie jako uniwersalna reguła kredytowa.

## Przygotowanie danych

Przed modelowaniem sprawdzono między innymi braki danych, duplikaty oraz redundancję pomiędzy zmiennymi.

Z sieci Bayesowskiej usunięto kilka zmiennych będących niemal deterministycznymi transformacjami innych cech:

- `current_debt`,
- `loan_to_income_ratio`,
- `payment_to_income_ratio`.

Zmienna `debt_to_income_ratio` została zachowana, ponieważ stanowi czytelną biznesowo miarę obciążenia dochodu zadłużeniem.

Zmienne liczbowe zostały w większości zdyskretyzowane do trzech kategorii:

- `niski`,
- `sredni`,
- `wysoki`.

W przypadku zmiennych o małej liczbie unikalnych wartości zachowano ich obserwowane wartości jako poziomy faktora.

## Modelowanie sieci Bayesowskiej

Porównano trzy algorytmy uczenia struktury:

| Algorytm | BIC |
|---|---:|
| Hill-Climbing | -671801.7 |
| **Tabu Search** | **-671787.1** |
| MMHC | -679563.5 |

W zastosowanej implementacji `bnlearn` wyższa wartość BIC oznacza korzystniejszy kompromis pomiędzy dopasowaniem i złożonością modelu. Dlatego do dalszej analizy wybrano **Tabu Search**.

Finalna sieć zawiera:

- **16 węzłów**,
- **24 skierowane łuki**.

Maksymalną liczbę rodziców pojedynczego węzła ograniczono do trzech.

Zmienna `loan_status` została potraktowana jako zmienna wynikowa, dlatego podczas uczenia struktury zablokowano łuki wychodzące z tego węzła. Kierunki łuków należy interpretować jako element przyjętej struktury modelu, a nie jako dowód zależności przyczynowo-skutkowej.

## Walidacja modelu

Jako dodatkową ocenę jakości zastosowano **5-krotną walidację krzyżową**.

| Model | Trafność CV |
|---|---:|
| Hill-Climbing | 77,33% |
| Tabu Search | 77,33% |
| MMHC | 77,33% |
| Klasyfikator większościowy | 55,0% |

Walidacja ma charakter uzupełniający, ponieważ progi dyskretyzacji zostały wyznaczone przed podziałem danych na foldy. Z tego powodu uzyskanej trafności nie należy traktować jako produkcyjnej estymacji jakości modelu na całkowicie nowych danych.

## Stabilność struktury

Do oceny stabilności łuków zastosowano metodę **bootstrap**.

Najbardziej stabilne bezpośrednie zależności prowadzące do `loan_status` to:

| Od | Do | Siła bootstrap |
|---|---|---:|
| `credit_score` | `loan_status` | 1,00 |
| `debt_to_income_ratio` | `loan_status` | 1,00 |
| `defaults_on_file` | `loan_status` | 0,98 |

Te trzy zmienne są również bezpośrednimi rodzicami `loan_status` w wybranej strukturze.

Wysoka stabilność bootstrapowa oznacza, że dane połączenie pojawia się często w sieciach uczonych na różnych próbach danych. Nie oznacza to jednak zależności przyczynowej.

## Wnioskowanie probabilistyczne

Na podstawie dopasowanej sieci porównano kilka scenariuszy klientów.

| Scenariusz | Prawdopodobieństwo pozytywnej decyzji |
|---|---:|
| Brak dodatkowych informacji | 55,04% |
| Wysoki credit score, brak defaultów, niski DTI | 94,03% |
| Średni credit score, brak defaultów, wysoki DTI | 36,84% |
| Default w historii | 0,01% |

W projekcie wykonano również wnioskowanie diagnostyczne:

`P(wysoki credit_score i niski DTI | loan_status = Pozytywna) = 18,46%`

Otrzymane wartości opisują zależności warunkowe w analizowanym zbiorze i nie powinny być interpretowane jako efekty przyczynowe.

## Technologie i biblioteki

Projekt został wykonany w **R / R Markdown** z wykorzystaniem pakietów:

- `bnlearn`,
- `gRain`,
- `dplyr`,
- `ggplot2`,
- `igraph`,
- `ggraph`,
- `tidygraph`.

## Struktura repozytorium

```text
.
├── README.md
├── analiza_decyzji_kredytowej_bayes.Rmd
├── analiza_decyzji_kredytowej_bayes.html
└── Loan_approval_data_2025.csv
```

Plik HTML zawiera wyrenderowany raport z tabelami, wizualizacjami oraz pełną analizą.

## Jak uruchomić projekt

1. Pobierz lub sklonuj repozytorium.
2. Umieść plik `Loan_approval_data_2025.csv` w tym samym katalogu co plik `.Rmd`.
3. W razie potrzeby zainstaluj wymagane pakiety R.
4. Otwórz `analiza_decyzji_kredytowej_bayes.Rmd` w RStudio.
5. Wybierz **Knit → Knit to HTML**.

## Ograniczenia

Najważniejsze ograniczenia projektu:

- wyuczonej struktury nie należy interpretować przyczynowo,
- dyskretyzacja zmiennych ciągłych prowadzi do utraty części informacji,
- preprocessing został wykonany przed walidacją krzyżową,
- część silnych zależności może być specyficzna dla analizowanego zbioru danych,
- bootstrap mierzy stabilność struktury, ale nie gwarantuje identyfikacji prawdziwej sieci,
- model nie obejmuje kalibracji prawdopodobieństw, kosztów błędów, fairness, driftu ani wymogów regulacyjnych.

Projekt należy więc traktować jako analityczne i edukacyjne zastosowanie sieci Bayesowskich, a nie jako gotowy do wdrożenia system scoringowy.

## Autorzy

**Bartosz Kawalec**  
**Anna Oleszko**

Projekt został wykonany wspólnie.
