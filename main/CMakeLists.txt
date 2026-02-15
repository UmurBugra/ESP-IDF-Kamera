idf_component_register(SRCS main.c wave_1ch_16bits.c
                    INCLUDE_DIRS "."
                    PRIV_REQUIRES esp_driver_gpio esp_driver_jpeg xfer_http)
