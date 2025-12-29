idf_component_register(SRCS "attack_dos.c" "attack_method.c" "main.c" "attack_handshake.c" "attack_pmkid.c" "attack.c"
                    INCLUDE_DIRS .
                    REQUIRES esp_wifi esp_event esp_timer wifi_controller webserver)
