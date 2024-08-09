**Grafana dashboard for Openwrt using Collectd and Influx (With Downsampling)**

# Wstęp
   
W niniejszej instrukcji opiszę jak wyeksportować statystyki Collectd do bazy danych InfluxDB a nastepnie zwizualizować je w Grafanie. Dzieki temu bedziemy mogli przegladać szczegółowe statystyki naszego rutera z Openwrt (lub dowolnego systemy na ktorym uruchomione jest Collectd). Wykresy są wykresami (prawie) czasu rzeczywistego - ich dokładność to 1 minuta dla ostatnich 24 godzin. Statystyki są agregowane i próbkowane w dół (downsampling) co pozwala zobaczyć wykresy z ostatnich kilku miesięcy lub lat. Dzięki agregacji rozmiar bazy danych jest mały, oczywiscie kosztem szczegółowości (np. szczegółowośc statystyk z ostatniego roku to 6 godzin, gdzie dla osttanich 24 godzin jest to tylko 1 minuta). 
Rozwiazanie które opisuje powstało na bazie [Collectd to InfluxDB v2 with downsampling using Tasks](https://cloud-infra.engineer/collectd-influxdb2-grafana-with-downsampling/).

Uwaga na marginesie - jezeli ktos chce latwego i prostego roziwazania polecam collectd w połączeniu z: [CGP](https://github.com/pommi/CGP). Używam tego z powodzeniem i uważam za bardzo kompleksowe rozwiązanie.

# Wymagania
Zanim zaczniemy konfigurację należy przygotować następujące aplikacja na naszym serwerze (ja używam do tego Raspberrypi4), 

    - Influx 2.0 (być może zadziała tez wersja 1.8)
    - Influx 2.0 CLI
    - Telegraf
    - Grafana
    - Collectd

Instrukcja jak zainstalowac te apliakacje nie jest częścią tego podręcznika. Jest jednak wiele stron które opisują jak to zrobic krok po kroku. Mozna też użyc gotowych obrazów Dockera.

# Konfiguracja InfluxdB
1. Tworzymy org `my-org` (jeżeli już istnieje to ignorujemy ten krok)
   ```
   influx org create -n my-org
   ```
2. Ściągamy i aplikujemy szablon Influxdb dla "Gociwd"
   ```
   wget https://raw.githubusercontent.com/mstojek/gociwd/main/gociwd-influx-template.yml
   influx apply -o my-org -f gociwd-influx-template.yml 
   ```
3. Tworzymy użytkownika 'collectd' który będzie miał prawo zapisu do "bucket" 'collectd/autogen' 
   ```
   BUCKET=$(influx bucket list --hide-headers -n collectd/autogen | awk '{print $1}')
   influx auth create --org my-org --description 'collectd' --read-bucket $BUCKET --write-bucket $BUCKET
   ```
   Kopiujemy sobie stworzony własnie Token ID (nazywany potem przeze mnie `InfluxDB-Token-For-Collectd-User`), token ten bedzie nam potem potrzebny do konfiguracji aplikacji Telegraf.
4. Tworzymy użytkownika  'grafana' który będzie miał prawo odczytu do "bucketów" 'collectd/*'
   ```
   BUCKET_AUTOGEN=$(influx bucket list --hide-headers -n collectd/autogen | awk '{print $1}')
   BUCKET_DAY=$(influx bucket list --hide-headers -n collectd/day | awk '{print $1}')
   BUCKET_WEEK=$(influx bucket list --hide-headers -n collectd/week | awk '{print $1}')
   BUCKET_MONTH=$(influx bucket list --hide-headers -n collectd/month | awk '{print $1}')
   BUCKET_YEAR=$(influx bucket list --hide-headers -n collectd/year | awk '{print $1}')
   BUCKET_TENYEAR=$(influx bucket list --hide-headers -n collectd/tenyear | awk '{print $1}')
   BUCKET_HUNDYEAR=$(influx bucket list --hide-headers -n collectd/hundyear | awk '{print $1}')
   influx auth create --org my-org --description 'grafana' --read-bucket $BUCKET_AUTOGEN --read-bucket $BUCKET_DAY --read-bucket $BUCKET_WEEK --read-bucket $BUCKET_MONTH --read-bucket $BUCKET_YEAR --read-bucket $BUCKET_TENYEAR  --read-bucket $BUCKET_HUNDYEAR
   ```
   Kopiujemy sobie stworzony własnie Token ID (nazywany potem przeze mnie `Grafana-Token`), token ten bedzie nam potem potrzebny do konfiguracji aplikacji Grafana.
# Konfiguracja Telegraf
1. Dodajemy licznik typu "stations" do pliku 'types.db' - Openwrt używa tego typu do zliczania liczby podłączonych klientów do sieci WIFI.
   ```
   cat /usr/share/collectd/types.db
   [...]
   # OpenWRT
   stations                value:GAUGE:0:256
   ```
2. Edytujemy plik konfiguracyjny aplikacji Telegraf
   ```
   cat /etc/telegraf/telegraf.d/collectd.conf
   [[inputs.socket_listener]]
     service_address = "udp://:25827"
     data_format = "collectd"
     collectd_typesdb = ["/usr/share/collectd/types.db"]
     collectd_parse_multivalue = "split"

     [inputs.socket_listener.tags]
       bucket = "collectd"

   [[outputs.influxdb_v2]]
     urls = ["http://<InfluxdDB-IP-address>:8086"]
     token = "<InfluxDB-Token-For-Collectd-User>"
     organization = "my-org"
     bucket = "collectd/autogen"

     [outputs.influxdb_v2.tagpass]
       bucket = ["collectd"]
   ```
   Proszę zauwazyc że używamy portu 25827 ponieważ port 25826 jest używany przez Collectd. Musimy tez tutaj wpisać wlaściwy adres IP serwera InfluxDB 'InfluxdDB-IP-address' jak i podac wczesniej utworzony Token 'InfluxDB-Token-For-Collectd-User'

3. Restartujemy usługę Telegraf
   
   `sudo service telegraf restart`

# Konfiguracja Openwrt/Collectd do wysyłania statystyk do Telegraf/InfluxDB
1. Logujemy sie do Luci i idziemy do `Statistics->Setup->Output plugins->Network->Configure`

   W Sekcji `Server Interface` ustawiamy `Server Host` na `Telegraf-IP-address` zaś `Server Port` na `25827`
   ![obraz](https://github.com/user-attachments/assets/b8742665-3d44-4a5a-b0a1-c76210575745)
   
   (w moim przykładzie `Telegraf-IP-address` to `192.168.100.99`)

2. Jeżeli wolimy edytować pliki konfiguracyjne zamiast klikac w Luci to w pliku `/etc/collectd/collectd.conf` wpisujemy:

   ```
   LoadPlugin network
   <Plugin network> 
   Server "<Telegraf-IP-address>" "25827"
   </Plugin>
   ```

# Konfiguracja Grafany
1. Konfiguracja "Flux Data Source"
   
   Otwieramy w przegladarce stronę Grafany `(http://<Grafana-IP-address>:3000)`

   Logujemy się do Grafany
     
   Idziemy do `Home->Connections->DataSources->AddDataSource`

   - Wpisujemy nazwę dla `Data Source` (np. `collectd`)

   - W sekcji `Query language` 
     Wybieramy `Flux`

   - W sekcji `HTTP`
     Podajemy adres naszego serwera InfluxDB wraz z prawidłowym portem (domyślnie 8086) `http://<InfluxdDB-IP-address>:8086`

   - W sekcji `Auth`
     odznaczamy "Basic auth" (nic nie powinno byc zaznaczone)

   - W sekcji `InfluxDB details`
     Jako `Organization` podajemy `my-org`
     Podajemy `Grafana-Token` który stworzyliśmy wczesniej

   - Klikamy "Save&Test"

   ![obraz](https://github.com/user-attachments/assets/25cef571-20a0-4bb9-9c07-2d6d7b96f0ff)

2. Importuujemy Panel Grafana 

   - Kopiujemy na nasz dysk plik szablonu panelu [gociwd-grafana-template.json](https://raw.githubusercontent.com/mstojek/gociwd/main/gociwd-grafana-template.json)

   - Idziemy do `Grafana->Home->Dashboards->Import dashboard` i przeciagamy skopiowany plik [gociwd-grafana-template.json](https://raw.githubusercontent.com/mstojek/gociwd/main/gociwd-grafana-template.json) do pola "Import Field".
  
   - Idziemy do `Grafana->Home->Dashboards` i otwieramy nowo zaimportowany panel `Openwrt Collectd Graph Panel (Flux)`
  
   Alternatywnie możemy użyc innego szablonu Grafany [gociwd-grafana-template_AllwaysConnectNullValues.json](https://github.com/mstojek/gociwd/blob/main/gociwd-grafana-template_AllwaysConnectNullValues.json) szczególy w [Connect Null Values](https://github.com/mstojek/gociwd/blob/main/README.md#connect-null-values)
   
# Przykładowe statystyki

- Load, Memory and CPU statistics
![obraz](https://github.com/user-attachments/assets/66e23c7c-7360-49a5-ac57-26fad7ffbbd4)

- Interfaces Traffic data
![obraz](https://github.com/user-attachments/assets/0dc80779-77d0-4f5d-a54d-e9977711e165)
![obraz](https://github.com/user-attachments/assets/4b1982bc-927e-46ca-b9bc-e91b7ab7f4bc)

- Client Traffic data ([Iptmon](https://github.com/oofnikj/iptmon) or [Nlbwmon2Collectd](https://github.com/mstojek/nlbw2collectd) required)
![obraz](https://github.com/user-attachments/assets/9afd8715-339d-4aef-a46f-d0057d9f949c)
![obraz](https://github.com/user-attachments/assets/5b7b3872-b1ae-494d-b826-a1b224ab5f78)

- WIFI, Connections, DNS and DHCP leases statistics
![obraz](https://github.com/user-attachments/assets/d1b21ac4-f36f-4eeb-a954-4ec2cf9332c6)

- Mo zna tez oczywiscie dodać sobie samodzielnie inne wykresy i statystyki. Domyslnie wszystkie statystyki zbierane przez Collectd sa wysylane do InfluxDB. Mój panel wyświetla tylko drobny ich podzbiór.
 
# Aggregation danych

1. Dane są agregowane wraz z upływem czasu. Używane sa następujące ageregacje:
   - Daily - z rozdzielczoscią 1 minuty
   - Weekly - z rozdzielczoscią 5 minut
   - Monthly - z rozdzielczoscią 30 minut
   - Yearly - z rozdzielczoscią  6 godzin
   - 10 Year - z rozdzielczoscią 1 dzień
   - 100 Year - z rozdzielczoscią 1 tydzień
   Oryginalne dane wysłane z Collectd są przecvhowywane w "bucket" `Collect/autogen`
2. Przykładowe wykresy:
   - Godzinowy
     ![obraz](https://github.com/user-attachments/assets/43b50a9a-11ae-4f9d-90b6-4dc1e1f6e5fa)
   - Dobowy
     ![obraz](https://github.com/user-attachments/assets/f123b22b-f4ff-46f1-b6a6-852176cd48bc)
   - Miesieczny
     ![obraz](https://github.com/user-attachments/assets/21df2b0e-cd9d-49e5-b1ba-362e56962167)
   - 1 rok temu we wrześniu
     ![obraz](https://github.com/user-attachments/assets/4425430e-4987-4c8a-beb3-76d2c1807496)
     
# Parametryzacja wykresów
![obraz](https://github.com/user-attachments/assets/8e49807b-01b0-4f49-91ec-4f2b648f7673)
1. Możemy wybrac nastepujac parametry wykresów (z menu dropdown):
   - `Hostname` - mamy możliwość zbierania statystyk z wielu instancji Collectd (z kilku serwerów/ruterów - możemu mieć np. kilka ruterów z Openwrt i jakis serwer na Raspberrypi i dla każdego wyświetlic osobne statystyki) 
   - `Client` - Jeżeli mamy zainstalowany [Iptmon](https://github.com/oofnikj/iptmon) lub [Nlbwmon2Collectd](https://github.com/mstojek/nlbw2collectd) możemy wyswietlić statystyki ruchu na wykresie `Client Traffic` tylko dla wybranego klienta w naszej sieci
   - `Interface` - możemy filtrować wykres `Interfaces Traffic` dla wybranego interfejsu
   - `Wifi-Interface` - możemy filtrować wykresy `Wifi` dla danego interfejsu Wifi
   - `Aggregation Type` - dostępne opcje to: `Min, Max, Mean`
      - Wyjaśnienie: Domyslnie statystyki są wysyłane co 30 sekund (czasami częściej np. co 10 sekund, zależy to od konfigracji serwera Collectd). Teraz jeżeli wyświetlimy w panelu wykresy za ostatni miesiąc nie bedziemy miec rozdzielczości 30 sekund, tylko 30 minut (dzięki temu oszczedzamy miejsce na dysku zajete przez bazę danych InfluxDB). Teraz musimy podjac decyzje czy ten jeden pukt który agreguje dane za 30 minut to ma byc wartość: średnai za ostatnich 30 minut, maksymalna czy moze minimalna z tych 30 minut.
   - `Retention policy` - pole informacyjne, informuje z ktorego "bucketu" pobieramy dane.

# Connect Null Values
W opcjach wykresów greafana mamy pole `Connect Null Values`. Przygotowałem dwa szablony:
- Szablon w którym to pole jest ustawione na `Threshold <1h` - jest to plik: [gociwd-grafana-template.json](https://github.com/mstojek/gociwd/blob/main/gociwd-grafana-template.json)

  Przy tym ustawieniu poprawnie wyświetlamy wykresy krótkoterminowe
  
  ![obraz](https://github.com/user-attachments/assets/de9547c0-33ae-4450-9366-970243209560)

  Niestety wykresy długoterminowe składaja się z niepołączonych ze sobą kropek:
  
  ![obraz](https://github.com/user-attachments/assets/04057aaf-d7d5-4217-a46e-a1980850c174)

     
- Szablon w którym to pole jest ustawione na `Always` - jest to plik: [gociwd-grafana-template_AllwaysConnectNullValues.json](https://github.com/mstojek/gociwd/blob/main/gociwd-grafana-template_AllwaysConnectNullValues.json)

  Przy tym ustawieniu niepoprawnie wyświetlamy wykresy krótkoterminowe
  
  ![obraz](https://github.com/user-attachments/assets/b5d99358-9eed-40ce-b57e-480f55328035)

  Prosze zauważyć że między 9:10 a 13:30 w systemie nic się nie działo, wykres powinien pokazywać zero w tym czasie. Zamiast tego Grafana połączyła ostatni puknt z ~9:10 z pierwszym punktem o ~13:30

  Ale za to wykresy długoterminowe sa ładne i poprawne
  
  ![obraz](https://github.com/user-attachments/assets/915bb5c0-6e33-45b3-b5a9-35297af747ed)

- Nie mam na to żadnego rozwiązania.

# Źródła

1. Collectd Graph Panel - świetne narzędzie do wyświetlania statystyk z plikow RRD z collectd: [CGP](https://github.com/pommi/CGP)
2. Moje rozwiazanie to ewolucja systemu opisanego w: [Collectd to InfluxDB v2 with downsampling using Tasks](https://cloud-infra.engineer/collectd-influxdb2-grafana-with-downsampling/)

   Dodałem następujące rzeczy
   - przepisanie wszystkiego na Flux (to była kiepska decyzja bo Flux wszedł w [maintenance mode](https://docs.influxdata.com/flux/v0/future-of-flux/) ....)
   - lepsza obsługa agregacji i próbkowania w dół (downsampling)
4. Wątek o tym jak poprawnie zliczać liczniki "całkowitego użycia": [Add previousN and nextN to range](https://github.com/influxdata/flux/issues/702)
5. Sprytne rozwiązanie do dynamicznego wybierania "bucketów" w zależnosci od wybranego zakresu czasowego w Grafanie: http://wiki.webperfect.ch/index.php?title=Grafana:_Dynamic_Retentions_%28InfluxDB%29
