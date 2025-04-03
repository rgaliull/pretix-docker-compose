FROM pretix/standalone:stable

USER root

ENV IMAGE_CRON_DIR="/image/cron" \
    IMAGE_CONFIG_DIR="/image/config"

ADD files /image
COPY crontab /tmp/crontab

ARG SSH_KEY
RUN mkdir -p /root/.ssh && \
    echo "${SSH_KEY}" > /root/.ssh/id_rsa && \
    chmod 600 /root/.ssh/id_rsa && \
    mv /image/supervisord/crond.conf /etc/supervisord/crond.conf && \
    pip install crontab && chmod 644 $IMAGE_CONFIG_DIR/ssl/*.crt && chmod +x $IMAGE_CRON_DIR/cron.py && \
    chmod -R 0600 /root/.ssh && \
    mkdir -p /etc/ssh && \
    ssh-keyscan -t rsa -p 10022 code.rami.io >> /root/.ssh/known_hosts && \
    echo StrictHostKeyChecking=no >> /root/.ssh/config && \
    DJANGO_SETTINGS_MODULE= pip3 install -U "git+ssh://git@code.rami.io:10022/pretix/pretix-seating.git@stable#egg=pretix-seating" && \
    cd /pretix/src && \
    sudo -u pretixuser make production

USER pretixuser

EXPOSE 443

ENTRYPOINT ["pretix"]
CMD ["all"]

