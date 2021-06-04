# Dokumentacja

## 1. Schemat urządzenia i środowisko w którym działa

Schemat
![boardirl](boardirl.jpg)
![boardschem](boardschem.jpg)
Wykorzysujemy:
- lyraT
  - dekoder
  - moduł wi-fi
  - przyciski dotykowe i "normalne"
- wi-fi
- głośniki
- zasilanie - bezpośrednie podłączenie do USB

[Espressify - documentation](https://docs.espressif.com/projects/esp-adf/en/latest/get-started/get-started-esp32-lyrat.html)

## 2. Zrealizowane funkcjonalności

Celem projektu było stworzenie radia internetowego. Gotowy produkt ma następujące funkcjonalności:
- lista dostępnych stacji jest zawarta w pliku źródłowym (poprzez adresy URL)
  - działają zarówno strony streamujące plik, jak i parsowanie playlist (zakodowanych np. w m3u8)
- obsługiwane jest wiele formatów kodowania:
  - dekoder "auto" automatycznie rozpoznaje pliki mp3 i aac 
  - dekoder aac jest ustawiony ręcznie dla plików mp4a (niestety dekoder auto miał problem z odpowiednim rozpoznawaniem tego formatu)
- możliwość przechodzenia za pomocą przycisku SET cylkicznie po liście stacji
- możliwość regulacji głośności za pomocą przycisków VOL+, VOL-
- możliwość przerywania i wznawiania odtwarzania radia za pomocą przycisku PLAY
- kończenie pracy radia za pomocą przycisku MODE

## 3. Stos technologiczny

- Platforman sprzętowa: lyraT v4.3
- Język programowania: C
- IDE: Visual Studio Code + ESP-IDF oficjalny plugin
- Biblioteki: esp-idf + esp-adf

Plugin do VSC jest szczególnie pomocny. Umożliwia bardzo wiele, w niezwykle przystępny sposób, a jego instalacja jest praktycznie bezproblemowa.
Feature'y (które są niezwykle wygodne, bo bardzo łatwo je wyszukać - nie zapamiętałem żadnych skrótów klawiszowych poza shif+ctrl/cmd+p na wyszukiwarkę poleceń pluginów) - dodatkowo komendy te wykorzystują stare terminale jeśli jest potrzeba, ale np. monitorowanie mają osobno, więc łatwo mieć dostęp do historii:
- Budowanie projektu.
- Czyszczenie projektu.
- Wybieranie portu na którym należy wgrać program (Flash).
- Monitorowanie portu, aby móc odczytywać komunikaty z płytki.
- Można debugować, ale z tego akurat nie korzystaliśmy.
- Można przeanalizować rozmiary binarek pod względem wykorzystania ramu.
- Włączenie przyjemnego GUI w którym można zmieniać ustawienia projektu (wi-fi, rodzaj płytki itp.).
- Tworzenie nowego projektu, czy to z gotowego przykładu, który chcemy rozbudować, czy po prostu czystego projektu.
- I pewnie jeszcze trochę innych rzeczy, z których nie korzystaliśmy.

Oto naszym zdaniem dwa najciekawsze przykłady, wykorzystania GUI z plugina (mimo, że najczęściej używaliśmy build, flash i monitor)
- Ustawienia projektu
  ![settings](sets.png)
- Analiza wykorzystywanej pamięci;
  ![memory1](mem1.png)
  ![memory2](mem2.png)

## 4. Opis kluczowych części projektu zrealizowanych samodzielnie

### 4.0 Nagłówki
```c
#include <string.h>
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "freertos/event_groups.h"
#include "esp_log.h"
#include "esp_wifi.h"
#include "esp_decoder.h"
#include "nvs_flash.h"
#include "sdkconfig.h"
#include "audio_element.h"
#include "audio_pipeline.h"
#include "audio_event_iface.h"
#include "audio_common.h"
#include "http_stream.h"
#include "i2s_stream.h"
#include "aac_decoder.h"
// #include "mp3_decoder.h"
// #include "ogg_decoder.h"

#include "esp_peripherals.h"
#include "periph_wifi.h"
#include "periph_touch.h"
#include "periph_adc_button.h"
#include "periph_button.h"
#include "board.h"

#if __has_include("esp_idf_version.h")
#include "esp_idf_version.h"
#else
#define ESP_IDF_VERSION_VAL(major, minor, patch) 1
#endif

#if (ESP_IDF_VERSION >= ESP_IDF_VERSION_VAL(4, 1, 0))
#include "esp_netif.h"
#else
#include "tcpip_adapter.h"
#endif

static const char *TAG = "RADIO";
```

### 4.1 Lista obsługiwanych stacji
```c
typedef struct{
    char* name;
    char* decoder_name;
    char* uri;
} radio_station_t;

radio_station_t stations[] = {
    {
        .name = "station1",
        .decoder_name = "auto", // aac
        .uri = "https://streams.radiomast.io/a622d414-52a6-4426-b3b8-ed2a4dbb704b"
    },
    {
        .name = "station2",
        .decoder_name = "auto", // aac
        .uri = "http://open.ls.qingting.fm/live/274/64k.m3u8?format=aac"
    },
    // { OGG doesn't work :(
    //     .name = "station4",
    //     .decoder_name = "auto", // ogg
    //     .uri = "https://listen.moe/stream"
    // },
    {
        .name = "station3",
        .decoder_name = "auto", // mp3
        .uri = "http://musicbird.leanstream.co/JCB104-MP3"
    },
    {
        .name = "station4",
        .decoder_name = "auto", // aac
        .uri = "http://stream2.bbnradio.org:8000/chinese.aac"
    },
    {
        .name = "station5",
        .decoder_name = "aac", // mp4a
        .uri = "http://101.79.244.199:1935/cocotv/_definst_/CH00007/playlist.m3u8"
    },
    {
        .name = "station6",
        .decoder_name = "aac", // mp4a
        .uri = "http://sk.cri.cn/hyhq.m3u8"
    }
};

#define RADIOS_NUMBER (sizeof(stations) / sizeof(radio_station_t))
int current_ix = 0;
```

Daną stację opisuje specjalna struktura `radio_station_t`, w której trzymamy:
- nazwę stacji (wyświetla się w jedynie w logach z powodu braku ekranu)
- nazwę dekodera, którego chcemy użyć do odbierania tej stacji (o dostępnych dekoderach i działadniu więcej w sekcji 4.2)
- URI stacji (link bezpośrednio do pliku z streamem audio, lub do pliku z playlistą .m3u8)

Playlisty .m3u8 (w zasadzie M3U, m3u8 to M3U kodowane w UTF-8) to niewielkie pliki tekstowe. Szczegółowe informacje na ich temat można przeczytać tutaj: [m3u8 - wiki](https://en.wikipedia.org/wiki/M3U).

Poza opcjonalnymi metadanymi po prostu zawierają adresy elementów playlisty. Element zawiera adres bezwzględny, lub względny (względem folderu, w którym znajduje się playlista) do docelowego pliku. W sczególności może istnieć playlista jednoelementowa lub elementem playlisty może być inna playlista. W przypadku playlisty konkretnego radia internetowego bardzo częstym przypadkiem jest stworzenie playlisty, której elementami są mirror streamy; dzięki takiemu rozwiązaniu gdy jeden stream ulegnie awarii odbiornik radia automatycznie przełącza się na kolejny adres z playlisty.

### 4.2 Inicjalizacja audio kodeku
// TODO: Kacper
### 4.3 Konfiguracja `audio_pipelnie`
// TODO: Kacper


### 4.4 Konfiguracja wi-fi i przycisków - peryferia
```c
esp_periph_config_t periph_cfg = DEFAULT_ESP_PERIPH_SET_CONFIG();
esp_periph_set_handle_t set = esp_periph_set_init(&periph_cfg);
periph_wifi_cfg_t wifi_cfg = {
    .ssid = CONFIG_WIFI_SSID,
    .password = CONFIG_WIFI_PASSWORD,
};
esp_periph_handle_t wifi_handle = periph_wifi_init(&wifi_cfg);
audio_board_key_init(set);
esp_periph_start(set, wifi_handle);
periph_wifi_wait_for_connected(wifi_handle, portMAX_DELAY);
```
Standardowa konfiguracja, SSID oraz hasło podajemy w pliku konfiguracyjnym.
Dodatkowo incjujemy obsługę przycisków. Oba uchwyty dodajemy do piepeline'a.

### 4.5 Ustawienie listenera
// TODO: Kacper
### 4.6 Start radia

#### 4.6.0 Odbieranie `music_info`
// TODO: 私

#### 4.6.1 Fail-over
// TODO: Kacper

#### 4.6.2 Obsługa poszczególnych przycisków
// TODO: 私
### 4.7 Zatrzymanie wszystkiego i deinicjalizacja 
// TODO: 私
