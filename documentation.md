<div style="text-align: right"> Stanisław Denkowski, Kacper Karoń 09.06.2021</div>
# Dokumentacja projektu radia internetowego na platformie ESP32 LyraT

## 1. Schemat urządzenia i fizyczne środowisko

<img src="boardirl.jpg" alt="boardirl" style="zoom: 80%;" />
*Powyższe zdjęcie zostało skopiowane z oficjalnej dokumentacji płytki: https://docs.espressif.com/projects/esp-adf/en/latest/get-started/get-started-esp32-lyrat.html* 
<img src="boardschem.jpg" alt="boardschem" style="zoom:80%;" />
*Powyższy diagram został skopiowany z oficjalnej dokumentacji płytki: https://docs.espressif.com/projects/esp-adf/en/latest/get-started/get-started-esp32-lyrat.html*

Wykorzysujemy:

- ESP32-LyraT V4.3
- Access Point Wi-Fi
- dodatkowe głośniki dołączone do płytki
- zasilanie z portu USB komputera

## 2. Zrealizowane funkcjonalności

Celem projektu było stworzenie radia internetowego. Gotowy produkt ma następujące funkcjonalności:
- lista dostępnych stacji jest zawarta w pliku źródłowym (poprzez adresy URL)
  - działają zarówno strony streamujące plik, jak i parsowanie playlist (zakodowanych np. w m3u8)
- obsługiwane jest wiele formatów kodowania:
  - dekoder "auto" automatycznie rozpoznaje pliki mp3 i aac 
  - dekoder aac jest ustawiony ręcznie dla plików mp4a (niestety dekoder "auto" miał problem z odpowiednim rozpoznawaniem tego formatu)
- możliwość przechodzenia za pomocą przycisku SET cylkicznie po liście stacji
- możliwość regulacji głośności za pomocą przycisków VOL+, VOL-
- możliwość przerywania i wznawiania odtwarzania radia za pomocą przycisku PLAY
- kończenie pracy radia za pomocą przycisku MODE

## 3. Stos technologiczny

- Platforman sprzętowa: ESP32-LyraT V4.3
- Język programowania: C
- IDE: Visual Studio Code + ESP-IDF oficjalny plugin
- Biblioteki: esp-idf + esp-adf

// TODO: nieprecyzyjnie
Warto wspomnieć, że jest możliwe ustawianie automatycznego wgrywania programów na płytkę, jednak my z tej możliwości, nie korzystaliśmy. Sami ręcznie wprowadzaliśmy płytkę w stan umożliwiający, usunięcie starego programu i wgranie nowego. Aby to zrobić należy równocześnie wcisnąć przyciski Boot i RST, a następnie je odklikiwać stopniowo, zaczynając od RST.

Plugin do VSC jest szczególnie pomocny. Umożliwia bardzo wiele, w niezwykle przystępny sposób, a jego instalacja jest praktycznie bezproblemowa.
// TODO: usunąć nawiasy (?), lepiej podrzędne zdanie
Funkcjonalności (które są niezwykle wygodne, bo bardzo łatwo je wyszukać - 
// TODO: wtrącenie jest subiektywne, lepiej osobny punkt zbiorczy
nie zapamiętałem żadnych skrótów klawiszowych poza shif+ctrl/cmd+p na wyszukiwarkę poleceń pluginów) - dodatkowo komendy te wykorzystują stare terminale jeśli jest potrzeba, ale np. monitorowanie mają osobno, więc łatwo mieć dostęp do historii:

- Budowanie projektu.
- Czyszczenie projektu.
- Wybieranie portu na którym należy wgrać program (Flash).
- Monitorowanie portu, aby móc odczytywać komunikaty z płytki.
- Mamy możliwość przeprowadzenia debugowania, jednak ta opcja nie była wykorzystywana przy realizacji projektu.
- Można przeanalizować rozmiary binarek pod względem wykorzystania ramu.
- Włączenie przyjemnego GUI w którym można zmieniać ustawienia projektu (wi-fi, rodzaj płytki itp.).
- Tworzenie nowego projektu, czy to z gotowego przykładu, który chcemy rozbudować, czy po prostu czystego projektu.
- I pewnie jeszcze trochę innych rzeczy, z których nie korzystaliśmy.

Oto naszym zdaniem dwa najciekawsze przykłady, wykorzystania GUI z plugina (mimo, że najczęściej używaliśmy build, flash i monitor)
- Ustawienia projektu zastosowane przy tworzeniu oprogramowania (obraz poglądowy)
  ![settings](sets.png)
- Analiza wykorzystywanej pamięci (obraz poglądowy)
  ![memory1](mem1.png)
  ![memory2](mem2.png)

## 4. Opis kluczowych części projektu zrealizowanych samodzielnie
// TODO: link do szkieletowego projektu
// TODO: porównać (można wyciągnąć gdzieś indziej)

### 4.0 Nagłówki
Poniżej umieszczono listing kodu, w którym są załączane pliki nagłówkowe potrzebne do realizacji projektu:
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
Poniżej przedstawiono listing, w którym ... // TODO
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
- nazwę dekodera, którego chcemy użyć do odbierania tej stacji (o dostępnych dekoderach i działadniu więcej w sekcji 4.3)
- URI stacji (link bezpośrednio do pliku z streamem audio, lub do pliku z playlistą .m3u8)

Playlisty .m3u8 (w zasadzie M3U, m3u8 to M3U kodowane w UTF-8) to niewielkie pliki tekstowe. Szczegółowe informacje na ich temat można przeczytać tutaj: [m3u8 - wiki](https://en.wikipedia.org/wiki/M3U).

Poza opcjonalnymi metadanymi po prostu zawierają adresy elementów playlisty. Element zawiera adres bezwzględny, lub względny (względem folderu, w którym znajduje się playlista) do docelowego pliku. W sczególności może istnieć playlista jednoelementowa lub elementem playlisty może być inna playlista. W przypadku playlisty konkretnego radia internetowego bardzo częstym przypadkiem jest stworzenie playlisty, której elementami są // TODO: wyjaśnić "mirror stream" mirror streamy; dzięki takiemu rozwiązaniu gdy jeden stream ulegnie awarii odbiornik radia automatycznie przełącza się na kolejny adres z playlisty.

### 4.2 Inicjalizacja interfejsu HAL
```c
audio_board_handle_t board_handle = audio_board_init();
audio_hal_ctrl_codec(board_handle->audio_hal, AUDIO_HAL_CODEC_MODE_DECODE, AUDIO_HAL_CTRL_START);
```
Inicjalizacja interfejsu HAL płytki. Jest to interfejs stanowiący warstwę abstrakcji pomiędzy wygodnymi dla użytkownika funkcjami a specyficznymi sterownikami sprzętowymi dla konkretnej płytki (w naszym przypadku `lyra_v4_3`). Interfejs pozwala nam np. konfigurować częstotliwość próbkowania dla ADC oraz DAC, szerokość bitów, parametry I2S oraz głośność. Można znaleźć specyficzne dla danej płytki konfiguracje w katalogu `esp-adf/components/audio_board/<nazwa_płytki>/`. Na przykład konfiguracja dla naszej płytki wygląda tak:  
(plik `esp-adf/components/audio_board/lyra_v4_3/board_def.h`)
```c
#define AUDIO_CODEC_DEFAULT_CONFIG(){                   \
        .adc_input  = AUDIO_HAL_ADC_INPUT_LINE1,        \
        .dac_output = AUDIO_HAL_DAC_OUTPUT_ALL,         \
        .codec_mode = AUDIO_HAL_CODEC_MODE_BOTH,        \
        .i2s_iface = {                                  \
            .mode = AUDIO_HAL_MODE_SLAVE,               \
            .fmt = AUDIO_HAL_I2S_NORMAL,                \
            .samples = AUDIO_HAL_48K_SAMPLES,           \
            .bits = AUDIO_HAL_BIT_LENGTH_16BITS,        \
        },                                              \
};
```

Po inicjalizacji interfejsu (funkcja `audio_board_init`) jeszcze należy uruchomić kodek w trybie dekodującym (jedynie będziemy dekodować dane, gdybyśmy potrzebowali równocześnie korzystać np. z mikrofonu i nagrywać audio w celu zapisania w formacie .mp3 musielibyśmy zamiast `AUDIO_HAL_CODEC_MODE_DECODE` ustawić `AUDIO_HAL_CODEC_MODE_BOTH`)

### 4.3 Konfiguracja `audio_pipelnie`
Podstawowym blokiem, z którego korzystamy podczas tworzenia aplikacji z biblioteką esp-adf jest `audio_element`. Wszystkie dekodery, enkodery, filtry, strumienie wejściowe oraz wyjściowe są właśnie `audio_element`. Zadaniem każdego elementu jest dostanie danych na wejściu, przetworzenie ich oraz przekazanie na wyjście.

Dzięki takiemu funkcyjnemu podejściu bardzo naturalne wydaje się składanie kilku elementów w celu stworzenia właśnie `audio_pipeline`, który zawiera kilka `audio_element`ów połączonych w pewien ciąg. Przykładowo:
![Audio Pipeline Example](audio_pipeline_example.png)  
Trzy środkowe elementy to właśnie `audio_element`y

Głównym zadaniem `audio_pipeline` jest przekazywanie danych z wyjścia poprzedniego elementu do wejścia kolejnego. Odbywa się to poprzez `ring_buffer`, który jest buforem pomiędzy każdymi dwoma połączonymi elementami. Dodatkową zaletą jest możliwość kontrolowania całego pipeline'u zamiast pojedynczych elementów (przydaje się np. podczas przełączania stacji).

Najpierw tworzymy pusty pipeline:
```c
audio_pipeline_cfg_t pipeline_cfg = DEFAULT_AUDIO_PIPELINE_CONFIG();
pipeline = audio_pipeline_init(&pipeline_cfg);
```
Następnie tworzymy poszczególne elementy:
- `http_stream` do strumieniowania audio z internetu przez http
  ```c
  http_stream_cfg_t http_cfg = HTTP_STREAM_CFG_DEFAULT();
  http_cfg.event_handle = _http_stream_event_handle;
  http_cfg.type = AUDIO_STREAM_READER;
  http_cfg.enable_playlist_parser = true;
  http_stream_reader = http_stream_init(&http_cfg);
  ```
  Ważną zaletą tego elementu jest jego zdolność do odczytytwania playlist w plikach .m3u (zobacz 4.1) i automatyczne przełączanie pomiędzy elementami tej playlisty.
- `i2s_stream` do strumieniowania już zdekodowanego audio do wyjścia przez i2s
  ```c
  i2s_stream_cfg_t i2s_cfg = I2S_STREAM_CFG_DEFAULT();
  i2s_cfg.type = AUDIO_STREAM_WRITER;
  i2s_stream_writer = i2s_stream_init(&i2s_cfg);
  ```
- Pomiędzy tymi dwoma elementami musimy umieścić jeszcze dekoder audio. Esp-adf udostępnia nam wiele różnych dekoderów audio, pełną listę możemy znaleźć w katalogu `esp-adf-libs/esp_codec/include/codec/`. Niestety dostępne są jedynie pliki nagłówkowe a same dekodery są udostępnione jedynie jako biblioteki statyczne w pliku `esp-adf-libs/esp_codec/lib/esp32/libesp_codec.a`.  

  W naszym przypadku korzystamy jedynie z dekoderów mp3 oraz aac. Ciekawa jest możliwość stworzenia auto-dekodera, któremu zadajemy listę dekoderów a on automatycznie rozpoznaje, któego należy użyć w momencie gdy dostaje pierwsze dane na wejście.
  - dekoder aac
    ```c
    aac_decoder_cfg_t aac_cfg = DEFAULT_AAC_DECODER_CONFIG();
    aac_decoder = aac_decoder_init(&aac_cfg);
    ```
    
  - auto-dekoder zawierający dekodery aac oraz mp3
    ```c
    audio_decoder_t auto_decode[] = {
        DEFAULT_ESP_MP3_DECODER_CONFIG(),
        DEFAULT_ESP_AAC_DECODER_CONFIG()
    };
    esp_decoder_cfg_t auto_dec_cfg = DEFAULT_ESP_DECODER_CONFIG();
    auto_decoder = esp_decoder_init(&auto_dec_cfg, auto_decode, sizeof(auto_decode) / sizeof(audio_decoder_t));
    ```
    Po przetestowaniu auto-dekodera na różnych stacjach radiowych okazało się, że radzi sobie dobrze z plikami .aac oraz .mp3 i wtedy właśnie go używamy. Natomiast w przypadku pliku .mp4a nie rozpoznaje go odpowiednio i wtedy wymuszamy stosowanie dekodera aac.
  - dekoder ogg: niestety okazało się, że posiada wadę i nie udało nam się go wykorzystać w celu dekodowania stacji radiowych w tym formacie. (znaleźliśmy wątek w którym inna osoba opisuje ten sam problem ale nie posiada rozwiązania: https://esp32.com/viewtopic.php?t=16693). W liście stacji pozostawiliśmy zakomentowaną stację, która z tego powodu nie działała, żeby o tym pamiętać.

Po stworzeniu wszystkich elementów spinamy je w jeden pipeline:
```c
audio_pipeline_register(pipeline, http_stream_reader, "http");
audio_pipeline_register(pipeline, auto_decoder,        "auto");
audio_pipeline_register(pipeline, aac_decoder,        "aac");
audio_pipeline_register(pipeline, i2s_stream_writer,  "i2s");


ESP_LOGI(TAG, "[2.6] Link it together http_stream-->auto_dec/aac_dec-->i2s_stream-->[codec_chip]");
audio_pipeline_link(pipeline, (const char *[]) {"http", stations[current_ix].decoder_name, "i2s"}, 3);
```

Łączenie elementów w pipeline polega na: 
1) zarejestrowaniu wszystkich elementów do pipeline'u wraz z przypisaniem każdemu unikalnej nazwy
2) podaniu ciągu nazw, który odzwierciedla kolejność elementów. W szczególności możemy zarejestrować więcej elementów niż używamy i właśnie to się dzieje powyżej, ponieważ rejestrujemy oba dekodery `"auto"` i `"aac"` natomiast w danym momencie używamy jedynie jednego z nich.

Na koniec jeszcze w ustalamy URI, z którego `http_stream` ma pobierać dane:
```c
audio_element_set_uri(http_stream_reader, stations[current_ix].uri);
```

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
Poza przetwarzaniem danych w pipeline potrzebujemy jakiegoś sposobu na komunikacje pomiędzy użytkownikiem a aplikacją. Do tego właśnie służą wydarzenia, które mogą być wysyłane przez zarówno pipeline (poszczególne elemty je wysyłają ale są przekazywane przez pipeline) jak i peryferia naszej płytki (np. przyciski).

Tworzymy naszego listenera, który będzie nasłuchiwał komunikatów:
```c
audio_event_iface_cfg_t evt_cfg = AUDIO_EVENT_IFACE_DEFAULT_CFG();
audio_event_iface_handle_t evt = audio_event_iface_init(&evt_cfg);
```

Ustawiamy aby nasłuchiwał wydarzeń z pipeline'u:
```c
audio_pipeline_set_listener(pipeline, evt);
```
oraz z peryferii płytki:
```c
audio_event_iface_set_listener(esp_periph_set_get_event_iface(set), evt);
```

### 4.6 Uruchomienie odtwarzania stacji radiowej
To jest główna część radia - cały punkt 4.6 znajduje się w nieskończonej pętli, z której jedynym wyjściem jest naciśnięcie przycisku "mode".

#### 4.6.0 Odbieranie `music_info`
```c
audio_event_iface_msg_t msg;
esp_err_t ret = audio_event_iface_listen(evt, &msg, portMAX_DELAY);
if (ret != ESP_OK) {
    ESP_LOGE(TAG, "[ * ] Event interface error : %d", ret);
    continue;
}

if (msg.source_type == AUDIO_ELEMENT_TYPE_ELEMENT
    && msg.source == (void *) current_decoder
    && msg.cmd == AEL_MSG_CMD_REPORT_MUSIC_INFO) {
    audio_element_info_t music_info = {0};
    audio_element_getinfo(current_decoder, &music_info);

    ESP_LOGI(TAG, "[ * ] Receive music info from %s using %s, sample_rates=%d, bits=%d, ch=%d",
                stations[current_ix].name, stations[current_ix].decoder_name, music_info.sample_rates,
                music_info.bits, music_info.channels);

    audio_element_setinfo(i2s_stream_writer, &music_info);
    i2s_stream_set_clk(i2s_stream_writer, music_info.sample_rates, music_info.bits, music_info.channels);
    continue;
}
```
Odczytujemy dane z interface'sie wszystkich wydarzeń, a następnie postępujemy zgodnie z tym, jakiego typu jest to wydarzenie.

Najpierw sprawdzamy czy nie wydarzył się jakiś błąd i czy funkcja nasłuchująca wydarzeń, zwróciła jakiś błądu. Jeśli tak się stało, po prostu go wypisujemy i pomijamy resztę while'a.

Następnie sprawdzamy, czy powinniśmy rozpocząć odtwarzanie stacji radiowej - a więc czy możemy odczytać i ustawić odpowiednie parametry (// TODO: zapróbkowana częstotliwość, ilu bitowa jest liczba i na ile kanałów jest nadawana muzyka).
Aby tak było, muszą być spełnione warunki:
- wiadomość musi dotyczyć audio
- źródłem musi być obecny dekoder
- typem komendy elementu odpowiedzialnego za audio musi być "report music info"
Gdy powyższe warunki są spełnione musimy odpowiednio ustawić wartości strumienia i2s oraz jego zegar.
Inicjujemy zatem i odczytujemy ustawienia audio, z obecnego dekodera oraz wypisujemy logi o tym zdarzeniu.
Następnie przy pomocy uzysknych danych, ustawiamy odpowiednie dane strumienia i2s, a następnie jego zegar.

Następnie pomijamy resztę pętli while.

#### 4.6.1 Fail-over
Czasami zdarzają się błedy odczytu w elemencie `http_stream`. Wówczas dostajemy odpowiedni komunikat i w celu uniknięcia dalszych błędów spowodowanych propagowaniem błędnych danych w głąb pipeline'u restartujemy go:
```c
/* restart stream when the first pipeline element (http_stream_reader in this case) receives stop event (caused by reading errors) */
        if (msg.source_type == AUDIO_ELEMENT_TYPE_ELEMENT && msg.source == (void *) http_stream_reader
            && msg.cmd == AEL_MSG_CMD_REPORT_STATUS && (int) msg.data == AEL_STATUS_ERROR_OPEN) {
            ESP_LOGW(TAG, "[ * ] Restart stream");
            audio_pipeline_stop(pipeline);
            audio_pipeline_wait_for_stop(pipeline);
            audio_pipeline_reset_ringbuffer(pipeline);
            audio_pipeline_reset_items_state(pipeline);
            audio_pipeline_run(pipeline);
            continue;
        }
```
W tym miejscu widzimy wygodę interfejsu `audio_pipeline`, który pozwala nam np. resetować stan wszystkich elementów.


#### 4.6.2 Obsługa poszczególnych przycisków
```c
if ((msg.source_type == PERIPH_ID_TOUCH || msg.source_type == PERIPH_ID_BUTTON)
    && (msg.cmd == PERIPH_TOUCH_TAP || msg.cmd == PERIPH_BUTTON_PRESSED)) {
    
    if ((int) msg.data == get_input_play_id()) {
        ESP_LOGI(TAG, "[ * ] [Play] touch tap event");
        audio_element_state_t el_state = audio_element_get_state(i2s_stream_writer);
        switch (el_state) {
            case AEL_STATE_INIT :
                ESP_LOGI(TAG, "[ * ] Starting audio pipeline");
                audio_pipeline_run(pipeline);
                break;
            case AEL_STATE_RUNNING :
                ESP_LOGI(TAG, "[ * ] Pausing audio pipeline");
                audio_pipeline_stop(pipeline);
                audio_pipeline_wait_for_stop(pipeline);
                audio_pipeline_reset_ringbuffer(pipeline);
                audio_pipeline_reset_items_state(pipeline);
                break;
            case AEL_STATE_STOPPED :
                ESP_LOGI(TAG, "[ * ] Resuming audio pipeline");
                audio_pipeline_resume(pipeline);
                break;
            default :
                ESP_LOGI(TAG, "[ * ] Not supported state %d", el_state);
        }
    } else if ((int) msg.data == get_input_set_id()) {
        ESP_LOGI(TAG, "[ * ] [Set] touch tap event");
        current_ix = (current_ix+1) % RADIOS_NUMBER;
        ESP_LOGI(TAG, "[ * ] changin' to radio %d %s", current_ix, stations[current_ix].name);
        ESP_LOGI(TAG, "[ * ] station's decoder %s and URI: %s", stations[current_ix].decoder_name, stations[current_ix].uri);
        audio_pipeline_stop(pipeline);
        audio_pipeline_wait_for_stop(pipeline);
        audio_pipeline_reset_ringbuffer(pipeline);
        audio_pipeline_reset_items_state(pipeline);
        audio_element_set_uri(http_stream_reader, stations[current_ix].uri);
        next_decoder = audio_pipeline_get_el_by_tag(pipeline, stations[current_ix].decoder_name);
        if(current_decoder != next_decoder){
            current_decoder = next_decoder;
            ESP_LOGI(TAG, "[ * ] changin' decoder to %s", stations[current_ix].decoder_name);
            audio_pipeline_breakup_elements(pipeline, current_decoder);
            audio_pipeline_relink(pipeline, (const char* []) {"http", stations[current_ix].decoder_name, "i2s"}, 3);
            audio_pipeline_set_listener(pipeline, evt);
        }
        audio_pipeline_run(pipeline);
    } else if ((int) msg.data == get_input_volup_id()) {
        ESP_LOGI(TAG, "[ * ] [Vol+] touch tap event");
        player_volume += 10;
        if (player_volume > 100) {
            player_volume = 100;
        }
        audio_hal_set_volume(board_handle->audio_hal, player_volume);
        ESP_LOGI(TAG, "[ * ] Volume set to %d %%", player_volume);
    } else if ((int) msg.data == get_input_voldown_id()) {
        ESP_LOGI(TAG, "[ * ] [Vol-] touch tap event");
        player_volume -= 10;
        if (player_volume < 0) {
            player_volume = 0;
        }
        audio_hal_set_volume(board_handle->audio_hal, player_volume);
        ESP_LOGI(TAG, "[ * ] Volume set to %d %%", player_volume);
    } else if ((int) msg.data == get_input_mode_id()) {
        ESP_LOGI(TAG, "[ * ] [Mode] button pressed");
        break;
    }
}
```
W tym fragmencie zajmujemy się obsługą przycisków.

Aby zakwalifikować wydarzenie jako przycisk i abyśmy go zaczęli przetwarzać muszą być spełnione następujące oba warunki:
- Źródłem wiadomości musi być jedno z:
  - przyciski dotykowe
  - zwykły przycisk fizyczne
- Komenda, czyli typ akcji jedna z:
  - dotknięcie dotykowego przycisku
  - wciśnięcie zwykłego przycisku

Następnie gdy mamy pewność, że przycisk został wciśnięty, należy go zidentyfikować i wykonać przypisane mu zadanie.
Przy pomocy pola dane sprawdzamy, który przycisk był wciśnięty - obsługujemy przyciski:
- Play
- Set
- Vol-
- Vol+
- Mode (pozostałe są dotykowe, ten jest "normalny")

W zależności od wciśniętego przycisku wykonujemy następującą akcję(od najprostszych):
- Wyjście z nieskończonego while'a, by zakończyć program i wyłączyć radio,
- Zmienienie głośności radia,
- Pausowanie i startowanie/odpauzowywanie radia,
  - W zależności od obecnego stanu, przechodzimy w następny,
  - Pierwsze wystartowanie audio pipeline'a robimy run'em, natomiast aby później odpauzować wykorzystujemy tylko resume - główną różnicą jest to, że run dodatkowo tworzy taski dla wszystkich elementów pipeline'a.
  - Aby zapauzować radio najpierw zatrzymujemy pipeline (i czekamy aż się zatrzyma), ze względu na to, że działamy na streamach, musimy również wyczyścić ringbuffer i zmienić stan itemów, aby później mogły być na nowo ustawione.
- Zmiana stacji.
  - Zasadniczo restartujemy stream, odpalając go na nowej stacji - stopujemy pipeline, czyścimy ringbuffer i items state, ustawiamy URI nowej stacji i uruchamiamy stream na nowej stacji.
  - Dodatkowo sprawdzamy, czy obecny dekoder jest tym samym, na którym będziemy odbierać nową stację - gdy tak nie jest, musimy go zmienić.
    - Aby zmienić dekoder najpierw zrywamy z pipieline'a obecny dekoder, a następnie linkujemy w pipeline'ie http, obecny dekoder oraz i2s (wszystkie 3 są audio element handlami, a odwołujemy się do nich za pomocą nadanych im nazw podczas rejestracji), na końcu dodajemy event listnera do pipeline'u.

Dodawanie obsługi przycisków jest bardzo proste, natomiast mieliśmy bardzo dużo problemów, z dekoderami które niestety miały swoje problemy oraz zatrzymywaniem i wznawianiem strumienia, w różnych wariantach, a więc i również zmiana stacji. Dlatego też, szczególnie w tych miejscach, możemy mieć nie najbardziej optymalne/poprawne rozwiązanie - za to na pewno jest działające.

Największe problemy z jakimi się tu zetknęliśmy:
- Nie działa dekoder OGG, pod względem wznawiania jego działania.
- Dekoder auto, nie radzi sobie z plikami mp4a, mimo że spokojnie powinien - działa chociażby dekoder acc.
- Musimy traktować każdą pauzę i wznowienie, jako koniec i początek nadawania oraz resetować ringbuffer i stan elementów, ponieważ nie ma sensu pamiętania wszystkich tych rzeczy w przypadku streamu audio - dodatkowo jest kilka dosyć podobnych i zarazem umiarkowanie dobrze opisanych funkcji.

### 4.7 Zatrzymanie wszystkiego i deinicjalizacja 
```c
ESP_LOGI(TAG, "[ 6 ] Stop audio_pipeline");
audio_pipeline_stop(pipeline);
audio_pipeline_wait_for_stop(pipeline);
audio_pipeline_terminate(pipeline);

audio_pipeline_unregister(pipeline, http_stream_reader);
audio_pipeline_unregister(pipeline, i2s_stream_writer);
audio_pipeline_unregister(pipeline, auto_decoder);
audio_pipeline_unregister(pipeline, aac_decoder);

/* Terminate the pipeline before removing the listener */
audio_pipeline_remove_listener(pipeline);

/* Stop all peripherals before removing the listener */
esp_periph_set_stop_all(set);
audio_event_iface_remove_listener(esp_periph_set_get_event_iface(set), evt);

/* Make sure audio_pipeline_remove_listener & audio_event_iface_remove_listener are called before destroying event_iface */
audio_event_iface_destroy(evt);

/* Release all resources */
audio_pipeline_deinit(pipeline);
audio_element_deinit(http_stream_reader);
audio_element_deinit(i2s_stream_writer);
audio_element_deinit(auto_decoder);
audio_element_deinit(aac_decoder);
esp_periph_set_destroy(set);
```
Na sam koniec pozostało nam poodpinanie odpowiednich elementów i deinicjalizację wszystkich wykorzystanych wcześniej komponentów.

Zatrzymujemy zatem cały pipeline i odpinamy od niego wszystkie elementy.
Zaczynamy od wyrejestrowania http_stream_reader'a, i2s_stream_reader'a oraz obu dekoderów - auto i acc.
Następnie odpinamy słuchaczy pipeline'a.
Pozostało już tylko pousuwanie zasobów, ale żeby to zrobić musimy najpierw zatrzymać wszystkie peryferia - domyślne dla ESP oraz ze względu na nasz projekt, moduł wi-fi oraz przyciski.
Odpinamy słuchaczy peryferiów i dopiero wtedy możemy zniszczyć interface wydarzeń.
Następnym krokiem jest już uwolnienie wszystkich zasobów, a więc deinicjuejmy oraz niszczymy:
- piepline
- http_stream_reader
- i2s_stream_writer
- oba dekodery
  - auto dekoder
  - aac dekoder
- zbiór peryferiów

W tym momencie kończy się cała zaimplementowana aplikacja i aby ją uruchomić ponownie, musimy zresetować płytkę, przy pomocy przycisku RST (reset).