# openstreetmap-tile-server

Kontener pozwalający na łatwe i szybkie uruchomienie serwera kafelków OpenStreetMap w domyślnym stylu graficznym na podstawie dostarczonego pojedynczego pliku `.osm.pbf`.

**Uwaga!**

Zwróć uwagę, że to repozytorium ma dwie główne gałęzie:
* `master` - uniwersalny kontener do obsługi dowolnych map;
* `poland` - kontener modyfikowany pod kątem serwowania mapy Polski z terenu Polski, czyli np. ustawiona jest odpowiednia stefa czasowa i nie ma chińskich czcionek.

Oprócz tego mogą tymczasowo być tworzne gałęzie eksperymentalne np. oferujące wyższe wersje oprogramowania bazodanowego, alternatywne mechanizmy importujące lub inne zmiany.

## Budowa kontenera

Kontener ten powstał jako specjalna wersja uniwersalnego kontenera overv/openstreetmap-tile-server i na chwilę obecną nie jest dostępny poprzez Docker Hub. Stąd należy go zbudować, aby z niego korzystać.

    git clone https://github.com/pot-gov-pl/openstreetmap-tile-server
    cd openstreetmap-tile-server
    sudo docker build --tag=openstreetmap-tile-server .

## Import danych

Utwórz wolumin dockerowy do przechowywania danych OpenStreetMap w bazie PosgtreSQL.

    docker volume create openstreetmap-data

Pobierz plik `.osm.pbf` z [geofabrik.de](https://www.geofabrik.de/) dla interesującego Cię regionu. Przykładowo, najświeższe dane o Polsce:

    wget http://download.geofabrik.de/europe/poland-latest.osm.pbf

Teraz zaimportuj je do PostgreSQL uruchamiając kontener z parametrem `import` z podmontowanym pobranym plikiem jako `/data.osm.pbf`.

    docker run -v /absolute/path/to/poland-latest.osm.pbf:/data.osm.pbf -v openstreetmap-data:/var/lib/postgresql/10/main openstreetmap-tile-server import

Jeżeli kontener zakończy pracę bez błędów, będzie to oznaczać, że Twoje dane zostały poprawnie zaimportowane i wszystko jest gotowe, aby uruchomić serwer.

## Uruchomienie serwera

Uruchom serwer następującym wywołaniem z parametrem `run` oraz przypisaniem go do portu 80:

    docker run -p 80:80 -v openstreetmap-data:/var/lib/postgresql/10/main -d openstreetmap-tile-server run

Kafelki będą dostępne pod adresem http://localhost:80/tile/{z}/{x}/{y}.png a pod adresem http://localhost:80/leaflet-demo.html powinna być dostępna prosta demonstracyjna aplikacja internetowa. Przy pierwszym uruchomieniu pojawienie się pierwszych kafelków może zająć chwilę, bo muszą zostać przygotowane.


## Zachowanie wyrenderowanych kafelków

Aby pracowicie wytworzone kafelki przetrwały restart kontenera, najpierw utwórz dla nich wolumin:

    docker volume create openstreetmap-rendered-tiles

Następnie uruchamiaj serwer z dodatkowym parametrem wskazującym na ten wolumin;

    docker run -p 80:80 -v openstreetmap-data:/var/lib/postgresql/10/main -v openstreetmap-rendered-tiles:/var/lib/mod_tile -d openstreetmap-tile-server run

## Optymalizacja wydajności

Domyślnie proces importowy oraz serwer korzystają z 4 wątków, ale wartość tę można zmienić ustawiając zmienną środowiskową `THREADS`, np. tak:

    docker run -p 80:80 -e THREADS=24 -v openstreetmap-data:/var/lib/postgresql/10/main -d openstreetmap-tile-server run

## Rozwiązywanie problemów

### ERROR: could not resize shared memory segment

Jeżeli trafisz na takie coś w logu, to znaczy to, że  domyślny dockerowy limit pamiędzi współdzielonej (równy 64 MB) jest zbyt niski i trzeba go zwiększyć:

    renderd[126]: ERROR: failed to render TILE ajt 6 32-39 16-23,
    renderd[126]: reason: Postgis Plugin: ERROR: could not resize shared memory segment

Nową wartość należy ustawć parametrem `--shm-size`. Przykład:

    docker run -p 80:80 -v openstreetmap-data:/var/lib/postgresql/10/main --shm-size="192m" -d openstreetmap-tile-server run

Zbyt wysokie wartości mogą sprawić, że znacząco wzrośnie użycie CPU oraz pamięci. Możliwe, że trzeba będzie eksperymentalnie określić odpowiednią wartość limitu.

## Licencja

```
Copyright 2018 Alexander Overvoorde

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

    http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.
```
