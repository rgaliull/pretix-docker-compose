# Pretix Docker-Compose setup
The repository includes a [Pretix](https://pretix.eu/about/de/) docker-compose configuration for development in Coolify.

## Usage

You need to setup Persistent storage for pretix.cfg, nginx.conf, crontab as FILE

make desired configuration in pretix.cfg
Expamle:
instance_name = WowCircus
url=https://tickets.wowcircus.de
currency=EUR

[locale]
default=en
timezone=Europe/Berlin

and so on in [mail] section

## License
This product is available under the Apache 2.0 license.
