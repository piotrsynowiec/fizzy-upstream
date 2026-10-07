# Nawigacja między kartami w kolumnie — analiza i propozycja

## Problem i efekt zmiany

Otwarta karta nie pozwala dziś przejść do sąsiedniej karty. Trzeba wrócić do boardu, odnaleźć kolumnę i ponownie przewinąć listę. To szczególnie przeszkadza przy przeglądaniu długiej kolejki i zamykaniu zadań po kolei.

Proponowana zmiana dodaje dwa okrągłe przyciski na prawej ramce karty, pod dzwonkiem i pinezką: ↑ nad ↓. Na telefonie są w drugim rzędzie pod tymi samymi ikonami. Shift+↑ otwiera poprzednią kartę, Shift+↓ następną. Esc nadal wraca do punktu wejścia. Na początku i końcu kolejki odpowiedni przycisk jest wyłączony; lista nie zawija się.

## Co wynika z kodu

Analiza dotyczy źródeł Basecamp Fizzy. Implementację przygotowałem na main `48f56c0453e27e825dbc55369263e6beb3e60da5`, aktualny commit zmiany: `c91ea677b156ec4b66d11e914b156cd02e5ac38b`.

- Widok boardu ma już nawigację klawiaturą przez `navigable_list_controller.js`. Otwarty widok karty nie ma jednak kontekstu poprzedniej i następnej karty.
- Kolumny wyświetlają aktywne karty przez `.latest.with_golden_first`. Golden Tickets mają pierwszeństwo, następnie decydują ostatnia aktywność i identyfikator.
- Lista kolumny jest stronicowana. Czytanie wyłącznie elementów obecnych w DOM zatrzymałoby nawigację na końcu załadowanej strony.
- Aktywność zmienia pozycję karty. Wyliczanie kolejki od nowa po każdym komentarzu mogłoby powodować przeskoki, powtórzenia i pomijanie kart.
- `turbo_navigation_controller.js` pamięta poprzedni ekran. Po kilku przejściach między kartami trzeba zachować pierwotny adres powrotu, aby Esc nadal prowadził do boardu.
- Fizzy ma już okrągłe przyciski i ikonę arrow-up. Przyciski korzystają z istniejących klas, a nowy wariant arrow-up-light zachowuje zaokrąglenia przy grubości 4,25 jednostki zamiast 5 jednostek trzonu pierwotnej ikony — około 15% mniej.

## Zachowanie kolejki

Przy otwarciu karty aplikacja pobiera numery wszystkich aktywnych kart jej kolumny, w kolejności boardu. W bieżącej karcie przeglądarki zapamiętuje kolejność, kolumnę oraz pierwotny adres powrotu. Nie pobiera treści wszystkich kart.

Przy każdym przejściu sprawdza aktualną dostępność kart. Zachowuje początkową kolejność, ale pomija karty zamknięte, przeniesione poza kolumnę albo usunięte. Zamknięcie lub przeniesienie bieżącej karty nie przełącza kolejki na inną kolumnę. Analogicznie działa kolejka Maybe.

Nowe karty dodane w trakcie przeglądania nie są dopisywane do zapamiętanej kolejki. Powrót do boardu i ponowne otwarcie karty rozpoczyna nową kolejkę. Jest to świadomy kompromis: przegląd ma pozostać przewidywalny mimo zmian aktywności.

Przejście klawiaturą jest blokowane podczas pisania w formularzu lub edytorze, przy otwartym dialogu oraz przy edycji samej karty. Powtarzanie przytrzymanego klawisza i podwójne uruchomienie przejścia są ignorowane. Komentarze korzystają z istniejącego mechanizmu lokalnego zapisu.

## Małe, oddzielne elementy implementacji

| Element | Odpowiedzialność |
| --- | --- |
| Card::Navigation | Dobór kolumny, zakresu kart i ich kolejności |
| Cards::NavigationsController | Odczyt numerów kart przez istniejące uprawnienia |
| card_navigation_controller.js | Zapamiętanie kolejki, skróty, przejścia i powrót |
| cards/_navigation.html.erb | Przyciski pod istniejącymi akcjami na prawej ramce |
| card-navigation.css | Układ przycisków na desktopie i telefonie oraz obrót strzałki w dół |

Endpoint `GET /{account}/cards/{number}/navigation` zwraca `numbers`, `column_id` i `name`. Bieżąca karta jest wyszukiwana przez `Current.user.accessible_cards`; wskazana kolumna musi należeć do boardu tej karty. Klucz pamięci przeglądarki zawiera ścieżkę konta, więc kolejki różnych kont i kart przeglądarki są rozdzielone.

Zmiana nie wymaga migracji bazy ani nowych zależności. Zajmuje 12 plików, w tym trzy pliki testów.

## Zakres i ograniczenia

Pierwsza wersja obejmuje aktywne karty nazwanej kolumny oraz Maybe. Nie odtwarza osobnej kolejki wyników wyszukiwania lub filtrów. Karta otwarta z filtrowanego boardu prowadzi przez całą swoją aktywną kolumnę. Done i Not Now nie mają własnych kolejek w tej wersji.

Pełna lista numerów jest pobierana przy wejściu i ponownie przy przejściu. To proste rozwiązanie bez pobierania opisów i załączników. Koszt rośnie liniowo z liczbą kart kolumny; bardzo duże kolumny mogą później uzasadnić endpoint z kursorami. Nie wykonywałem benchmarku dużych produkcyjnych boardów.

Błąd pobierania wyłącza przyciski, pozostawiając kartę dostępną; ponowne otwarcie strony ponawia pobranie. Sprawdzenie przed przejściem ogranicza wyścigi, ale usunięcie karty dokładnie między sprawdzeniem a otwarciem nadal może zakończyć się standardowym 404.

## Weryfikacja

Zestaw dla pierwszego commitu: **18 testów, 76 asercji, bez błędów i pominięć**. Obejmuje nowe testy modelu, endpointu i przeglądarki oraz istniejące testy powrotu do boardu i jego nawigacji klawiaturą.

Sprawdzone przypadki: Golden Tickets, pełna lista ponad pierwszą stronę, Maybe, dostęp do prywatnego boardu, obca kolumna, oba przyciski i skróty, brak zawijania, stała kolejność mimo aktywności, pomijanie zamkniętej karty, zamknięcie bieżącej karty, edytor, dialog oraz Esc.

Po przeniesieniu przycisków ponownie przeszło 10 testów systemowych i 53 asercje, w tym zamknięcie bieżącej karty i Esc. Zmianę grubości SVG oraz końcowy układ sprawdziłem ponownie w przeglądarce.

Rubocop: sześć plików, bez uwag. Kontrola składni JavaScript i diffu również przeszła.

Ręczny odbiór wykonany w prawdziwej przeglądarce na **lokalnym podglądzie**, na kodzie tej zmiany i przykładowym boardzie z 70 kartami: desktop 1280×720 oraz telefon 390×844. Sprawdziłem kliknięcia, skrót, przejście 15→16 poza pierwszą stronę kolumny oraz Esc. Na telefonie oba przyciski są pod dzwonkiem i pinezką i mają ten sam rozmiar 48×48 px; szerokość dokumentu wynosi 390 px, bez poziomego przepełnienia. Zrzuty pokazują wyłącznie przykładowe dane.

Nie wdrożono zmiany na produkcję. Do użycia we własnej instancji potrzebny będzie obraz aplikacji zawierający ten commit; obecna konfiguracja wdrożenia nadal wskazuje dotychczasowy obraz.

## Rekomendacja

Zostawiłbym ten zakres jako pierwszy, atomowy PR. Usuwa główną przeszkodę w przeglądaniu kolumny i nie wymaga nowego systemu skrótów ani przebudowy boardu. Kolejki filtrowanych wyników warto dodać osobno, jeśli okażą się potrzebne w codziennej pracy.
