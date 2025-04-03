FROM pretix/standalone:stable

USER root

ENV IMAGE_CRON_DIR="/image/cron" \
    IMAGE_CONFIG_DIR="/image/config"

ADD files /image
COPY crontab /tmp/crontab

# Добавляем аргумент сборки
ARG SSH_KEY

# Настраиваем SSH
RUN mkdir -p /root/.ssh 
# Запишем ключ в файл
RUN echo '${SSH_KEY}' > /root/.ssh/id_rsa
RUN cat /root/.ssh/id_rsa
RUN chmod 600 /root/.ssh/id_rsa
RUN chmod -R 0600 /root/.ssh
RUN mkdir -p /etc/ssh
RUN ssh-keyscan -t rsa -p 10022 code.rami.io >> /root/.ssh/known_hosts
RUN echo "StrictHostKeyChecking no" > /root/.ssh/config

# Установим остальные компоненты
RUN mv /image/supervisord/crond.conf /etc/supervisord/crond.conf && \
    pip install crontab && \
    chmod 644 $IMAGE_CONFIG_DIR/ssl/*.crt && \
    chmod +x $IMAGE_CRON_DIR/cron.py

# Установим приватный пакет
RUN DJANGO_SETTINGS_MODULE= pip3 install -U "git+ssh://git@code.rami.io:10022/pretix/pretix-seating.git@stable#egg=pretix-seating"

# Выполним финальные команды
RUN cd /pretix/src && sudo -u pretixuser make production

# Удалим приватный ключ после использования
RUN rm -f /root/.ssh/id_rsa

USER pretixuser

EXPOSE 443

ENTRYPOINT ["pretix"]
CMD ["all"]

