# Zadanie 06 - Raport

## 1. Własne testy

### Wynik polecenia `docker compose exec backend pytest -v` (wszystkie testy zaliczone):

```text
============================= test session starts ==============================
platform linux -- Python 3.12.16, pytest-8.4.2, pluggy-1.7.0 -- /usr/local/bin/python3.12
cachedir: .pytest_cache
rootdir: /app
collecting ... collected 8 items

tests/test_check.py::test_leaked_hash_is_found PASSED                    [ 12%]
tests/test_check.py::test_unknown_hash_is_not_found PASSED               [ 25%]
tests/test_check.py::test_missing_hash_field PASSED                      [ 37%]
tests/test_check.py::test_plaintext_password_is_rejected PASSED          [ 50%]
tests/test_check.py::test_uppercase_hash_is_found PASSED                 [ 62%]
tests/test_check.py::test_short_hash_is_rejected PASSED                  [ 75%]
tests/test_check.py::test_non_hex_hash_is_rejected PASSED                [ 87%]
tests/test_check.py::test_non_json_request_is_rejected PASSED            [100%]

============================== 8 passed in 0.29s ===============================
```

### Wynik polecenia `docker compose exec backend pytest` bez `.lower()`:

```text
============================= test session starts ==============================
platform linux -- Python 3.12.16, pytest-8.4.2, pluggy-1.7.0
rootdir: /app
collected 8 items

tests/test_check.py ....F...                                             [100%]

=================================== FAILURES ===================================
_________________________ test_uppercase_hash_is_found _________________________

client = <FlaskClient <Flask 'app'>>

    def test_uppercase_hash_is_found(client):
        response = client.post("/check", json={"hash": LEAKED.upper()})
>       assert response.status_code == 200
E       assert 400 == 200
E        +  where 400 = <WrapperTestResponse streamed [400 BAD REQUEST]>.status_code

tests/test_check.py:27: AssertionError
=========================== short test summary info ============================
FAILED tests/test_check.py::test_uppercase_hash_is_found - assert 400 == 200
========================= 1 failed, 7 passed in 0.30s ==========================
```

### Dlaczego testy z zajęć nie wykryły usunięcia `.lower()`:

Testy przygotowane na zajęciach przesyłały w ciele żądania wyłącznie hashe zdefiniowane małymi literami (zarówno `LEAKED`, jak i `NOT_LEAKED` składały się z małych znaków hex). Żaden z pierwotnych testów nie weryfikował obsługi skrótów z wielkimi literami A-F, przez co brak konwersji `.lower()` nie naruszał walidacji wyrażeniem regularnym `[0-9a-f]{40}`.

---

## 2. Awaria a blokada

### Wynik polecenia curl po zatrzymaniu bazy:

```text
HTTP 500
```

### Wynik poleceń `docker compose ps -a` oraz `docker inspect`:

```text
NAME                        IMAGE                    COMMAND                  SERVICE   CREATED          STATUS                      PORTS
breach-checker-backend-1    breach-checker-backend   "flask --app app run…"   backend   38 minutes ago   Exited (1) 10 seconds ago   
breach-checker-db-1         postgres:16-alpine       "docker-entrypoint.s…"   db        7 days ago       Exited (0) 36 seconds ago   
breach-checker-frontend-1   breach-checker-frontend  "/docker-entrypoint.…"   frontend  7 days ago       Up About an hour            0.0.0.0:8080->80/tcp, [::]:8080->80/tcp
```

```text
exited 1
```

### Tabela porównawcza: Awaria vs Blokada

| Cecha | Awaria (zatrzymana baza) | Blokada z kroku 4 (lock bazy) |
|---|---|---|
| **Status w `docker compose ps`** | `Exited (1)` po restarcie (lub `Up (unhealthy)`) | `Up (healthy)` (wszystkie usługi działają) |
| **CPU w `docker stats`** | ~0.00% (brak procesów lub natychmiastowe zakończenie) | ~0.00% (proces czeka na zwolnienie blokady, stan wait) |
| **Ostatnie linie logów** | `psycopg2.OperationalError: could not translate host name "db"` | Zapytanie `check: hash=...` oczekujące na odpowiedź |
| **Co dostał klient** | Kod błędu `HTTP 500` z tracebackiem debuggera | Długie oczekiwanie, a ostatecznie `HTTP 504 Gateway Timeout` |
| **Po jakim czasie** | Natychmiast (< 1 s) | Po upłynięciu limitu czasu (np. 60 s w Nginx) |

### Co zawierał `blad.html` i dlaczego byłby zagrożeniem na produkcji:

Plik `blad.html` zawierał pełną interaktywną stronę debuggera Werkzeug wraz ze szczegółowym stosem wywołań (traceback), fragmentami kodu źródłowego aplikacji, ścieżkami w kontenerze oraz parametrami połączenia ze środowiskiem bazy. Na serwerze produkcyjnym ujawnienie takich danych stanowi poważny wyciek informacji (Information Disclosure), ułatwiający przeprowadzenie ukierunkowanego ataku, a obecność aktywnej konsoli debugera z kodem PIN mogłaby umożliwić nieuwierzytelnione zdalne wykonanie kodu (RCE).

---

## 3. Frontend w trybie deweloperskim

### Wynik polecenia `curl http://localhost:8080/nginx.conf` przed poprawką:

```nginx
server {
    listen 80;

    # Pliki strony (index.html, style.css, script.js)
    location / {
        root /usr/share/nginx/html;
        index index.html;
    }

    # Wszystko pod /api/ nginx przekazuje do usługi backend w sieci Compose
    location /api/ {
        proxy_pass http://backend:5000/;
    }
}
```

### Wynik polecenia `curl.exe -I http://localhost:8080/nginx.conf` po poprawce:

```text
HTTP/1.1 404 Not Found
Server: nginx/1.31.6
Date: Fri, 06 Nov 2026 13:49:08 GMT
Content-Type: text/html
Content-Length: 153
Connection: keep-alive
```

### Obraz produkcyjny a bezpieczeństwo bind mountów:

Obraz produkcyjny frontendu nie ma tego problemu, ponieważ instrukcja `COPY` w [frontend/Dockerfile](file:///c:/Users/anton/OneDrive/Dokumenty/metody_wirtualizacji_lab/breach-checker/Frontend/Dockerfile) precyzyjnie przenosi do katalogu `/usr/share/nginx/html` jedynie pliki interfejsu (`index.html`, `style.css`, `script.js`), natomiast `nginx.conf` kopiowany jest w dedykowane miejsce konfiguracji Nginxa (`/etc/nginx/conf.d/default.conf`). Montowanie całego katalogu projektu jako web root serwera WWW jest ryzykowne, ponieważ ujawnia w sieci publicznej pliki konfiguracyjne, definicje kontenerów (`Dockerfile`), skrypty czy potencjalnie ukryte pliki z danymi wrażliwymi.
