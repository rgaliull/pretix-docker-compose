FROM pretix/standalone:latest

USER root

ENV IMAGE_CRON_DIR="/image/cron" \
    IMAGE_CONFIG_DIR="/image/config"

ADD files /image
COPY crontab /tmp/crontab

RUN mv /image/supervisord/crond.conf /etc/supervisord/crond.conf && \
    pip install crontab && chmod 644 $IMAGE_CONFIG_DIR/ssl/*.crt && chmod +x $IMAGE_CRON_DIR/cron.py && \
    pip install pretix-manualseats && \
    python -m pretix migrate && \
    python -m pretix rebuild && \

USER pretixuser

EXPOSE 443

ENTRYPOINT ["pretix"]
CMD ["all"]