FROM pretix/standalone:2026.7.0

USER root

ENV IMAGE_CRON_DIR="/image/cron" \
    IMAGE_CONFIG_DIR="/image/config"

ADD files /image
COPY crontab /tmp/crontab
# Slovenian is an incubating Pretix translation and is not included in the
# standalone image by default. The public PO files are compiled by Pretix's
# production build below.
COPY locale/ /pretix/src/pretix/locale/
COPY patches/add-slovenian-language.patch /tmp/add-slovenian-language.patch

RUN mkdir -p /root/.ssh /etc/ssh \
    && ssh-keyscan -t rsa -p 10022 code.rami.io >> /root/.ssh/known_hosts \
    && printf '%s\n' 'Host code.rami.io' '  Port 10022' '  StrictHostKeyChecking yes' > /root/.ssh/config


# Установим остальные компоненты
RUN mv /image/supervisord/crond.conf /etc/supervisord/crond.conf && \
    pip install crontab && \
    find "$IMAGE_CONFIG_DIR/ssl" -type f -name '*.crt' -exec chmod 644 {} + && \
    chmod +x $IMAGE_CRON_DIR/cron.py

# The private plugin key is mounted only for this build step and is never copied
# into an image layer or stored in the repository.
RUN --mount=type=ssh,id=pretix_seating \
    DJANGO_SETTINGS_MODULE= pip3 install -U "git+ssh://git@code.rami.io:10022/pretix/pretix-seating.git@stable#egg=pretix-seating"

# Выполним финальные команды
RUN chown -R pretixuser:pretixuser /pretix/src/pretix/locale/sl && \
    patch -d /pretix/src -p1 < /tmp/add-slovenian-language.patch && \
    cd /pretix/src && sudo -u pretixuser make production

USER pretixuser

EXPOSE 443

ENTRYPOINT ["pretix"]
CMD ["all"]
