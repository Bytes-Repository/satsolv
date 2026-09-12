# satsolv

Bindingi [H#](https://) do [libsolv](https://github.com/openSUSE/libsolv) —
biblioteki SAT-solvera używanej m.in. przez `libzypp`/`zypper` (openSUSE),
`dnf`/`rpm`, `conda`/`rattler` i inne menedżery pakietów do rozwiązywania
zależności.

**100% H#.** Żadnego kodu C ani Rust napisanego przez nas — łączymy się
bezpośrednio z prawdziwymi, eksportowanymi symbolami `libsolv`/`libsolvext`
przez `extern static [c, "..."]`, zgodnie ze składnią FFI opisaną w README H#.

> **Status:** biblioteka nie została jeszcze skompilowana ani przetestowana
> wobec prawdziwego libsolv w tym środowisku (brak zbudowanego toolchaina
> H# + zainstalowanego libsolv-dev). Składnia jest ręcznie zweryfikowana
> wobec parsera/typecheckera H#, a sygnatury/stałe libsolv wobec jego
> nagłówków źródłowych — ale realna kompilacja + `bytes test` to konieczny
> następny krok przed użyciem produkcyjnym.

## Wymagania systemowe

**Tylko platformy 64-bitowe (LP64): x86_64, aarch64, riscv64.** `mod queue`
i `mod repo::set_priority/get_priority` czytają/piszą surową pamięć
prawdziwych struktur C libsolv pod stałymi offsetami wyliczonymi dla
64-bitowych wskaźników — na platformie 32-bitowej byłyby błędne. Biblioteka
weryfikuje to założenie w runtime (zobacz "Uwagi techniczne" niżej) i
**panikuje z czytelnym komunikatem**, zamiast po cichu psuć pamięć, jeśli
się nie zgadza — ale i tak nie próbuj używać tego na wasm32 czy platformie
32-bitowej.

`satsolv` **linkuje** się z libsolv — sama biblioteka nie jest w to
wliczona. Przed budową projektu korzystającego z `satsolv` zainstaluj
nagłówki/biblioteki dev libsolv (razem z rozszerzeniami — potrzebujemy
`libsolvext` dla parsera formatu "testtags"), np.:

```sh
# Debian/Ubuntu
sudo apt install libsolv-dev

# Fedora
sudo dnf install libsolv-devel

# openSUSE
sudo zypper install libsolv-devel

# Arch
sudo pacman -S libsolv
```

`extern static [c, "libsolv"]` / `extern static [c, "libsolvext"]` w kodzie
H# próbują najpierw `pkg-config --static --libs libsolv` (odpowiednio
`libsolvext`), a jeśli `pkg-config` nie znajdzie pliku `.pc`, spadają na
`-lsolv` / `-lsolvext`.

## Instalacja

```sh
bytes add satsolv
```

## Szybki start

```h#
use "satsolv" from "sv"

fn main() is
    let p: int = sv::pool::create()
    sv::pool::set_arch(p, "x86_64")

    ;; repo "systemowe" (nic zainstalowanego na start)
    let system: int = sv::repo::create(p, "system")
    sv::pool::set_installed(p, system)
    sv::repo::internalize(system)

    ;; repo "dostępne" — dwa pakiety zbudowane "z palca"
    let available: int = sv::repo::create(p, "available")
    sv::repo::set_priority(available, 10, 0)

    let foo: sv::repo::Pkg = sv::repo::new_pkg("foo", "1.0", "1", "x86_64")
    let mut bar: sv::repo::Pkg = sv::repo::new_pkg("bar", "2.0", "1", "x86_64")
    bar.requires = ["foo >= 1.0"]

    match sv::repo::add_packages(p, available, [foo, bar]) is
        sv::SatResult::Ok(rc) => is end
        sv::SatResult::Err(msg) => is
            write("Blad wczytywania pakietow: " + msg)
            return
        end
    end
    sv::repo::internalize(available)

    sv::pool::createwhatprovides(p)

    ;; job: zainstaluj "bar" w wersji >= 1.5 (a solver dociągnie "foo")
    let jobs: int = sv::queue::new()
    match sv::job::install_name_version(jobs, p, "bar", ">=", "1.5") is
        sv::SatResult::Ok(rel_id) => is end
        sv::SatResult::Err(msg) => is
            write("Blad joba: " + msg)
            return
        end
    end

    let s: int = sv::solver::create(p)
    sv::solver::allow_downgrade(s, false)

    let problems: i32 = sv::solver::solve(s, jobs)

    if problems > 0 is
        for msg in sv::solver::all_problems_as_strings(s) is
            write(msg)
        end
    else is
        let t: int = sv::transaction::create_from_solver(s)
        for sid in sv::transaction::installed_result(t) is
            write(sv::pool::solvable_to_str(p, sid))
        end
        sv::transaction::free(t)
    end

    sv::queue::free(jobs)
    sv::solver::free(s)
    sv::repo::free(available, false)
    sv::repo::free(system, false)
    sv::pool::free(p)
end
```

## Struktura API

| moduł | co robi |
|---|---|
| `pool` | cykl życia `Pool`, internowanie stringów/Id (`str2id`/`id2str`), porównania wersji (`evrcmp`/`evrcmp_str`), typ dystrybucji (`set_disttype`) |
| `repo` | cykl życia `Repo`, priorytety (`set_priority`/`get_priority`), wczytywanie pakietów z plików `.solv` (`add_solv_file`) lub "z palca" (`add_packages` — z walidacją, patrz niżej) |
| `queue` | odpowiednik `Queue` z libsolv (kolejka `Id`), własna implementacja w H# — patrz sekcja "Uwagi techniczne" |
| `job` | stałe `SOLVER_*` + budowanie kolejki zadań solvera (`install_name`, `install_name_version`, `erase_name`, `verify_all`, ...) |
| `solver` | cykl życia `Solver`, flagi konfiguracyjne (`set_flag`/`allow_downgrade`/`allow_vendorchange`/`best_obey_policy`/`keep_orphans`/...), `solve()`, odczyt problemów |
| `transaction` | wynik `solve()`: klasy kroków (`classify`), konkretne pakiety danej klasy (`classify_pkgs`), pełna lista do instalacji (`installed_result`) |

Funkcje, które mogą faktycznie zawieść (I/O na plikach, parsowanie repo,
zły operator porównania wersji), zwracają `SatResult<T>`
(`SatResult::Ok(wartosc)` / `SatResult::Err(opis_bledu)`) zamiast surowego
kodu powrotu — sprawdź wynik przez `match`, tak jak w przykładach wyżej.

## Budowanie pakietów bez pliku `.solv`

Do syntetycznego tworzenia pakietów `satsolv` **nie** grzebie w wewnętrznym
layoucie struktury `Solvable` (to zbyt kruche i zależne od wersji libsolv).
Zamiast tego używa oficjalnego, udokumentowanego formatu tekstowego libsolv
o nazwie *testtags* przez prawdziwą funkcję `testcase_add_testtags`
(`libsolvext`):

```h#
let mut pkg: sv::repo::Pkg = sv::repo::new_pkg("nginx", "1.24.0", "2", "x86_64")
pkg.requires   = ["libc.so.6()(64bit)", "openssl >= 3.0"]
pkg.provides   = ["webserver"]
pkg.conflicts  = ["apache2"]

match sv::repo::add_packages(p, myrepo, [pkg]) is
    sv::SatResult::Ok(rc) => is end
    sv::SatResult::Err(msg) => is write("blad: " + msg) end
end
```

**Walidacja wejścia.** Format *testtags* jest rozdzielany liniami (linia
`"#>=Tag: wartość"` trwa do końca linii), więc `add_packages` waliduje
każde pole *przed* sklejeniem tekstu:

- `name`/`version`/`release`/`arch` muszą być pojedynczym tokenem — bez
  spacji, tabulatorów ani znaków nowej linii (są rozdzielone spacją w
  jednej linii `#>=Pkg: name version release arch`, więc spacja w
  środku przesunęłaby kolejne pola).
- Wpisy w `requires`/`provides`/`conflicts`/`obsoletes`/`recommends`/
  `suggests` mogą zawierać spacje (np. `"foo >= 1.0"`), ale nie mogą
  zawierać `\n`/`\r` — to jedyny sposób, żeby wartość „uciekła” poza swój
  tag i stała się osobną, dowolną linią `#>=...` (czyli wstrzyknęła
  dodatkowy, niechciany pakiet do repo).

Jeśli którekolwiek pole nie przejdzie walidacji, `add_packages` zwraca
`SatResult::Err(...)` z opisem problemu i **nic nie zapisuje** — ani jeden
pakiet z podanej listy nie trafia do repo.

## Uwagi techniczne (dla ciekawych/kontrybutorów)

* Wszystkie nieprzezroczyste wskaźniki C (`Pool*`, `Repo*`, `Solver*`,
  `Transaction*`, `Queue*`, `FILE*`) są reprezentowane jako H# `int`
  (64-bitowy adres) — tak samo jak `malloc`/`free` w przykładzie z README H#.
  Nie wykonuj na nich arytmetyki poza tym modułem.
* `Id` w libsolv to `typedef int Id` (C `int`, 32 bity ze znakiem) — dlatego
  wiążemy go jako H# `i32`, nie `int` (który w H# to 64-bitowy `int64_t`).
  Użycie `int` tutaj dałoby błędne wartości dla ujemnych Id/kodów błędów.
* `queue_push`/`queue_pop`/`queue_shift` z `queue.h` libsolv są
  `static inline` — **nie mają eksportowanego symbolu** w `.so`/`.a`, więc
  nie da się ich zbindować przez `extern`. Moduł `queue` odtwarza ich
  logikę w czystym H#, operując na surowej pamięci struktury `Queue`
  (layout `elements`/`count`/`alloc`/`left`, 32 bajty — niezmienny od lat
  w publicznym ABI libsolv) przez wbudowane `ptr_read_i32`/`ptr_write_i32`/
  `ptr_read_ptr`/`arc_alloc` w blokach `unsafe`. Sam rozrost bufora
  (`queue_alloc_one`) **jest** prawdziwym, zlinkowanym kodem libsolv.
* **Założenie LP64 i samotest.** Powyższy layout `Queue` (i analogiczny
  layout pól `priority`/`subpriority` w `mod repo`) zakłada 64-bitowe
  wskaźniki — prawdziwe dla x86_64/aarch64/riscv64, jedynych sensownych
  natywnych celów kompilatora H# do linkowania z libsolv (H# potrafi też
  kompilować na wasm32, gdzie wskaźniki są 32-bitowe i te offsety byłyby
  błędne — ale wasm32 i tak nie linkuje natywnych `.so`, więc to nie jest
  realny scenariusz). `queue::new()` **weryfikuje to w runtime**:
  `queue_init()` to prawdziwy, skompilowany kod libsolv, który na LP64
  zapisuje zera pod dokładnie znanymi nam offsetami — jeśli po jego
  wywołaniu odczytamy pod tymi offsetami coś innego niż zera, layout się
  nie zgadza i biblioteka **panikuje** z czytelnym komunikatem zamiast po
  cichu kontynuować z uszkodzoną pamięcią. To nie jest dowód matematyczny
  (teoretycznie "obcy" bajt mógłby przypadkiem wylądować jako zero), ale w
  praktyce bardzo skuteczna siatka bezpieczeństwa.
* Świadomie nie dotykamy wewnętrznego layoutu `Pool`/`Solvable` — zbyt
  złożone i zbyt kruche na zmiany między wersjami. Stąd wybór
  `testcase_add_testtags` do tworzenia pakietów programistycznie i
  `repo_add_solv` do wczytywania prawdziwych metadanych repo. `Repo`
  (dla `priority`/`subpriority`) i `Queue` to jedyne dwie struktury C,
  których pola czytamy/piszemy bezpośrednio — obie mają w pełni publiczny,
  udokumentowany prefiks pól (poza `#ifdef LIBSOLV_INTERNAL`), więc jest
  to świadomy, ograniczony wyjątek od reguły "nie dotykamy layoutu".
* **`transaction::classify` nie zwraca listy Id pakietów.** Zwraca
  kolejkę wypełnioną czwórkami `(type_id, count, from_id, to_id)` — po
  jednej czwórce na każdą *klasę* kroków transakcji (potwierdzone wobec
  SWIG-owych bindingów libsolv, `bindings/solv.i`). `mod transaction`
  dekoduje to poprawnie na `[TransactionClass]`; żeby dostać faktyczne Id
  pakietów jednej klasy, użyj `classify_pkgs()` z wynikiem `classify()`.

## Licencja

MIT (bindingi). libsolv samo w sobie jest na licencji BSD — patrz jego
własne repozytorium.
