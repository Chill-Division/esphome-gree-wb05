# sinclair_ac

This component is from [piotrva/esphome_gree_ac](https://github.com/piotrva/esphome_gree_ac)
(GPL-3.0), commit `48cc9d49e9ce2faf00a3d7b6ebba8075fb84c4ca`, with one change for Realtek
targets:

* `esppac.h`: undefine `ON`, `OFF`, `IN` and `OUT` after the includes. Realtek's AmebaD SDK
  (LibreTiny `rtl87xx`) defines them as macros, which breaks the option-name constants.

The original protocol work and component are piotrva's; see the upstream README for the
protocol notes and supported features.
